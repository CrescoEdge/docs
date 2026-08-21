# Configuration Parameters

This is the exhaustive reference for Cresco node configuration parameters. Every parameter
listed here corresponds to a typed getter call (`getStringParam`, `getIntegerParam`,
`getLongParam`, `getBooleanParam`, `getDoubleParam`) in the source tree, grouped by concern.

## How configuration works

A Cresco node is configured through a `Config` helper that exposes **typed getters**. Each
getter takes a parameter name and (usually) a default value, e.g.
`getIntegerParam("broker_port", 32010)`. The resolution order is:

1. **`-D<name>=<value>` JVM system properties** passed on the launch command line.
2. **Config-file / plugin-config values** loaded into the node's config map (the same map
   that `getConfigMap()` persists to the controller DB).
3. **The default baked into the getter call** — used when neither of the above supplies a value.

Because every getter ships with a default, **an unconfigured node runs with sane, shipped
behavior**; you only set the parameters you want to change. Most feature-gates are
**default-off** (e.g. `broker_security_enabled=false`, `net_autotune=false`,
`tenant_namespacing=false`, `enable_udp_discovery=false`), so enabling a capability is an
explicit, opt-in decision. A handful of getters have **no default** (see the *Unset =
disabled* note on each such row); when unset they return `null` and the corresponding feature
is skipped.

!!! example "Setting parameters on the command line"
    ```bash
    java -Dis_region=true \
         -Dregionname=east \
         -Dbroker_security_enabled=true \
         -Dtenant_namespacing=true \
         -jar cresco-agent-*.jar
    ```

!!! warning "Types are enforced at read time"
    A few parameters are read with `getStringParam` even though they hold a numeric or
    boolean value (e.g. `failover_reconnect_delay`, `max_reconnect_attempts`,
    `use_exponential_backOff`, `wsapi_port`). Supply them as plain strings; they are parsed
    downstream. These are flagged in the tables below.

Unless noted, byte-size defaults shown as expressions (e.g. `2 * 1024 * 1024`) are given in
their evaluated form (`2 MB`). Where a default is **computed at runtime**, the table says so
explicitly.

---

## Security & tenancy (read this first)

These flags govern who can connect to the broker, whose identity is asserted, and how tenants
are isolated. **All of them are default-off**, so a stock node behaves exactly as it always
has — enabling them is a deliberate hardening/tenancy decision. Enable them consistently
across a region; a mixed fleet (some nodes enforcing, some not) will not behave as intended.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `broker_security_enabled` | boolean | `false` | Master switch for broker authentication/authorization. When on, external (non-`vm://`) clients must assert an identity (JMS username `tenant\|region\|agent`, or a cert subject under mutual TLS) and are checked against the tenant policy; when off the ACL plugin is never installed and the broker behaves as before. |
| `broker_require_client_auth` | boolean | `false` | Require mutual TLS (client certificate) on the broker transport connector. When on, clients without a trusted certificate are rejected at the TLS layer. |
| `broker_security_secret` | String | `cresco` | Shared password the local controller and clients present as the JMS password when `broker_security_enabled` is on. Change this from the default in any secured deployment. |
| `broker_superuser_tenants` | String | `cresco-system` | Comma-separated list of tenant IDs whose **network** clients are granted the cross-tenant SUPERUSER ("god view") role — e.g. a fabric-admin dashboard that must see every tenant. The local `vm://` controller and broker bridges are already exempt. |
| `broker_shared_destinations` | String | `agent.event,region.event,global.event` | Comma-separated destination prefixes treated as fabric-shared control destinations that all tenants may use (the control-plane event topics). |
| `broker_security_log_allow` | boolean | `false` | Log *allowed* (not just denied) authorization decisions in the broker ACL plugin — verbose; for debugging tenant policy only. |
| `security_regional_ca` | boolean | `false` | Enable Regional-CA identity mode ("Option C"): during discovery the region acts as issuing authority and signs each joining node a region-signed leaf certificate, and joiners present that identity. When off, the legacy self-signed discovery exchange is used. |
| `tenant_namespacing` | boolean | `false` | Namespace broker destinations, dataplane topics, and producer routing per tenant so that tenants are isolated on shared topics. Off = the historical single flat namespace. |
| `tenant_id` | String | `default` | The tenant this node belongs to; used to build the asserted broker identity, tenant-namespaced destinations, and the node's certificate subject. |
| `discovery_secret_agent` | String | random UUID | Shared secret validating **agent**-level discovery messages. Computed default is a random UUID per boot, so peers must be given a matching explicit value to discover each other. |
| `discovery_secret_region` | String | random UUID | Shared secret validating **region**-level discovery messages (region joining/forming). Random UUID by default — set explicitly across the region. |
| `discovery_secret_global` | String | random UUID | Shared secret validating **global**-level discovery messages (region-to-global join). Random UUID by default — set explicitly across the mesh. |

