# controller — The Fabric Brain

The `controller` module (bundle `io.cresco.controller`) is the heart of Cresco. It owns lifecycle, messaging, discovery, persistence, the three role tiers, application scheduling, health, and measurement. It is the largest module in the codebase and implements the `AgentService`, `DataPlaneService`, and `ControllerState` interfaces that every plugin depends on.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/controller` |
| Bundle symbolic name | `io.cresco.controller` |
| Java files | 110 |
| Loaded | At boot, after `library` and `core`. |
| Bootstrap component | `AgentServiceImpl` (OSGi `@Component`, implements `AgentService`) |
| Central holder | `ControllerEngine` |
| State authority | Apache MINA finite-state machine (`ControllerSMHandler`) |

`AgentServiceImpl` activates on the OSGi component runtime, builds the `PluginBuilder`, the Derby `DBEngine`/`DBInterfaceImpl`, the `ControllerStateImp`, the `PluginAdmin`, and then creates and starts the `ControllerEngine` on a thread. The engine wires the shared subsystems and starts the MINA state machine.

## Lifecycle: the state machine

Lifecycle **state** is authoritative in an Apache MINA finite-state machine. `ControllerSM` is the fired-event proxy interface (`start`, `stop`, `globalControllerLost`, `regionalControllerLost`); `ControllerSMHandler` holds the `@State`/`@Transition` logic and a `stateUpdateTask` watchdog.

The controller runs in one of the `ControllerMode` states (defined in [library](library.md)):

| Mode | Meaning |
|------|---------|
| `STANDALONE` | Self-contained; no parent. |
| `AGENT` | A leaf agent attached to a region's broker (`is_agent=true` + `regional_controller_host`). |
| `REGION` | A regional controller over a set of agents. |
| `REGION_GLOBAL` | A regional controller that has federated up to a global. |
| `GLOBAL` | The root; also acts as its own regional controller + agent, so one global is a complete fabric (`is_global=true`). |

Each mode has `_INIT`/`_FAILED`/`_SHUTDOWN` variants. `stateInit()` is the driver: it switches on the configured mode (0 dynamic, 1 standalone, 2 agent, 6 agent+host, 8 region, 24 region+global-host, 32 global), loops discovery/init until the target mode is reached, then starts the measurement/perf monitors when `enable_controllermon` is set. A lost parent link fires `globalControllerLost`/`regionalControllerLost`, which re-drives `stateInit()`.

The full state/health interplay is documented in [Health & State](../architecture/health.md).

## Subsystems

### Communication — the ActiveMQ messaging fabric

The regional controller runs an embedded ActiveMQ broker; agents attach to it; regions federate to the global over a network-of-brokers bridge. Key classes:

| Class | Role |
|-------|------|
| `ActiveBroker` | Embedded ActiveMQ `SslBrokerService` (NIO+SSL, KahaDB) with tuned policies; the regional controller's local broker (default port `32010`). Builds duplex `static:`/failover network connectors to peer brokers. |
| `ActiveClient` | JMS connection/session factory manager; owns the single `AgentConsumer`/`AgentProducer`. `isFaultURIActive()` is the widely-consumed fabric-liveness signal. |
| `AgentProducer` / `ActiveProducerWorker` | Outbound send orchestration — isolated control-plane sender, bounded bulk-file pool, per-destination telemetry workers. |
| `AgentConsumer` | Inbound JMS queue listener; checks each message's authentication, dispatches text `MsgEvent`s and reassembles chunked byte-part file transfers (verified by SHA-256 content hash). |
| `MsgQoS` / `ControlPlaneSender` | Control-plane tiering (`LIVENESS > CONTROL > TELEMETRY > BULK`) so telemetry/bulk floods cannot starve agent-liveness traffic. |
| `MsgRouter` | Regional/global router: computes a route code from src/dst + role and forwards local or remote; times each transaction via the `MeasurementEngine`. |
| `ActiveBrokerManager` / `BrokeredAgent` / `BrokerMonitor` | Consume discovered candidate brokers and establish/track peer bridges; a lost global path fires the region's fast global-loss detector. |
| `CertificateManager` | Per-node PKI: makes (standard profile) or loads (production, operator-issued) the node's X.509 identity, keeps its PKCS12 key/trust stores (BCFKS in FIPS mode), the region CA, revocations and reloads. |
| `MsgSigning` / `MsgSessions` / `MsgBatchSigner` / `ReplayMemory` | End-to-end [message authentication](../security/message-auth.md): recipient-bound signatures, direct links, batch signatures, session keys, the replay memory. |
| `DataplaneBridgeInbound` | Broker plugin: a bridged topic message waits in line while its topic is full (dataplane [backpressure](../architecture/dataplane.md#backpressure-nothing-lost-order-kept)), at most `broker_bridge_flow_control_wait_ms` with nothing taken. |

See [Messaging & Routing](../architecture/messaging.md) and [Security & Identity](../architecture/security.md).

### netdiscovery — peer/broker discovery

Finds other agents via UDP broadcast/multicast (IPv4 and IPv6), Netty TCP listen/respond, and static point-to-point TCP for cross-region/NAT peers. Locator messages carry a validator keyed from the discovery secret (`DiscoveryCrypto`, AES-256-GCM); trust is established only by the authenticated discovery v2 exchange (`DiscoveryAuth`: HMAC-SHA-384 keyed by the discovery secret, or signatures under the operator's PKI in production). Default discovery port `32005`. Detailed in [Discovery](../architecture/discovery.md).

### db — persistence (embedded Derby)

State is persisted in an embedded Apache Derby database (JDBC + Commons DBCP2 pooling). "Nodes" and "edges" are relational tables: `rnode`/`anode`/`pnode` (region/agent/plugin), `agentof`/`pluginof` (edges), plus resource/pipeline/tenant/state tables. `DBEngine` is the JDBC DAO; `DBInterfaceImpl` delegates to it and maps status codes to `NodeStatusType`; `DBManager` polls a queue of regional-DB exports. Node status is an `int status_code` (10 = live).

### The role tiers

Message dispatch is layered per role:

| Tier | Executor | Responsibilities |
|------|----------|------------------|
| Agent | `AgentExecutor` + `PluginAdmin` | Plugin lifecycle (add/remove/list/status/upload), controller update/stop/restart, log/file fetch, discovery echo. |
| Region | `RegionalExecutor` + `RegionHealthWatcher` | Agent enable/disable, ping/pong, watchdog `nodeUpdate`, KPI forwarding; maintains peer connections and ages agents/plugins STALE→LOST. |
| Global | `GlobalExecutor` + `GlobalHealthWatcher` + `globalscheduler` | Authoritative registry/query surface, application (pipeline) scheduling (`AppScheduler`/`ResourceScheduler`), KPI sink; ages global-wide edges. |

`PluginAdmin` is the OSGi plugin lifecycle manager (resolve JAR → install → start → `PluginNode` → poll for `PluginService`), and `getPluginStatusCodes()` feeds the health subsystem. `AgentHealthWatcher` actively pings the parent and stamps the last PONG.

### health — Felix Health Checks

**Health** is a separate Felix Health Check subsystem, distinct from state. `CrescoHealthExecutor` schedules `HealthCheck` services and debounces their verdicts (grace/sticky). `LocalHealthChecks` register the broker/dataplane/DB/disk/memory/plugin **and `cep`** checks (the `cep` check monitors the embedded in-process Siddhi engine, whose active-query count is also exported as the `cep.queries.active` metric); `LinkHealthChecks` verdict the parent link; `MeshHealthChecks` roll up children's health carried on the ping/pong. Every deployed plugin additionally registers its own `local`-tagged check (executor/filerepo/repo/sysinfo/wsapi/stunnel), discovered by the same executor. The whole set is queryable over the `gethealthinventory` action — the parallel of `getmetricinventory`. Only a sustained `link:parent` → `CRITICAL` drives a state-machine transition, via the one-directional `HealthMinaBridge`. See [Health & State](../architecture/health.md).

### netmetrics & measurement

`PerfControllerMonitor` registers Micrometer JVM/system meters into the shared `MeasurementEngine` and serves on-demand resource/KPI queries (RPC `getsysinfo` to [sysinfo](sysinfo.md)). `PerfMonitorNet` pushes network-discovery topology as a compressed KPI `MsgEvent`. See [Metrics & Measurements](../architecture/metrics.md).

The `netmetrics` package also hosts [cost-aware routing](../architecture/dynamic-routing.md): `LinkMetrics`/`LinkMetricsRegistry` (per-edge smoothed RTT + composite cost), `AutoTuner` (the control loop that scales bridge connectors and drives advertising), `RouteAdvertiser`/`RouteView` (link-state pushed over the data plane and assembled into a mesh-wide graph), `RouteComputer` (Dijkstra path + waypoint stack), `PathTable` (per-peer direct-vs-via-global choice with hysteresis), and `NetworkStateJson` (live topology for the dashboard). `RegionHealthWatcher` runs the probe/infer/verify loop and `MsgRouter` enforces the chosen path and relays transit traffic.

### data — the data plane

`DataPlaneServiceImpl` is the concrete `DataPlaneService`: agent/region/global JMS topics, send lanes and paced listeners (backpressure), stream sequence numbers (`cresco_seq`/`cresco_gap`), QoS classification, Siddhi CEP creation, and chunked file transfer (5 MB parts, verified by SHA-256 content hash). `CEPEngine`/`CEPInstance` bridge a data-plane input listener to a Siddhi runtime and republish results. See [Data Plane](../architecture/dataplane.md).

## Key configuration

The controller reads a large configuration surface (see [Configuration](../getting-started/configuration.md)). Notable keys:

| Key | Default | Meaning |
|-----|---------|---------|
| `is_global` / `is_region` / `is_agent` | — | Role selection. |
| `regionname` / `agentname` | — | This node's identity. |
| `global_controller_host` / `global_controller_port` | `32005` | Parent global's discovery host/port. |
| `regional_controller_host` | — | Parent region's host (agents). |
| `discovery_port` | `32010` | Broker port. |
| `discovery_port_remote` | `32010` | Peer broker port for the network bridge. |
| `discovery_secret_{agent,region,global}` | — | Discovery shared secrets (standard profile; refused in production). |
| `watchdog_interval` | `15000` | Watchdog config-update period (ms). |
| `enable_{udp,tcp}_discovery` | — | Discovery transports. |
| `enable_controllermon` | — | Start measurement/perf monitors. |
| `db_name` / `db_driver` / `db_jdbc` | — | Derby DB connection. |

## See also

- [Overview](overview.md) — controller vs. functional plugins.
- [library](library.md) — the API the controller implements and plugins consume.
- Architecture deep-dives: [Messaging & Routing](../architecture/messaging.md) · [Data Plane](../architecture/dataplane.md) · [Discovery](../architecture/discovery.md) · [Security & Identity](../architecture/security.md) · [Health & State](../architecture/health.md) · [Metrics & Measurements](../architecture/metrics.md)
