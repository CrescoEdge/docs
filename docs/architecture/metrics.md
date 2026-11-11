# Metrics & Measurements

Cresco has a single, unified measurement model: **every bundle** — the controller and every plugin —
registers its metrics in one Micrometer registry and exposes them through one contract, and the
controller aggregates them locally, region-wide, and global-wide. This is distinct from
[health](health.md), which consumes measurements to form pass/fail verdicts.

This page is exhaustive: it documents the measurement machinery, then **every metric the fabric
emits** — its name, group, type, unit, exactly how it is measured, and what it means.

## The model

| Piece | Role |
|-------|------|
| `MeasurementEngine` | per-bundle façade over a Micrometer registry; register gauges/timers/counters/distribution-summaries and read them back with `getAllMetrics()` |
| `CMetric` | metadata wrapper for one metric — its `name`, `group`, measure class, and scope (`NODE`/`APP`/`EDGE`) |
| `CrescoMeterRegistry` | a Micrometer `DropwizardMeterRegistry` (one per bundle), obtained via `plugin.getCrescoMeterRegistry()` |

Metrics are organized into **groups** — the metric analogue of health tags. The controller populates
`jvm`, `controller`, `cep`, `netlink`, `regional`, and `global`; each plugin contributes its own
(`sysinfo` → `processor`/`memory`/`disk`/`network`/`sensors`/`power`/`system`/`gpu`/`devices`,
`executor` → `executor`, `filerepo` → `filerepo`, `repo` → `repo`, `stunnel` → `stunnel`,
`wsapi` → `wsapi`).

### Measure classes

Every metric carries a `CMetric.MeasureClass` that determines how it is stored and serialized:

| Measure class | Backing | Notes |
|---|---|---|
| `GAUGE_INT` | `AtomicInteger` | integer point-in-time value |
| `GAUGE_LONG` | `AtomicLong` | long point-in-time value (byte counts, timestamps) |
| `GAUGE_DOUBLE` | Guava `AtomicDouble` | fractional value (loads, rates, temperatures) |
| `GAUGE_AUTO` | Micrometer meter | value read straight from a Micrometer `Gauge.builder`/binder meter adopted via `setExisting` |
| `TIMER` | Micrometer `Timer` | mean/max/total/count of recorded durations |
| `DISTRIBUTION_SUMMARY` | Micrometer `DistributionSummary` | mean/max/total/count of recorded amounts |
| `COUNTER` | Micrometer `Counter` | monotonic count |

`CMetric.Type` is the scope: `NODE` (default — a node-level metric), `APP` (needs an inode + resource
id), `EDGE` (needs an edge id). All engine-registered metrics are `NODE`; the per-process and
per-runner streams use `APP`.

### Registration & update API

- `setGauge(name, description, group, measureClass)` — create a gauge (idempotent; a repeated name is
  a no-op that returns `false`).
- `updateIntGauge` / `updateLongGauge` / `updateDoubleGauge(name, value)` — set the current value.
- `setTimer(name, description, group)` + `updateTimer(name, startNanos)` — records
  `System.nanoTime() − startNanos`.
- `setDistributionSummary(name, description, group)` + `updateDistributionSummary(name, value)`.
- `setExisting(name, group)` — adopt a meter already on the shared registry (a Micrometer binder
  meter, or a `Gauge.builder` gauge with a live supplier) so it surfaces in `getAllMetrics()`.

## The single plugin contract

Every metric-bearing bundle answers the same EXEC action:

```java
case "getmetrics":
    incoming.setParam("metrics", gson.toJson(measurementEngine.getAllMetrics()));
    incoming.setParam("status", "10");
    return incoming;
```

`getAllMetrics()` returns `Map<group, List<row>>`. Each **row** is a flat `Map<String,String>` whose
keys depend on the measure class — read live on every call:

| Measure class | Row keys |
|---|---|
| `GAUGE_INT` / `GAUGE_LONG` / `GAUGE_DOUBLE` / `GAUGE_AUTO` | `name`, `class` (meter type, e.g. `GAUGE`), `type` (`NODE`/…), `value`, `group` |
| `TIMER` | `name`, `class`, `type`, `mean`, `max`, `totaltime`, `count`, `group` |
| `DISTRIBUTION_SUMMARY` | `name`, `class`, `type`, `mean`, `max`, `totalAmount`, `count`, `group` |
| `COUNTER` | `name`, `class`, `type`, `count`, `group` |