!!! danger "The discovery secrets default to random UUIDs"
    Because `discovery_secret_agent/region/global` default to a fresh UUID on every boot, two
    nodes will **not** discover one another unless you set the same secret on both. In a real
    deployment these are effectively required, not optional.

---

## Identity & roles

Determines what a node *is* in the hierarchy (agent / region / global / standalone) and how it
names itself. Exactly one role flag is normally set; the state machine reads them to decide the
boot path.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `is_agent` | boolean | `false` | Boot as an **agent** that reports to a regional controller. |
| `is_region` | boolean | `false` | Boot as a **regional controller** that hosts agents (and may connect to a global). |
| `is_global` | boolean | `false` | Boot as a **global controller** that manages regions. |
| `is_standalone` | boolean | `false` | Boot as a self-contained node with no parent controller. |
| `agentname` | String | *unset* | Explicit name for this agent/controller instance. **Unset = auto-generated.** |
| `regionname` | String | *unset* | Explicit name for this node's region. **Unset = auto-generated.** |
| `ipv6` / `isIPv6` | boolean | `false` | Prefer IPv6 for this node's networking/discovery (`ipv6` is read in the library `PluginBuilder`; `isIPv6` in the controller engine). Off = IPv4. |
| `pluginID` | String | *unset* | Identifier of the running plugin (set by the framework per loaded plugin, not a node-wide knob). |
| `pluginname` | String | *unset* | Human name of the running plugin (framework-supplied per plugin). |

---

## Static plugins (feature enable flags)

Which built-in plugins the controller's `StaticPluginLoader` starts. These gate whole
subsystems on or off.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `enable_sysinfo` | boolean | `true` | Load the **sysinfo** plugin (host resource/metrics collection). |
| `enable_stunnel` | boolean | `true` | Load the **stunnel** plugin (secure TCP tunneling over the fabric). |
| `enable_wsapi` | boolean | `true`¹ | Load the **wsapi** plugin (WebSocket API gateway). ¹Default is `true` in the static loader; a separate call in the wsapi plugin uses `false` as its own guard. |
| `enable_repo` | boolean | `false`¹ | Load the **repo** plugin (plugin/jar repository). ¹Read in two places with differing defaults (`false` in one, `true` in another); set explicitly if you rely on it. |
| `enable_controllermon` | boolean | `true` | Enable the controller performance monitor (`PerfControllerMonitor`). |
| `enable_console` | boolean | `false` | Attach the interactive Felix/host console on startup (host application). |
| `enable_perf` | boolean | `false` | Enable the sysinfo continuous performance sampler. |
| `plugin_config_file` | String | `conf/plugins.ini` | Path to the static plugin definition file the loader reads. |

---

## Broker (ActiveMQ) & transport

