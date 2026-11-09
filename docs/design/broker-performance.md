# Cresco — Broker & Data-Plane Performance (v1.3)

**Scope:** every performance change made to the Cresco message broker and data paths on the
`felix-hygiene` line, with the measured effect and the config lever that controls it. All levers are
`-D`-overridable (the `Config` System-property fallthrough), defaults preserve shipped behavior.

> Companion: **`docs/distributed-identity-trust-design.md`** (security / tenant isolation). Security
> enforcement (`broker_security_enabled`) is orthogonal to every performance lever here and, when off
> (default), has zero effect on these paths.

!!! note "wsapi data-plane server has since migrated to Netty"
    The data-plane throughput work in §6 was measured on the then-current **Jetty 12** wsapi server. The
    wsapi server has since been rewritten on **Netty** (`NettyWsServer`), carrying forward the same
    techniques — permessage-deflate off, large adaptive I/O buffers, big socket buffers, and a write
    high-water backpressure mark (`wsapi_read_chunk_bytes` / `wsapi_socket_buffer_bytes` /
    `wsapi_write_high_water_bytes`). The Jetty-specific numbers and bug references below are the historical
    baseline; see the [wsapi plugin](../plugins/wsapi.md) for the current server.

!!! note "Since this was written: topics run with producer flow control (dataplane backpressure)"
    Queues (the control plane) still run with producer flow control off and priority governing dispatch, as below.
    Topics (the data plane) now have their own policy with producer flow control **on**
    (`activemq_topic_producer_flow_control`, `activemq_topic_memory_limit`) and no discarding pending-message limit
    (`dataplane_backpressure`, default on): a slow subscriber slows the topic's writers instead of losing messages, data
    bridge connectors send synchronously, and `broker_bridge_prefetch_data` is 64. Security is on by default since the
    1.3 hardening. See [Data Plane](../architecture/dataplane.md#backpressure-nothing-lost-order-kept) and the
    [configuration reference](../reference/configuration.md).

---

## 0. Summary of measured wins

| Change | Path | Measured effect |
|--------|------|-----------------|
| ActiveMQ 6.1.4 → **6.2.7** | broker core | no regression; CVE-2023-46604-class hardening |
| Transport `socketBufferSize` 64KB → **2MB** | inter-node TCP bridge | **2.1× (64KB) – 6.1× (256KB)** throughput; **eliminates 40–48% data-plane loss** |
| KahaDB `enableJournalDiskSyncs` on → **off** | persistent bulk | **~60× at 16KB (2.6→159 MB/s), ~21× at 256KB (40→911)** |
| Control-plane **priority QoS** | liveness ping under load | ping send p99 **6041ms → 15ms**, zero drops (was agent-drop under load) |
| Data-plane **sharding** + per-shard dedicated connections | agent→region | 800 flat → **1,150** scaling, 0 loss |
| Parallel **bridge connectors** (per-shard) | region↔global | removes single-TLS-socket single-core cliff |
| Native TLS (**tcnative/BoringSSL** wss, **Conscrypt** broker) | TLS crypto | wss single-stream **2.1×** (289→604), aggregate **4,380**; broker bridge **+15–40%** cross-node |
| **Jetty 12.1.10** data-plane (permessage-deflate off + input buffers) | wsapi dataplane | Python 256KB **217 → 444–474 MB/s**; Java now matches Python (**366 vs 362**), byte-exact, 0 loss |
| stunnel single-buffer egress + buffers | stunnel tunnel | **~9× (250 → 2,200 steady, 3,110 at 1GB)** |

---

## 1. ActiveMQ 6.2.7 upgrade
- **What:** embedded ActiveMQ Classic **6.1.4 → 6.2.7** (`controller/pom.xml` `activemq.version`). Only the
  4 activemq jars change in the transitive tree (broker/client/kahadb/openwire-legacy); no other churn.
- **Why:** current release; hardens the CVE-2023-46604 OpenWire deserialization vector. Needs Java 17+ (JDK 21 ok).
- **Cresco-specific notes:** 6.2.x removed `java.lang` from default trusted serializable packages — N/A to
  the core path (MsgEvent is **Gson JSON `TextMessage`**, not `ObjectMessage`). If a plugin ever sends+receives
  data-plane `ObjectMessage`, set `activeMQSslConnectionFactory.setTrustedPackages(...)`. **Do not** move the
  core MsgEvent path to `ObjectMessage` (RCE surface, version-brittle, breaks the Python client, not faster).

## 2. Broker throughput levers (all `-D`-configurable)
Added to `ActiveBroker.java`; defaults preserve behavior.
- `activemq_socket_buffer_size` (**2MB**, was ~64KB default): broker-side accept-socket buffers. The client
  already set large buffers; the broker acceptor did not → the single biggest inter-node throughput lever.
  **2.1×–6.1×** and kills 40–48% data-plane loss across the TCP bridge.
- `activemq_max_frame_size` (**128MB**): wire-format max frame so large binary blocks aren't rejected/split.
- `activemq_journal_disk_syncs` (**false**): KahaDB fsync-per-write off. Cresco uses persistence as a
  **flow-control** mechanism, not for crash durability — so removing the per-write fsync latency (the
  persistent-bulk bottleneck) is free for the stated intent. **~60×/~21×** on persistent bulk.
- `activemq_use_cache` (**false**, by design): the dispatch cache is a store-dispatch optimization that A/B
  showed buys **nothing** on Cresco's in-JVM `vm://` / non-persistent paths (≈360 MB/s either way). Kept off.
- `activemq_dedicated_task_runner` (**false**): pooled task runner instead of thread-per-destination →
  scales to large agent counts without thread blowup.
- System usage ceilings: `broker_memory_limit` (default = max(512MB, ½ heap)), `broker_store_limit` (8GB),
  `broker_temp_limit` (4GB). The old fixed 256MB non-persistent ceiling collapsed aggregate throughput
  (~16 MB/s at 4×256KB streams vs ~470 with headroom) via the pending-message eviction throttle.
- Also configurable: `activemq_producer_flow_control` (false), `activemq_prioritized_messages` (true),
  `activemq_destination_memory_limit`, `activemq_gc_inactive_destinations`, `activemq_inactive_timeout_before_gc`,
  `activemq_persistent`, `activemq_destination_purge_period`.

**Backpressure vs speed — the critical distinction:** buffers and fsync are **speed** knobs, not backpressure.
Backpressure comes from `producerFlowControl` / prefetch / system-usage. Fast-producer/slow-consumer test:
flow-control ON throttled the producer ~4× and blocked; OFF ran ahead with a memory-bounded (~36%) backlog, no
crash. The old small-buffer+fsync combo was only *accidental* backpressure. Cresco runs flow-control **off** and
lets **priority QoS** govern dispatch (§3), with non-persistent eviction bounding telemetry memory.

## 3. Control-plane priority QoS (agent-liveness never starved)
- **Problem:** under bulk+telemetry load the non-persistent liveness ping (shared flow-controlled producer)
  was evicted/blocked → a single 5s miss → `globalControllerLost` → agent/region dropped.
- **Fix (controller `f157677`):** `MsgQoS` classifier → tiers **LIVENESS (pri 9, persistent) > CONTROL (pri 7,
  persistent) > TELEMETRY (pri 4, non-persistent) > BULK (pri 1)**; liveness+control on a **separate producer/
  session** from telemetry so floods can't serialize/evict/block the ping; flow-control off (priority governs
  dispatch); N-consecutive-miss + retry + jitter before declaring loss.
