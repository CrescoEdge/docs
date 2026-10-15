# stunnel — Secure TCP Tunnel

The `stunnel` plugin (bundle `io.cresco.stunnel`) builds bidirectional TCP tunnels across the fabric. It bridges a local TCP listener (SRC) to a remote target (DST) by tunneling raw bytes as JMS messages over the [data plane](../architecture/dataplane.md), using [Netty](https://netty.io/) for socket I/O. This lets a TCP service on one agent's network be reached through Cresco from another agent.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/stunnel` |
| Bundle symbolic name | `io.cresco.stunnel` |
| Java files | 9 |
| Loaded | At runtime by the controller's `StaticPluginLoader`. |
| Type | Functional plugin (`@Component` implementing `PluginService`). |
| Transport | Raw bytes as JMS `BytesMessage` over `TopicType.GLOBAL`, one consumer per tunnel (see [data path design](../design/stunnel-dataplane-design.md)). |
| Socket I/O | Netty event loops. |

## How a tunnel works

A tunnel has a **SRC** side (a local TCP listener accepting connections) and a **DST** side (a Netty client connecting to the real target). Bytes accepted on the SRC socket are wrapped as `BytesMessage`s and published on `TopicType.GLOBAL`; the DST side reassembles them and writes them to the target socket, and vice versa. Tunnel configs are persisted so tunnels auto-reconnect after a restart.

Each message carries `stunnel_id`, `client_id` and `direction` properties. A tunnel opens **one**
dataplane consumer per direction and demultiplexes to per-client handlers in Java, so opening a
session is a map insert rather than a JMS listener attach. Ordering, buffering and the bounds on
every wait are covered in [stunnel Data Path Design](../design/stunnel-dataplane-design.md).

## Key classes

| Class | Role |
|-------|------|
| `PluginExecutor` | Handles the CONFIG/EXEC actions (below). Every handler sets `status`/`status_desc` (10 ok, 9 fail, 99 unknown, 400 missing param, 500 internal). |
| `SocketController` | The core: Netty event loops, tunnel lifecycle, config persistence, health checks, and reconnection. `startSrcTunnel`/`createDstTunnel`/`createDstSession`, `connectWithRetry`, `startHealthCheck`, `checkStartUpConfig` (rescan + reconnect at startup), teardown, and an inner `ReconnectTask`. |
| `TunnelDemux` | One dataplane consumer per tunnel per direction (selector `stunnel_id` + `direction`), dispatching to per-client handlers by `client_id`. Buffers a client's payload between `expect()` and `register()` so a session's first bytes cannot be lost, and drains that backlog without reordering. |
| `SrcChannelInitializer` / `DstChannelInitializer` | Netty pipelines for accepted SRC connections / DST target connections; the handlers forward bytes both ways as `BytesMessage`s and handle `eos`/status `8` (graceful close)/`9` (failure) markers with half-close. |
| `PerformanceMonitor` | Per-direction throughput meter (Micrometer `DistributionSummary`); periodically publishes a throughput record (tagged `cresco_msg_type="stunnel_trace"`) carrying `bits_per_second`, the observed broker **hop path** (`setHops`, from the arriving `cresco_hops` property), and the src/dst region/agent endpoints. |
| `SocketControllerSM` | UMPLE-generated tunnel state machine (only its initial `pluginActive` state is read). |

## Managing tunnels from a client

The [Java client](../clients/java.md#stunnel) wraps these actions: `deploy_plugin`,
`create_tunnel`, `get_tunnel_list`, `get_tunnel_status`, `get_tunnel_config`,
`tunnel_health_check`, `get_metrics`, `set_net_tuning` and `remove_tunnel` — no
CADL pipeline needed. The [Python client](../clients/python.md) has the same
core calls on its `stunnel` submodule.

## Executor actions

CONFIG actions ([`MsgEvent`](../api/msgevent.md) type `CONFIG`):

| Action | Behaviour |
|--------|-----------|
| `configsrctunnel` | Create the SRC-side listener for a tunnel. |
| `configdsttunnel` | Create the DST-side tunnel to the target. |
| `configdstsession` | Establish a DST session for an accepted SRC connection. |
| `removesrctunnel` / `removedsttunnel` | Tear down a tunnel side. |

EXEC actions (type `EXEC`): `tunnelhealthcheck`, `listtunnels`, `gettunnelstatus`, `gettunnelconfig`.

## Health checking

`SocketController.startHealthCheck` runs per SRC tunnel on a **fixed delay** (default 5s,
`stunnel_health_check_interval_sec`) so a slow probe cannot queue an overdue one behind it. Recent
byte activity is treated as proof-of-life; otherwise it issues a `tunnelhealthcheck` RPC bounded by
`stunnel_health_probe_timeout_ms` (default 10s; a per-tunnel `watchdog_timeout` overrides it).

After `stunnel_health_failure_threshold` consecutive failures (default 3) it sends **one further
verification probe** before tearing down — teardown closes every client session on the tunnel, and
the counted failures may all have been probes sent into a stall that has since cleared. Teardown
leaves the persisted config in place, so `ReconnectTask` rebuilds the tunnel. On startup,
`checkStartUpConfig` rescans persisted configs and reconnects tunnels.

## Configuration read

Per-tunnel config keys:

| Key | Default | Meaning |
|-----|---------|---------|
| `stunnel_id` | — | Tunnel identifier. |
| `src_port` | — | Local SRC listen port. |
| `dst_host` / `dst_port` | — | Remote target host/port. |
| `dst_region` / `dst_agent` / `dst_plugin` | — | DST-side stunnel plugin address. |
| `src_region` / `src_agent` / `src_plugin` | — | SRC-side stunnel plugin address. |
| `buffer_size` | `8192` | I/O buffer size. |
| `watchdog_timeout` | — | Health-probe RPC budget in ms for this tunnel; overrides `stunnel_health_probe_timeout_ms`. |
| `performance_report_rate` | `5000` | `PerformanceMonitor` report interval (ms). |
| `debug_performance` | `false` | Verbose throughput logging. |

Plugin-level config keys (read from `getConfig`, apply to the whole stunnel plugin):

| Key | Default | Meaning |
|-----|---------|---------|
| `stunnel_beacon_ms` | `3000` | How often each SRC-configured tunnel pushes an existence/status beacon on the `stunnel_trace` stream. |
| `stunnel_socket_buffer_bytes` | `4194304` | Netty socket send/receive buffer (4 MB). |
| `stunnel_read_chunk_bytes` | `262144` | Max adaptive read chunk (256 KB), and so the max size of one dataplane payload message. |
| `stunnel_write_high_water_bytes` | `2097152` | Write-buffer high-water mark for backpressure (2 MB). |
| `stunnel_health_check_interval_sec` | `5` | Tunnel health-check period (seconds), scheduled as a fixed delay. |
| `stunnel_health_failure_threshold` | `3` | Consecutive failures before the verification probe and teardown. |
| `stunnel_health_probe_timeout_ms` | `10000` | RPC budget for a health probe; per-tunnel `watchdog_timeout` overrides it. |
| `stunnel_dst_init_timeout_ms` | `10000` | RPC budget for `configdstsession` when a client connects. |
| `stunnel_dst_init_max_concurrent` | `64` | Concurrent DST session initiations; excess connects fail fast. |
| `stunnel_demux_buffer_max` | `256` | Messages buffered per client before its handler registers; overflow fails that session. |
| `stunnel_demux_open_timeout_ms` | `15000` | Bound on opening a tunnel's dataplane consumer. |

How these interact is covered in
[stunnel Data Path Design](../design/stunnel-dataplane-design.md).

See [Configuration](../getting-started/configuration.md).

## Path tracing & existence beacons

Because tunnel bytes ride the JMS data plane, transit nodes never see them — only the brokers do. stunnel makes the full path observable and pushes it on a subscribable stream (`cresco_msg_type="stunnel_trace"`):

- **Hop tracing.** Outbound tunnel `BytesMessage`s are marked `cresco_trace="1"` (`SrcChannelInitializer`/`DstChannelInitializer`), so each broker they transit stamps its node identity onto a `cresco_hops` property (see [`CrescoTraceBroker`](../architecture/tunnel-tracing.md)). On arrival, the handler reads `cresco_hops` and feeds it to `PerformanceMonitor.setHops`, so the throughput record carries the real broker path the tunnel crossed.
- **Existence beacons.** `SocketController` publishes one status beacon per SRC-configured tunnel every `stunnel_beacon_ms` (default 3000) on the same `stunnel_trace` stream, so a subscriber knows a tunnel **exists even with zero traffic** (status ACTIVE/INACTIVE, endpoints).

See [Tunnel Path Tracing & Push Observability](../architecture/tunnel-tracing.md) for the end-to-end design (broker hop stamping, the pushed trace stream, per-link bandwidth attribution, and RTT-driven connector auto-scaling).

## Observability

Distinct from the per-tunnel self-healing check above, stunnel is wired into the fabric's central
observability:

- **Metrics** — `getmetrics` exposes `MeasurementEngine` gauges `stunnel.active.tunnels`,
  `stunnel.active.clients`, and `stunnel.active.targets`, aggregated by `getmetricinventory`.
  Per-tunnel throughput is streamed separately as a dataplane `stats` message
  (`bits_per_second`/`bytes_delta`/`total_bytes`). See
  [Metrics & Measurements › stunnel](../architecture/metrics.md#plugin-stunnel).
- **Health** — registers the `stunnel` Felix HealthCheck (tag `local`) reporting the configured-tunnel
  count, discovered by the controller's health executor. See
  [Health & State](../architecture/health.md#plugin-checks).

## See also

- [stunnel Data Path Design](../design/stunnel-dataplane-design.md) — session lifecycle, demux, ordering, failure modes.
- [Tunnel Path Tracing & Push Observability](../architecture/tunnel-tracing.md) — hop tracing, the `stunnel_trace` stream, per-link bandwidth, and connector auto-scaling.
- [Data Plane](../architecture/dataplane.md) — the JMS transport tunneled bytes travel over.
- [Metrics & Measurements](../architecture/metrics.md) — `PerformanceMonitor` throughput reporting.
- [Overview](overview.md) — the functional-plugin model.