Core embedded ActiveMQ broker settings — listener, transport, persistence, and resource limits.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `broker_port` | int | `32010` | TCP port the embedded broker listens on for client connections. |
| `enable_dynamic_broker_port` | boolean | `true` | If the configured `broker_port` is busy, probe upward for a free port instead of failing. |
| `enable_broker_transport` | boolean | `true` | Enable the broker's network transport connector. When off the broker is `vm://`-only (no external clients). |
| `activemq_transport` | String | `nio+ssl` | Transport/wire protocol for broker connections (e.g. `nio+ssl` for TLS-over-NIO). |
| `activemq_persistent` | boolean | `true` | Persist messages to the KahaDB store (survive restarts). Off = in-memory only. |
| `activemq_max_frame_size` | long | `128 MB` | Maximum size of a single JMS message/frame the broker accepts. |
| `activemq_socket_buffer_size` | int | `2 MB` | TCP send/receive socket buffer size for broker transport connections. |
| `activemq_prioritized_messages` | boolean | `true` | Enable priority ordering on queues/topics (required for the tiered QoS scheme). |
| `activemq_producer_flow_control` | boolean | `false` | Block producers when a destination hits its memory limit. Off by default to avoid priority inversion of liveness/control traffic behind bulk. |
| `activemq_use_cache` | boolean | `false` | Enable the broker's in-memory dispatch cache. Off — no measured throughput gain, saves memory. |
| `activemq_dedicated_task_runner` | boolean | `false` | Use a dedicated thread per broker task instead of a shared pool. |
| `activemq_journal_disk_syncs` | boolean | `false` | Force `fsync` on each KahaDB journal write. Off trades durability-on-crash for I/O throughput. |
| `activemq_gc_inactive_destinations` | boolean | `true` | Garbage-collect idle queues/topics after `activemq_inactive_timeout_before_gc`. |
| `activemq_inactive_timeout_before_gc` | int | `15000` | Milliseconds of inactivity before an idle destination becomes eligible for GC. |
| `activemq_destination_purge_period` | int | `2500` | Milliseconds between broker sweeps that purge expired messages. |
| `activemq_destination_memory_limit` | long | `256 MB` | Per-destination memory ceiling (any single queue/topic). |
| `broker_memory_limit` | long | **computed** | Broker-wide message memory ceiling. Computed default = `max(512 MB, maxHeap/2)`. |
| `broker_store_limit` | long | `8 GB` | Maximum disk for the KahaDB persistent store. |
| `broker_temp_limit` | long | `4 GB` | Maximum disk for broker temporary (spooled) data. |
| `broker_message_ttl` | int | `5` | Default message time-to-live, **in minutes**; expired messages are discarded. |
| `broker_bridge_connections` | int | `1` | Number of parallel network-bridge connectors per remote peer broker (spreads load across sockets). |
| `broker_bridge_prefetch` | int | `100` | Prefetch limit on the **control** inter-broker bridge connector. |
| `broker_bridge_prefetch_data` | int | `10` | Prefetch limit on **data** bridge connectors, bounding how much bulk is committed ahead of later traffic on the data socket. |
| `broker_control_bridge_split` | boolean | `true` | Split each broker bridge group by destination type: connector 0 forwards queues only (the MsgEvent control fabric), connectors 1..N topics only (the data plane). See [Control/Data Plane Isolation](../design/control-dataplane-isolation.md). |
| `broker_bridge_decrease_consumer_priority` | boolean | `true` | Decrease a bridged consumer's priority by hop count so ActiveMQ's demand-forwarding prefers the fewest-broker-hop (direct) path. This gives [cost-aware routing](../architecture/dynamic-routing.md) a stable baseline to deliberately override when the direct link is the slow one. |
| `broker_tls_ready_probe` | boolean | `true` | Probe the broker TLS listener for readiness before proceeding through the boot state machine. |
| `broker_tls_ready_wait_ms` | long | `15000` | Maximum milliseconds to wait for the TLS listener to become ready when probing. |

### Failover / reconnect

Client-side failover transport behavior. Note these are read as **strings** and parsed downstream.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `failover_reconnect_delay` | String (ms) | `"5000"` | Initial delay before a failover reconnect attempt. |
| `max_reconnect_attempts` | String (int) | `"5"` | Maximum failover reconnect attempts before giving up. |
| `use_exponential_backOff` | String (bool) | `"false"` | Use exponential backoff between failover reconnect attempts. |

---

## Certificates, keystore & TLS material

Where the node finds its TLS keystore/truststore and how it generates message-signing keys.
The keystore/truststore getters have **no default** (unset = the framework generates/derives
material rather than loading a supplied store).

| Parameter | Type | Default | Description |
|---|---|---|---|
| `keystorefile` | String | *unset* | Path to the TLS keystore file. **Unset = generated in-memory.** |
| `keystorepwd` | String | *unset* | Password for the keystore. |
| `truststorefile` | String | *unset* | Path to the TLS truststore file. **Unset = derived.** |
| `truststorepwd` | String | *unset* | Password for the truststore. |
| `messagekeysize` | int | `2048` | RSA key size (bits) for generated message-signing certificates; values `<512` are refused and clamped to 512. |
| `public_key_directory` | String | `cresco-data/public-keys` | Directory where peer public keys/certificates are cached. |

---

## Discovery