- **Proof (StarvationRepro, real broker, flood):** ping send-call **p50 503ms / p99 6041ms → p50 7.6ms / max 15ms**,
  8 → 58 pings delivered in 15s, **zero starvation**.
- Levers: `region_ping_retries`, `region_ping_max_failures`, `controlplane_ttl`, `agentproducer_data_threads`,
  `agentproducer_data_queue`.

## 4. Data-plane sharding + parallel bridges
- **Sharding** (`DataPlaneServiceImpl`): split `global.event` into N shard topics (`dataplane_shards`,
  `dataplane_shard_topic`); a stream hashes to a shard; each shard gets a **dedicated broker connection**
  (`ActiveClient.createDedicatedSession`) instead of multiplexing everything over one socket (the agent→region
  throughput funnel). Result: 800 flat → **1,150** scaling, 0 loss.
- **Parallel bridge connectors** (`ActiveBroker.AddNetworkConnector`): the region↔global bridge is
  destination-partitioned so connector *i* forwards exactly shard *i* over its **own TLS socket/core** —
  removing the single-duplex-connector single-core cliff. Runtime-adjustable
  (`addBridgeConnections`/`removeBridgeConnections`, `broker_bridge_connections`).

## 5. Native TLS acceleration
- **wss paths (wsapi server + Java clientlib):** `netty-tcnative-boringssl-static` (Netty `SslProvider.OPENSSL`).
  Applying it to **both** ends (the Java client's JSSE crypto was the bottleneck; Python already used OpenSSL)
  gave **2.1× single-stream (289 → 604 MB/s)** and **4,380 MB/s aggregate**.
- **Broker bridge (JSSE `nio+ssl`):** **Conscrypt** JCE provider (BoringSSL) installed for the ActiveMQ TLS
  bridge → **+15–40% cross-node**. (`conscrypt-openjdk-uber` 2.6-alpha5, first with osx/linux aarch_64 natives.)

## 6. Jetty 12 data-plane (correctness + throughput) — *historical; server now Netty*
- **Correctness:** the jakarta **partial** upload handler under Jetty 12 delivered ~8KB chunks as separate
  BytesMessages (256KB → ~8 broker messages). Fixed to a **whole-message** handler → 1000/1000 byte-exact.
- **Throughput killer — permessage-deflate:** Python `websockets` requests it by default; Jetty 12 negotiated it
  → every binary block deflate-compressed on both ends (pointless, CPU-bound), capping Python 256KB at ~217 MB/s.
  **Disabled in 3 places** (server extension unregister, python `compression=None`, Java client unregister) →
  Python **217 → 444–474 MB/s**, beating the old 434.
- **Jetty 12.0.17 → 12.1.10:** the `setInputBufferSize` corruption was a 12.0.x bug fixed in 12.1; on 12.1.10 the
  256KB input buffer sticks byte-exact → Java receiver keeps up with a fast sender, **no best-effort loss**, and
  sustained 256KB is **Java 366 ≈ Python 362 MB/s**, both 20000/20000 byte-exact. (Server AND client must be
  the same Jetty version.) Other tuning: I/O buffers 4KB→256KB, maxFrameSize→16MB, TCP socket buffers 4MB,
  direct TLS buffers, blocking `getBasicRemote()` send for backpressure.

## 7. stunnel
- **Root cause:** 8KB re-chunking on egress. **Fix:** single-buffer write (`Unpooled.wrappedBuffer`) +
  configurable read allocator + socket buffers + live `nettuning`. Result: **~9× (250 → 2,200 steady,
  3,110 at 1GB)**. (`getTunnelStatus` also now reports real ACTIVE/RECOVERING/DOWN — see B13.)

---

## 8. Why the single-node client suite looked "flat"
The client-facing speed suite (`run/tests/results-history/`) is **flat** across v1.2/phase0/2/3 because those
were OSGi-correctness commits, and because the single-node paths are **in-JVM `vm://` and/or non-persistent** —
they never touch the TCP bridge socket buffers or the KahaDB journal, which is exactly where the big wins live.
Two same-host A/Bs prove the flatness is environmental, not regression: `useCache` true≈false, and ActiveMQ
6.1.4≈6.2.7 (holding tuning constant). The tuning's real target — **persistent multi-node bulk over the bridge**
— is measured by `run/tests/dataplane_internode.py` and `run/tests/java/BrokerBulkPerf.java`, which show the
2.1–6.1× (socket buffers) and ~60× (disk-syncs) wins directly.

## 9. Reproduce
- Internode bridge: `run/tests/dataplane_internode.py` (agent-001 with own wsapi; sender→global, receiver→agent → demand-forwarded over the bridge).
- Broker-direct persistent bulk: `run/tests/java/BrokerBulkPerf.java` against a plaintext connector
  (`-Dactivemq_transport=nio -Denable_dynamic_broker_port=false`, `tcp://localhost:32010`).
- Broker-bench micro: `broker-bench/` (`TuningBench` proved persistent 256KB 22 → 1167 MB/s, 52×).
- Client suite: `run/tests/run_speedtests.sh <label>` → `results-history/<label>.md`.