This is the wire schema for the whole fabric.

## Mesh-wide aggregation — `getmetricinventory`

`getmetricinventory` is the single aggregating surface (metrics' analogue of
[`gethealthinventory`](health.md#querying-health-gethealthinventory)). It merges the controller's own
metrics + every local plugin's `getmetrics` + an optional resource summary, and can **fan out** by
scope:

| Scope | Returns |
|-------|---------|
| `node` (default) | this node only |
| `region` | this node + every child agent in its region |
| `global` | the whole mesh |

Parameters: `action_scope` = `node`\|`region`\|`global`; `action_include_plugins` (default **true** —
set `false` to return only controller metrics); `action_include_resource` (default **false** — opt-in;
adds a sysinfo CPU/memory/disk resource summary at the cost of an extra RPC).

**How the node inventory is built.** The controller assembles `metrics_by_source`:

1. Its own `MeasurementEngine.getAllMetrics()` keyed `<region>_<agent>:io.cresco.agent.controller`
   (the `jvm`, `controller`, `cep`, `regional`, `global`, `netlink` groups below).
2. Each local plugin's metrics — the controller sends the `getmetrics` EXEC to every plugin id on the
   node (timeout `metrics_rpc_timeout_ms`, default 2500 ms), keyed `<region>_<agent>:<pluginId>`.
   Plugins that don't answer are silently skipped.
3. If requested, a `resource_summary` (sysinfo CPU/memory/disk totals, timeout
   `resource_rpc_timeout_ms`, default 3000 ms).

**How the fan-out works.** For `region`, the target set is every agent in this region; for `global`,
every agent in every region. Each child is queried **concurrently** (one daemon thread per child)
with a node-scoped `getmetricinventory`, timeout `metrics_fanout_timeout_ms` (default 12000 ms). The
join deadline is that timeout + 1000 ms; unreachable/slow nodes are skipped. Whole-mesh latency is
therefore bounded by the slowest single node, not the sum of all nodes.

Final shape: `{ node, metrics_by_source:{…}, [resource_summary], scope, [children:{…}] }`. Both
[client libraries](../clients/overview.md) expose `get_metric_inventory(...)` to pull it.

---

# The full metric catalog

Every metric the fabric emits, grouped by source. Format: **`metric.name`** (group, type, unit) —
how it is measured — what it means.

## Controller — native metrics

Registered by `PerfControllerMonitor` on the controller's `MeasurementEngine` at startup.

| Metric | Group / type | How measured | Meaning |
|---|---|---|---|
| **`message.transaction.time`** | `controller`, TIMER | `updateTimer(...)` in `MsgRouter` records `nanoTime − messageTimestamp` for each routed message | End-to-end message-routing latency through the controller (mean / max / total / count) |
| **`cep.queries.active`** | `cep`, GAUGE | supplier reads `DataPlaneServiceImpl.getActiveCEPCount()` | Number of active Complex-Event-Processing (Siddhi) queries in the embedded in-process CEP engine |
| **`brokered.agent.count`** | `regional`, GAUGE | supplier reads `controllerEngine.getBrokeredAgents().size()` | Agents currently brokered by this regional controller (reads 0 on a leaf agent) |
| **`reachable.agent.count`** | `global`, GAUGE | supplier reads `controllerEngine.reachableAgents().size()` | Agents reachable in this controller's view of the mesh |
| **`incoming.candidate.brokers`** | `global`, GAUGE | supplier reads `controllerEngine.getIncomingCanidateBrokers().size()` | Depth of the discovery candidate-broker queue awaiting processing |
| **`dataplane.broker.discarded`** | `dataplane`, COUNTER | `CrescoDiscardAccountingBroker` counts the broker's slow-subscriber discards | Dataplane messages this node's broker discarded for slow topic subscribers (only when backpressure is off: `dataplane_backpressure=false` or `activemq_topic_producer_flow_control=false`; logged too) |
| **`dataplane.broker.slow.consumers`** | `dataplane`, COUNTER | the broker's slow-consumer advisories | Slow-subscriber episodes (more than prefetch pending); with backpressure these slow writers, they lose nothing |
| **`dataplane.stream.gap`** | `dataplane`, COUNTER | `StreamSequence` checks each stream's `cresco_seq` at the subscriber's controller | Stream messages this node's subscribers did not get; the first message after a gap carries `cresco_gap` ([Data Plane](dataplane.md#stream-sequence-numbers-loss-is-never-silent)) |

## Controller: what became of each message

Group `msg` (and `broker`), all COUNTERs unless noted, registered by `MsgMetrics`. They account for every control
message (MsgEvent) a node had in hand; the dataplane has its own (`dataplane.*`). The reason or cause is the last part
of the name, because the inventory keys a meter by its name.

| Metric | Meaning |
|---|---|
| **`msg.originated`** | messages this node originated (its controller's, a plugin's, or an API user's through its wsapi), replies included |
| **`msg.relayed`** | messages taken off the wire and sent on toward another node |
| **`msg.delivered`** | messages handed to their recipient on this node: its controller, a plugin, or the call waiting for a reply |
| **`msg.held`**, **`msg.hold.full`** | messages the inbox held until it could judge them, and messages it could not hold for lack of room |
| **`msg.refused`**, **`msg.refused.<reason>`** | messages that failed authentication, in all and by reason (`unsigned`, `not_for_this_node`, `no_recipient`, `too_old`, `ahead_of_clock`, `replay`, `replay_memory_full`, `credential`, `revoked`, `session_key`, `bad_signature`, `caller_mismatch`, `other`) |
| **`msg.dropped`**, **`msg.dropped.<cause>`** | messages that ended on this node without reaching their recipient, in all and by cause ([the causes](../operations/troubleshooting.md#message-dropped)) |
| **`msg.sign.failed`**, **`msg.trust.refused`**, **`msg.filepart.dropped`** | sends this node could not sign; trust data from a parent it refused; file parts it did not take (not counted as messages) |
| **`broker.caller.stripped`** | caller stamps this node's broker stripped (forged, or carried by a bridge that may not carry them) |
| **`broker.bridge.withheld`**, **`broker.bridge.discarded`** | messages a bridge demanded and was not given, or brought in for a destination its peer may not write |
| **`broker.anonymous.discarded`**, **`broker.recipient.withheld`** | messages from connections without a client certificate; messages withheld from a consumer that is not their recipient |
| **`broker.quarantine.connections`** (GAUGE) | connections without a verified identity held now |

On a quiet fabric, `msg.originated` summed over the nodes equals `msg.delivered + msg.refused + msg.dropped` summed
over them. What a broker loses between two nodes (a message that expired, a queue with no consumer) ends on no node, so
the sum then falls short by it: that is what the identity is for. An authorization denial (403) counts as delivered:
the recipient answered.

## Controller — JVM & system (Micrometer binders)

The controller binds four standard Micrometer binders (`JvmMemoryMetrics`, `JvmThreadMetrics`,
`ClassLoaderMetrics`, `ProcessorMetrics`) to its registry and adopts the following into the `jvm`
group. (`JvmGcMetrics` and `system.load.average.1m` are intentionally not adopted — the latter
misbehaves on Windows; sysinfo carries load average instead.)

| Metric | Meaning |
|---|---|
| **`jvm.memory.used`** | Bytes of JVM memory in use, per pool (heap / non-heap) |
| **`jvm.memory.committed`** | Bytes committed (guaranteed available), per pool |
| **`jvm.memory.max`** | Max bytes the JVM will attempt to use per pool (`-1` where unbounded) |
| **`jvm.buffer.memory.used`** | Bytes of direct/mapped NIO buffer memory in use |
| **`jvm.buffer.total.capacity`** | Total capacity of NIO buffers per pool |
| **`jvm.buffer.count`** | Number of buffers in the pool |
| **`jvm.threads.live`** | Live threads (daemon + non-daemon) |
| **`jvm.threads.daemon`** | Daemon threads |
| **`jvm.threads.peak`** | Peak live threads since JVM start |
| **`jvm.classes.loaded`** | Classes currently loaded |
| **`jvm.classes.unloaded`** | Classes unloaded since JVM start (counter) |
| **`system.cpu.count`** | Logical processors available to the JVM |
| **`system.cpu.usage`** | Recent whole-system CPU usage (0–1) |
| **`process.cpu.usage`** | Recent CPU usage of this JVM process (0–1) |

## Controller — network link metrics (`netlink`)

The `netlink` group carries **per-edge link quality**, one set of gauges per neighbor edge, named
`link.<region_agent>.<metric>` (non-alphanumerics in the path become `_`). Values are pushed on the
`AutoTuner` loop interval. These feed the [`link:quality`](health.md#linkquality-tags-link-linkquality)
health check, the auto-tuner control loop (which adapts socket buffers, block sizes, and bridge
connection counts to the bandwidth-delay product), and a composite link **cost** used for
performance-aware routing.

| Metric (per edge) | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`link.<path>.rtt_ms`** | GAUGE_DOUBLE, ms | Jacobson/Karels smoothed RTT (`srtt`, α=1/8) of the parent health-ping RPC | Smoothed round-trip latency to the neighbor |
| **`link.<path>.jitter_ms`** | GAUGE_DOUBLE, ms | Jacobson/Karels RTT variation (`rttvar`, β=1/4) computed alongside srtt | RTT variability / jitter of the edge |
| **`link.<path>.tx_mbps`** | GAUGE_DOUBLE, MB/s | windowed rate from a `LongAdder` fed by `dp_bytes` on the producer send path | Transmit throughput on the edge |
| **`link.<path>.rx_mbps`** | GAUGE_DOUBLE, MB/s | windowed rate from the receive-byte adder | Receive throughput on the edge |
| **`link.<path>.sendlat_ms`** | GAUGE_DOUBLE, ms | EWMA (α=0.2) of producer `send()` dwell time | Backpressure/congestion signal — rises when the broker is flow-controlled or memory-pressured |
| **`link.<path>.backlog`** | GAUGE_LONG, messages | broker `QueueSize` via JMX, sampled by the auto-tuner | Broker pending-message backlog on the edge (native congestion signal) |

Derived link values used internally by the cost-aware router but not published as gauges:
`rttHigh = srtt + 4·rttvar`, link `utilization` (tx bits ÷ NIC speed ceiling from OSHI), and a
composite `cost = rttHigh + sendLatEwma + backlog·0.1 + 50/throughput`.

## Plugin: `sysinfo` — host telemetry (richest set)

`sysinfo` registers 30 gauges on its own `MeasurementEngine` from OSHI (`SystemInfo` →
`HardwareAbstractionLayer`/`OperatingSystem`). All are always emitted (0 where the platform can't
report a value). `getmetrics` caches the serialized result for `sysinfo_metrics_ttl_ms` (default
5000 ms) because NIC/filesystem enumeration is expensive.

### Group `processor`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`system.cpu.count`** | GAUGE_INT | `cpu.getLogicalProcessorCount()` | Logical CPU count |
| **`system.cpu.load`** | GAUGE_DOUBLE, 0–1 | `cpu.getSystemCpuLoadBetweenTicks()` (tick deltas across calls, non-blocking) | System CPU load since last read |
| **`system.load.average.1m`** | GAUGE_DOUBLE | `cpu.getSystemLoadAverage(1)[0]` (`-1` if unsupported) | 1-minute load average |
| **`cpu.context.switches`** | GAUGE_LONG | `cpu.getContextSwitches()` | Context switches since boot |
| **`cpu.interrupts`** | GAUGE_LONG | `cpu.getInterrupts()` | Interrupts since boot |

### Group `memory`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`memory.total`** | GAUGE_LONG, bytes | `mem.getTotal()` | Total physical RAM |
| **`memory.available`** | GAUGE_LONG, bytes | `mem.getAvailable()` | Available physical RAM |
| **`memory.swap.total`** | GAUGE_LONG, bytes | `getVirtualMemory().getSwapTotal()` | Total swap |
| **`memory.swap.used`** | GAUGE_LONG, bytes | `getSwapUsed()` | Used swap |

### Group `disk`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`disk.total`** | GAUGE_LONG, bytes | Σ `fs.getTotalSpace()` over all file stores | Total filesystem space |
| **`disk.available`** | GAUGE_LONG, bytes | Σ `fs.getUsableSpace()` | Usable filesystem space |
| **`disk.read.bytes`** | GAUGE_LONG, bytes | Σ `disk.getReadBytes()` over all disk stores | Bytes read across all disks since boot |
| **`disk.write.bytes`** | GAUGE_LONG, bytes | Σ `disk.getWriteBytes()` | Bytes written across all disks since boot |

### Group `network`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`net.bytes.sent`** | GAUGE_LONG, bytes | Σ `net.getBytesSent()` over all NICs | Bytes sent, all interfaces |
| **`net.bytes.recv`** | GAUGE_LONG, bytes | Σ `net.getBytesRecv()` | Bytes received, all interfaces |
| **`net.packets.sent`** | GAUGE_LONG | Σ `net.getPacketsSent()` | Packets sent, all interfaces |
| **`net.packets.recv`** | GAUGE_LONG | Σ `net.getPacketsRecv()` | Packets received, all interfaces |

### Group `sensors`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`sensor.cpu.temperature`** | GAUGE_DOUBLE, °C | `sensors.getCpuTemperature()` (0 if unavailable) | CPU temperature |
| **`sensor.cpu.voltage`** | GAUGE_DOUBLE, V | `sensors.getCpuVoltage()` (0 if unavailable) | CPU voltage |
| **`sensor.fan.max.rpm`** | GAUGE_INT, RPM | max of `sensors.getFanSpeeds()` (0 if unavailable) | Fastest fan speed |

### Group `power`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`power.remaining.percent`** | GAUGE_DOUBLE, 0–100 | mean `ps.getRemainingCapacityPercent()` ×100 (`-1` if no battery) | Battery remaining |
| **`power.remaining.seconds`** | GAUGE_DOUBLE, s | `getTimeRemainingEstimated()` (`-1` on AC/unknown) | Estimated runtime on battery |

### Group `system`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`system.uptime.seconds`** | GAUGE_LONG, s | `os.getSystemUptime()` | OS uptime |
| **`system.process.count`** | GAUGE_INT | `os.getProcessCount()` | Running processes (OS-wide) |
| **`system.thread.count`** | GAUGE_INT | `os.getThreadCount()` | Running threads (OS-wide) |

### Group `gpu`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`gpu.count`** | GAUGE_INT | count of `hal.getGraphicsCards()` | Graphics cards detected |
| **`gpu.vram.total`** | GAUGE_LONG, bytes | Σ `g.getVRam()` | Total VRAM capacity (OSHI cannot report live GPU load) |

### Group `devices`
| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`display.count`** | GAUGE_INT | `hal.getDisplays().size()` | Attached displays |
| **`usb.device.count`** | GAUGE_INT | `hal.getUsbDevices(false).size()` | Connected USB devices |
| **`soundcard.count`** | GAUGE_INT | `hal.getSoundCards().size()` | Sound cards |

## Plugin: `executor`

### Inventory gauges (group `executor`)
| Metric | Type | How measured | Meaning |
|---|---|---|---|
| **`executor.runners.configured`** | GAUGE_INT | `runnerEngine.getRunnerCount()` | Configured runners (running or not) |
| **`executor.runners.active`** | GAUGE_INT | `runnerEngine.getActiveCount()` | Runners currently executing a process |

### Per-runner process telemetry (dataplane stream, opt-in `metrics=true`)
When a runner is configured with `metrics=true`, a `RunnerMetrics` thread samples the launched
process tree (root PID + descendants via `ProcessHandle`) every 5 s and publishes a JMS `MapMessage`
on the node-local `AGENT` topic. Each row is scoped `APP`, grouped `runner-<streamName>`, summed
across the tree: `process.count`, `bytes.read`, `bytes.written`, `kernel.time` (ms),
`thread.count`, `set.size` (RSS bytes), `virtual.size` (bytes).

## Plugin: `filerepo`

| Metric | Group / type | How measured | Meaning |
|---|---|---|---|
| **`filerepo.files.count`** | `filerepo`, GAUGE_LONG | `repoEngine.getRepoCount()` | Files tracked in the local repo catalog |
| **`filerepo.active.transfers`** | `filerepo`, GAUGE_INT | `transferStreams.size()` | In-flight `streamfile` transfers |

## Plugin: `repo`

| Metric | Group / type | How measured | Meaning |
|---|---|---|---|
| **`repo.plugin.count`** | `repo`, GAUGE_INT | `getPluginInventory(repoDir).size()` | Plugin jars in the local repository |
| **`repo.bytes`** | `repo`, GAUGE_LONG, bytes | Σ `f.length()` over `*.jar` in the repo dir | Total on-disk size of repository jars |

## Plugin: `stunnel`

### Inventory gauges (group `stunnel`)
| Metric | Type | How measured | Meaning |
|---|---|---|---|
| **`stunnel.active.tunnels`** | GAUGE_INT | `activeServerChannels.size()` | Active SRC tunnel listeners |
| **`stunnel.active.clients`** | GAUGE_INT | `activeClientChannels.size()` | Active client channels |
| **`stunnel.active.targets`** | GAUGE_INT | `activeTargetChannels.size()` | Active target (destination) channels |

### Per-tunnel throughput (dataplane `stats` stream)
Each tunnel direction runs a `PerformanceMonitor` that accumulates transferred bytes in a
`LongAdder` and, every `performance_report_rate` ms (default 5000), publishes a JSON `TextMessage`
on the `GLOBAL` topic with `type=stats`: `bits_per_second` (the primary throughput number,
`bytesDelta/elapsed × 8`), `bytes_delta` (this interval), `total_bytes` (cumulative), plus
`stunnel_id`, `direction`, `is_healthy`, `elapsed_time`, `buffer_size`.

## Plugin: `wsapi`

The `wsapi` gauges (group `wsapi`) read process-wide atomics maintained by the Netty dataplane
handler — one wsapi plugin per JVM, so the counts are node-wide.

| Metric | Type / unit | How measured | Meaning |
|---|---|---|---|
| **`wsapi.dataplane.connections`** | GAUGE_INT | `ACTIVE_CONNECTIONS` atomic, incremented on `channelActive` | Active dataplane WebSocket connections |
| **`wsapi.dataplane.bytes`** | GAUGE_LONG, bytes | `DATAPLANE_BYTES` atomic, added per text/binary frame | Total dataplane bytes ingested from clients |
| **`wsapi.dataplane.messages`** | GAUGE_LONG | `DATAPLANE_MESSAGES` atomic, incremented per frame | Total dataplane messages ingested from clients |
| **`wsapi.dataplane.egress.pending.bytes`** | GAUGE_LONG, bytes | bytes queued to all WS consumers | Dataplane bytes queued to WS consumers, not yet written |
| **`wsapi.dataplane.egress.max.pending.bytes`** | GAUGE_LONG, bytes | largest backlog one consumer reached | How close bursts come to a consumer's budget (`wsapi_dataplane_egress_buffer_bytes`) |
| **`wsapi.dataplane.egress.paused`** | GAUGE_LONG | incremented each time a consumer's listener waits | Times a WS consumer more than its budget behind had its stream held at the broker (backpressure: writers slowed, nothing lost) |
| **`wsapi.dataplane.egress.stalled`** | GAUGE_LONG | incremented on each 1013 stall close | WS consumers closed because, held, they read nothing for `wsapi_dataplane_stall_close_ms` |
| **`wsapi.dataplane.egress.dropped`** | GAUGE_LONG | incremented per dropped frame | Frames dropped past a consumer's budget (only with `wsapi_dataplane_backpressure=false`) |
| **`wsapi.dataplane.ingress.failed`** | GAUGE_LONG | incremented per client frame not taken | Client dataplane frames the controller did not take (no session, controller not active) |
| **`wsapi.auth.failures`** / **`.servicekey.failures`** / **`.refused`** / **`.revoked.connections`** | GAUGE_LONG | `AuthRateLimiter`, `WsRouter.Gate` | Failed authentications, wrong service keys, authentications refused before they were checked (429), connections closed because their credential was revoked or changed |
| **`wsapi.inbound.inflight.bytes`** / **`wsapi.inbound.budget.closed`** | GAUGE_LONG | `InboundBudget` | Inbound message bytes held now; connections closed (1013) for too many bytes in flight |

Each forwarded binary dataplane message is stamped with `dp_bytes`, which is what feeds the
controller's per-edge `link.*.tx_mbps` throughput accounting above.

---

## Legacy / complementary telemetry

Two richer, non-Micrometer streams predate the unified model and are folded in for back-compat:

- **`sysinfo` `perf` stream + `getsysinfo`** — `PerfSysMonitor` publishes a per-component JSON
  `MapMessage` every `perftimer` ms (default 10000) on the `AGENT` topic, and the `getsysinfo` EXEC
  returns the same on demand: full per-CPU/mem/disk/filesystem/partition/NIC/OS/process records
  (e.g. per-NIC `link-speed`, `mac`, `errors-in/out`; per-disk `disk-readbytes`/`disk-writebytes`;
  sampled `cpu-user-load`/`cpu-sys-load`/`cpu-idle-load`). This is what the controller's
  `resourceinfo` path parses to build region/agent resource totals (`cpu_core_count`,
  `mem_available`, `mem_total`, `disk_available`, `disk_total`).

## Benchmarks

On-demand CPU benchmarking (SciMark2) is available via `sysinfo`'s `getbenchmark` action —
deliberately not run at startup, to keep node bring-up cheap. See the metrics-unification design doc
via [Design Docs](../reference/design-docs.md).