How nodes find each other (TCP/UDP/static) and the timeouts governing each discovery mode.
See also the *Security & tenancy* table for the `discovery_secret_*` shared secrets.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `enable_tcp_discovery` | boolean | `true` | Enable the TCP discovery engine (peer-controller discovery). |
| `enable_udp_discovery` | boolean | `false` | Enable UDP broadcast discovery. Off by default. |
| `netdiscoveryport` | int | `32005` | Port the TCP/UDP discovery listener binds. |
| `netdiscoveryssl` | boolean | `false` | Enable SSL on discovery connections (currently gated off in code). |
| `discovery_port` | int | `32010` | Port advertised/used for local broker (agent) connections resolved via discovery. |
| `discovery_port_remote` | int | `32010` | Port used when connecting to a **remote** broker (multi-global / cross-host meshes). |
| `discovery_ipv4_agent_timeout` | int | `2000`¹ | Milliseconds to wait for IPv4 agent-discovery responses. ¹Read as `2000` in the state machine and `10000` in the net perf monitor. |
| `discovery_ipv6_agent_timeout` | int | `2000`¹ | Milliseconds to wait for IPv6 agent-discovery responses. ¹Same dual-default caveat as the IPv4 timeout. |
| `discovery_static_agent_timeout` | int | `10000` | Timeout (ms) for **static** (explicitly-configured host) agent discovery. |
| `discovery_static_region_timeout` | int | `10000` | Timeout (ms) for static region discovery. |
| `discovery_static_global_timeout` | int | `10000` | Timeout (ms) for static global-controller discovery. |
| `peer_discovery_timeout` | int | `5000` | Timeout (ms) for discovering regional peer controllers. |

---

## Region / global federation

Static wiring and liveness of the region↔global (and region↔region peer) links. When a
static host is set it overrides dynamic discovery for that link.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `regional_controller_host` | String | *unset* | Static hostname/IP of the regional controller an agent should attach to (overrides discovery). |
| `regional_controller_port` | int | `32005` | Port for the regional-controller connection. |
| `global_controller_host` | String | *unset* | Static hostname/IP of the global controller a region should attach to (overrides discovery). |
| `global_controller_port` | int | `32005` | Port for the global-controller connection (also used to key same-host multi-global meshes). |
| `regional_peers` | String | *unset* | Comma-separated list of peer regional-controller addresses to maintain mesh links with. |
| `localpluginrepo` | String | *unset* | Local plugin-repository directory the global controller uses for plugin discovery/distribution. |
| `forward_global_kpi` | boolean | `true` | Forward regional KPI/metrics up to the global controller. |
| `gc_connect_retry` | int | `25` | Maximum attempts to connect to the global controller after discovery before giving up. |
| `region_ping_interval` | long | `5000` | Milliseconds between region→global liveness pings. |
| `region_ping_timeout` | long | `5000` | Milliseconds to wait for a region→global ping response. |
| `region_ping_retries` | int | `2` | Consecutive ping misses tolerated within an interval before recording a failure. |
| `comm_watchdog_interval` | long | `5000` | Milliseconds between local broker/discovery health checks in the region watchdog. |

---

## Watchdog, state machine & timing

General liveness/retry timing across the boot state machine and the region/global/agent
watchdogs. `watchdog_interval`, `watchdog_interval_delay`, and `period_multiplier` are read
in several watchers with the same defaults shown.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `watchdog_interval` | long | `15000` | Base watchdog tick interval (ms) for the state machine and region/global/agent health watchers. |
| `watchdog_interval_delay` | long | `5000` | Initial delay (ms) before the first watchdog tick. |
| `watchdog_period` | int | `5000` | Watchdog period (ms) recorded on node records persisted to the controller DB. |
| `watchdog_scan_multiplier` | long | `3` | Multiplier on `watchdog_interval` setting the node-status scan cadence. Falls back to `period_multiplier` when unset. |
| `watchdog_stale_multiplier` | int | `10` | Multiplier on a node's `watchdog_period` setting the staleness threshold. Falls back to `period_multiplier` when unset. |
| `period_multiplier` | int / long | `10` (int) / `3` (long)¹ | **Legacy.** Fallback for the two multipliers above, kept for existing configs. ¹Its two readers historically used different defaults — that ambiguity is why the split keys exist; prefer them. |
| `state_update_full_resend_ticks` | int | `4` | Force a full config export every N state-update ticks, so a lost first export self-heals instead of leaving a node's rows missing at its parent. |
| `retrywait` | long | `3000` | Milliseconds to wait between state-machine retry attempts. |
| `agent_ping_interval` | long | `5000` | Milliseconds between agent-liveness pings (agent health watcher). |
| `agent_ping_timeout` | long | `5000` | Milliseconds to wait for an agent ping response. |

---

## Health checks

Felix-Health-Check-based monitoring: overall executor cadence, disk/memory thresholds, and
the parent-link / subtree / link-quality checks. All intervals/thresholds below have shipped
defaults; the subsystem runs out of the box.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `health_check_interval_ms` | long | `5000` | Default interval at which registered health checks execute. |
| `health_summary_interval_ms` | long | `30000` | Interval for logging the periodic health rollup summary. |
| `health_grace_ms` | long | `20000` | How long a check may stay TEMPORARILY_UNAVAILABLE before being promoted to CRITICAL. |
| `health_sticky_ms` | long | `60000` | Window during which a recently-recovered check is still surfaced as WARN (dampens flapping). |
| `health_disk_floor_mb` | long | `100` | Free-disk floor (MB); below this, disk health is CRITICAL. |
| `health_mem_warn_pct` | int | `85` | Heap-usage percentage at which memory health becomes WARN. |
| `health_mem_crit_pct` | int | `95` | Heap-usage percentage at which memory health becomes CRITICAL. |
| `health_link_interval_sec` | long | `5` | Seconds between parent-link health-check executions. |
| `health_link_grace_sec` | long | `10` | Grace window (s) for link staleness before escalating to CRITICAL. |
| `health_link_stale_ms` | long | `0` | Staleness threshold (ms) for a parent link's last pong (`0` = derived/disabled). |
| `health_link_region_ping` | boolean | `false` | Drive region→global link health from an application-level ping instead of broker-bridge detection. |
| `health_subtree_interval_sec` | long | `10` | Seconds between child-health rollup executions. |
| `health_subtree_stale_ms` | long | `30000` | Staleness threshold (ms) for child health reports before they're treated as stale. |

### Link-quality thresholds

Warn levels for the parent-link quality check; exceeding any degrades reported link quality.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `link_quality_rtt_warn_ms` | double | `50.0` | Smoothed RTT (ms) above which link quality is degraded. |
| `link_quality_jitter_warn_ms` | double | `25.0` | Jitter (ms) above which link quality is degraded. |
| `link_quality_sendlat_warn_ms` | double | `25.0` | Producer send-latency (ms) above which link quality is degraded. |
| `link_quality_backlog_warn` | long | `1000` | Broker backlog (pending messages) above which link quality is degraded. |

---

## Metrics & measurements

Timeouts for capability/metrics/resource RPCs and fan-outs, plus the sysinfo sampler.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `capability_rpc_timeout_ms` | int | `2500` | Timeout for a single plugin capability RPC. |
| `capability_fanout_timeout_ms` | int | `12000` | Timeout for a capability-inventory fan-out across remote agents. |
| `metrics_rpc_timeout_ms` | int | `2500` | Timeout for a single plugin metrics RPC. |
| `metrics_fanout_timeout_ms` | int | `12000` | Timeout for a metrics fan-out across remote agents. |
| `resource_rpc_timeout_ms` | int | `3000` | Timeout for a sysinfo resource RPC. |
| `sysinfo_metrics_ttl_ms` | int | `5000` | Cache TTL for sysinfo metrics JSON (avoids repeated expensive OSHI host queries). |
| `perftimer` | long | `10000` | Interval (ms) for the performance sampler (sysinfo `PerfSysMonitor` and net perf monitor). |
| `inode_id` | String | `netdiscovery_inode` | Identifier the net perf monitor uses for its measurement inode. |
| `resource_id` | String | `netdiscovery_resource` | Identifier the net perf monitor uses for its measurement resource. |

---

## Network link metrics & auto-tuning

Adaptive tuning of socket buffers, read-chunk sizes, and per-link bridge connection counts
based on live link metrics (RTT, send-latency, backlog, BDP). **Auto-tuning is off by
default** (`net_autotune=false`); the min/base/max profile values still define the static
buffer sizes used when auto-tuning is off.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `net_autotune` | boolean | `false` | Master switch for network-aware auto-tuning of buffers, chunk sizes, and connection counts. |
| `net_autotune_interval_sec` | int | `5` | Seconds between metric-sampling and tuning-adjustment cycles. |
| `net_autotune_cooldown_ms` | long | `15000` | Minimum interval between successive connection scale actions on the same link. |
| `net_autotune_backlog_high` | long | `500` | Broker backlog (pending messages) above which the tuner scales connections up. |
| `net_autotune_sendlat_high_ms` | double | `20.0` | Send-latency (ms) above which the tuner scales connections up. |
| `net_autotune_rtt_high_ms` | double | `40.0` | Smoothed-RTT (ms) above which the tuner scales connections up. Catches dataplane byte floods (e.g. [stunnel tunnels](../architecture/tunnel-tracing.md)) that inflate RTT but not send-latency/backlog. |
| `net_trace_hops` | boolean | `true` | Broker stamps `cresco_hops` on `cresco_trace`-marked data-plane messages, so a tunnel's bytes carry the full broker path they traversed ([tunnel tracing](../architecture/tunnel-tracing.md)). Only marked messages pay. |
| `stunnel_beacon_ms` | long | `3000` | How often each stunnel tunnel pushes its existence/status beacon on the subscribable `stunnel_trace` stream. |
| `net_autotune_sendlat_low_ms` | double | `2.0` | Send-latency (ms) below which idle links are scaled down. |
| `net_autotune_bdp_safety` | double | `3.0` | Bandwidth-delay-product multiplier used to size socket buffers with headroom. |
| `net_link_speed_bps` | long | `0` | Configured uplink capacity (bits/s) used as a BDP cap; `0` = unknown. |
| `net_source_routing` | boolean | `false` | Carry and honor a source-route waypoint stack (`srcroute`) so a flow can be steered along a chosen multi-hop path. See [Dynamic Routing](../architecture/dynamic-routing.md). |
| `net_cost_routing` | boolean | `false` | Measure candidate-path latency, select the lowest-cost path (Dijkstra over the pushed link-state graph), and **inject** its route at the origin. Requires `net_source_routing` to enforce multi-hop paths. |
| `net_route_advertise_interval_sec` | int | `5` | How often each controller pushes its link-state advertisement onto the `GLOBAL` data-plane topic. |
| `net_route_stale_sec` | int | `20` | Age after which a silent node's advertisement is dropped from the mesh-wide `RouteView`. |
| `net_route_hysteresis_ms` | double | `10.0` | Margin (ms) by which an alternate path must beat the incumbent before the selection flips (anti-flap). |
| `net_metrics_log` | boolean | `false` | Log detailed per-link network metrics each tuning cycle. |
| `net_connections_per_link` | int | `1` | Initial number of concurrent bridge connections per remote link. |
| `net_connections_min` | int | `1` | Floor on concurrent connections per link. |
| `net_connections_max` | int | `16` | Ceiling on concurrent connections per link. |
| `net_socket_buffer_bytes` | int | `4 MB` | Base TCP socket send/receive buffer size. |
| `net_socket_buffer_min` | int | `256 KB` | Floor the auto-tuner may set for the socket buffer. |
| `net_socket_buffer_max` | int | `32 MB` | Ceiling the auto-tuner may set for the socket buffer. |
| `net_read_chunk_bytes` | int | `256 KB` | Base per-read block size for network I/O. |
| `net_read_chunk_min` | int | `16 KB` | Floor the auto-tuner may set for the read-chunk size. |
| `net_read_chunk_max` | int | `4 MB` | Ceiling the auto-tuner may set for the read-chunk size. |
| `net_write_high_water_bytes` | int | `2 MB` | Outbound-buffer high-water mark that triggers write backpressure. |

---

## Coordinator decentralization (region-first, multi-global)

Decompose the single static global into a region-first mesh with elected/sharded coordinators. All flags
default to the prior single-static-global behaviour; see the
[Coordinator Decentralization plan](../design/coordinator-decentralization-plan.md).

| Parameter | Type | Default | Description |
|---|---|---|---|
| `global_optional` | boolean | `false` | Region-first autonomy: a region comes up and peers with **no global present**, starting its discovery engine + peer maintenance immediately and bounding the global-join instead of blocking. |
| `global_connect_attempts` | int | `3` | With `global_optional`, how many times to try joining a global before settling into region-first `REGION` steady state. |
| `failure_phi_suspect` | double | `4.0` | φ-accrual suspicion at which SWIM indirect probing of a peer begins. |
| `failure_phi_dead` | double | `8.0` | φ-accrual suspicion at which (after SWIM also fails) a peer is concluded unreachable. |
| `failure_swim_k` | int | `2` | Number of other peers asked to indirect-probe a suspected peer (SWIM). |
| `coordinator_expected` | int | `0` | Stable coordinator-cluster size for quorum (2f+1). `0` = infer from the high-water mark. Set it to the intended count so a lone survivor cannot self-commit. |
| `coordinator_election_policy` | String | `identity` | Leader policy: `identity` (lowest coordinator path) or `centroid` (k-center: most central over the learned latency graph). |
| `coordinator_lease_sec` | int | `15` | Coordinator heartbeat/lease window; a coordinator silent this long drops from the live set. |
| `security_peer_federation` | boolean | `false` | Two independently-rooted regions cross-trust as **equals** (bilateral CA exchange, identity preserved) instead of one being subordinated under a common issuer — removes the need for a global as trust broker. |

---

## Data plane

The high-throughput event data plane and its sharding/journaling. Sharding splits data-plane
traffic across N topics for parallel inter-broker transfer.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `dataplane_shards` | int | `1` | Number of shard topics the data plane is split into (`1` = unsharded). |
| `dataplane_shard_topic` | String | `global.event` | Base topic name for shards (e.g. `global.event.0..N-1`). |
| `dataplane_parallel_connections` | boolean | **computed** (`shards > 1`) | Use a dedicated broker connection per shard instead of multiplexing all shards over one session. Default is `true` when `dataplane_shards > 1`, else `false`. |
| `dataplane_io_buffer_bytes` | int | `64 KB` | I/O buffer size for data-plane message handling (wsapi data-plane path). |
| `journal_dir` | String | **computed** | Directory for the data-plane producer-worker journal. Default resolves to a path under the working directory / node data location. |
| `dp-journal_dir` | String | **computed** | Directory for the data-plane service journal (KahaDB). Default resolves under the node data location. |

---

## Producer / consumer & QoS

Producer-worker pools, retry behavior, prefetch limits, exclusive-consumer defaults, and the
tiered-priority TTLs that keep liveness/control traffic ahead of bulk.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `queue_prefetch_limit` | int | `100` | Consumer prefetch size for **queue** consumers (the control inboxes). |
| `topic_prefetch_limit` | int | `100` | Consumer prefetch size for **topic** consumers. |
| `controlplane_dedicated_connection` | boolean | `true` | Give the control-plane sender its own TCP connection, so bulk payload cannot delay it at the wire. Ignored for `vm://` (in-JVM). |
| `agentconsumer_dedicated_connection` | boolean | `true` | Give the agent inbox consumer (all inbound control, including RPC replies) its own TCP connection. Ignored for `vm://`. |
| `dataplane_session_wait_sec` | int | `30` | Bound (s) on waiting for the messaging plane before a data-plane session request returns `null` instead of blocking its caller. |
| `prefetch_rate_multiplier` | double | `2.5` | Multiplier on prefetch used by the pending-message-limit strategy (controls non-persistent message eviction). |
| `queue_all_consumers_exclusive` | boolean | `true` | Make all queue consumers exclusive (one active consumer per destination). |
| `topic_all_consumers_exclusive` | boolean | `true` | Make all topic consumers exclusive by default. |
| `activeproducerworker_default_priority` | int | `4` | Default JMS priority for messages sent by a producer worker (ActiveMQ default is 4). |
| `activeproducerworker_ttl` | long | `300000` | TTL (ms) on messages sent by producer workers (5 min). |
| `controlplane_ttl` | long | `300000` | TTL (ms) on control-plane (liveness/control) messages (5 min). |
| `agentproducer_data_queue` | int | `1000` | Bounded queue size for bulk data-transfer messages in the agent producer. |
| `agentproducer_data_threads` | int | `8` | Worker-thread count for the bulk data-transfer executor. |
| `agentproducer_max_send_retries` | int | `3` | Maximum send retries before a message send is failed. |
| `agentproducer_initial_retry_delay` | long | `500` | Initial backoff delay (ms) for send retries. |
| `agentproducer_worker_cleanup_delay` | long | `15000` | Delay (ms) after producer init before the idle-worker cleanup task first runs. |
| `agentproducer_worker_cleanup_interval` | long | `30000` | Interval (ms) between idle producer-worker cleanup scans. |

---

## Database (controller state store)

The embedded controller DB (Derby by default). `db_username`/`db_password` have **no
default** (unset = no credentials, appropriate for the embedded Derby driver).

| Parameter | Type | Default | Description |
|---|---|---|---|
| `db_name` | String | `cresco-controller-db` | Logical database name. |
| `db_driver` | String | `org.apache.derby.jdbc.EmbeddedDriver` | JDBC driver class (HSQLDB alternative is present but commented out). |
| `db_jdbc` | String | **computed** | JDBC URL. Computed default = `jdbc:derby:<dbPath>;create=true` from the resolved DB path. |
| `db_username` | String | *unset* | DB username. **Unset = none** (embedded Derby needs no credentials). |
| `db_password` | String | *unset* | DB password. **Unset = none.** |

---

## Repository & data paths

Where the node stores its data, plugin cache, and jar repository.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `cresco_data_location` | String | `cresco-data` | Root directory for this node's persistent data (DB, keys, journals). |
| `agent_data_directory` | String | **computed** | Agent data directory; default derives from `cresco_data_location` / the default filesystem working directory. |
| `tmp_data` | String | *unset* | Override for the host temporary-data directory (host application). |
| `repo_dir` | String | `repo` | Directory served by the **repo** plugin as the jar/plugin repository root. |
| `repo_cache_dir` | String | `cresco-data/agent-repo-cache` | Local cache directory for plugins the agent downloads from a repo. |

---

## wsapi (WebSocket API gateway)

Settings for the wsapi plugin: listener port, TLS material, socket tuning, and the service
key that authenticates HTTP-upgrade requests.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `wsapi_port` | String (int) | `8282`¹ | Port the wsapi WebSocket server binds. ¹Read as a string; supply as plain text. |
| `wsapi_keystore_password` | String | *unset* | Password for the wsapi TLS keystore. **Unset = generated self-signed default.** |
| `wsapi_ssl_provider` | String | `JDK` | TLS engine: `JDK` (standard Java SSL) or `OPENSSL` (native/BoringSSL). |
| `cresco_service_key` | String | *unset* | API key/token required on the WebSocket HTTP-upgrade handshake. **Unset = no key required.** |
| `wsapi_read_chunk_bytes` | int | `256 KB` | Max bytes per socket read (larger reduces syscalls on bulk streams). |
| `wsapi_socket_buffer_bytes` | int | `4 MB` | TCP socket send/receive buffer size for wsapi connections. |
| `wsapi_write_high_water_bytes` | int | `2 MB` | Outbound-buffer high-water mark that triggers backpressure. |

---

## stunnel (secure TCP tunneling)

Socket tuning, health checking, and session/demux bounds for the stunnel plugin. See
[stunnel Data Path Design](../design/stunnel-dataplane-design.md) for how these interact.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `stunnel_read_chunk_bytes` | int | `256 KB` | Max bytes per socket read on tunnel sockets, and so the max size of one dataplane payload message. |
| `stunnel_socket_buffer_bytes` | int | `4 MB` | TCP socket send/receive buffer size for tunnel sockets. |
| `stunnel_write_high_water_bytes` | int | `2 MB` | Outbound-buffer high-water mark that triggers backpressure on a tunnel. |
| `stunnel_beacon_ms` | long | `3000` | Interval (ms) of the per-tunnel existence beacon on the `stunnel_trace` stream. |
| `stunnel_health_check_interval_sec` | long | `5` | Tunnel health-check period (seconds), scheduled as a **fixed delay**. |
| `stunnel_health_failure_threshold` | int | `3` | Consecutive health-check failures before a verification probe and tunnel teardown. |
| `stunnel_health_probe_timeout_ms` | long | `10000` | RPC budget (ms) for a health probe. A per-tunnel `watchdog_timeout` overrides it. |
| `stunnel_dst_init_timeout_ms` | long | `10000` | RPC budget (ms) for `configdstsession` when a client connects to a SRC listener. |
| `stunnel_dst_init_max_concurrent` | int | `64` | Concurrent DST session initiations; connects beyond this fail fast rather than parking threads. |
| `stunnel_demux_buffer_max` | int | `256` | Messages buffered per client between `expect()` and handler registration; overflow fails that session. |
| `stunnel_demux_open_timeout_ms` | long | `15000` | Bound (ms) on opening a tunnel's dataplane consumer, so a wedged dataplane cannot park the plugin message thread. |

---

## Notes on defaults

- **`min`/`max`/`base` tuning triples** (`net_socket_buffer_*`, `net_read_chunk_*`,
  `net_connections_*`) only diverge when `net_autotune=true`. With auto-tuning off, the base
  value (`net_socket_buffer_bytes`, `net_read_chunk_bytes`, `net_connections_per_link`) is used
  as-is and the min/max bounds are inert.
- **Computed defaults** (`broker_memory_limit`, `db_jdbc`, `journal_dir`, `dp-journal_dir`,
  `agent_data_directory`, `dataplane_parallel_connections`) are derived from the JVM heap, the
  resolved data location, or another parameter — set them explicitly only to override the
  derivation.
- **Dual-default parameters** (`discovery_ipv4_agent_timeout`, `discovery_ipv6_agent_timeout`,
  `period_multiplier`, `enable_repo`, `enable_wsapi`) are read in more than one place with
  differing shipped defaults; the effective value depends on which subsystem reads it first.
  Set them explicitly if the exact value matters to you.
