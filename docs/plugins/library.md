# library — The Plugin & Controller API

The `library` module (bundle `io.cresco.library`) is the shared OSGi **uber-bundle** that every plugin and the [controller](controller.md) compiles against. It defines the `io.cresco.library.*` API surface and embeds the shared third-party runtime (Jakarta JMS, Siddhi CEP, Micrometer, Quartz, Guava, ANTLR, LMAX Disruptor, …).

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/library` |
| Bundle symbolic name | `io.cresco.library` |
| Java files | 45 |
| Loaded | At boot, before `core` and `controller`. |
| Declarative Services | None (this bundle is API + embedded deps only). |

The bundle `Export-Package` re-exports `io.cresco.library.*` plus the embedded runtime (`io.siddhi.*`, `io.micrometer.*`, `jakarta.jms.*`, `org.quartz.*`, `com.google.common.*`, `com.codahale.metrics.*`, `org.antlr.v4.runtime.*`, `com.lmax.disruptor.*`, `org.atteo.classindex.*`, `org.LatencyUtils`). `Import-Package` hard-requires only `org.osgi.framework(.*)` and `org.slf4j(.*)`; everything else is `resolution:=optional`. `Embed-Dependency` embeds all dependencies except OSGi core and slf4j-api.

!!! note
    This page is a map of the API. For the fuller reference see [Library API](../api/library-api.md) and, for the wire message, [MsgEvent](../api/msgevent.md).

## Plugin authoring surface (`io.cresco.library.plugin`)

### PluginBuilder

`PluginBuilder` is the object every plugin holds — the primary public API. It acquires the controller's `AgentService` via an OSGi `ServiceTracker` (re-acquired transparently on controller restart) or a directly-injected instance (the agent/controller path).

| Concern | Methods |
|---------|---------|
| Identity | `getAgent()`, `getRegion()`, `getPluginID()` |
| Inbound/outbound | `msgIn(MsgEvent)`, `msgOut(MsgEvent)`, `receiveRPC(callId, MsgEvent)` |
| Synchronous RPC | `sendRPC(MsgEvent)` / `sendRPC(MsgEvent, timeoutMs)` (default ~30s) |
| Handler registration | `setExecutor(Executor)` — **required**; without it inbound messages are dropped. `isActive()` / `setIsActive(bool)` |
| Config | `setConfig(map)` / `getConfig(): Config` |
| Logging | `getLogger(issuingClass, Level): CLogger`, `setLogLevel(logId, Level)` |
| Metrics | `getCrescoMeterRegistry(): CrescoMeterRegistry` |
| Data dir | `getPluginDataDirectory()` → `<agentData>/plugin-data/<pluginID>` |
| Bundle context | `getBundleContext()` — to register OSGi services such as Felix Health Checks (may be null on the injected path) |

`PluginBuilder` also exposes **message factories** that build a typed [`MsgEvent`](../api/msgevent.md) addressed to a tier — e.g. `getGlobalControllerMsgEvent(Type)`, `getGlobalPluginMsgEvent(Type, dstRegion, dstAgent, dstPlugin)`, `getRegionalAgentMsgEvent(Type, dstAgent)`, `getAgentMsgEvent(Type)`, `getPluginMsgEvent(Type, dstPlugin)`, `getKPIMsgEvent()` — and JAR helpers (`getPluginName`, `getPluginVersion`, `getMD5`, `getPluginInventory`).

### Executor and PluginService

| Interface | Role |
|-----------|------|
| `Executor` | Implemented by a plugin and registered via `setExecutor`; dispatched by message type: `executeCONFIG`/`executeDISCOVER`/`executeERROR`/`executeINFO`/`executeEXEC`/`executeWATCHDOG`/`executeKPI(MsgEvent): MsgEvent`. Each returns a reply or `null`. |
| `PluginService` | Lifecycle interface the agent drives: `inMsg(MsgEvent): boolean`, `isStarted()`/`isStopped()`/`isActive(): boolean`, `setIsActive(bool)`. |

### Config

`io.cresco.library.plugin.Config` — a thread-safe view over a `Map`, with system properties and `CRESCO_`-prefixed environment variables overriding map entries. Read via `PluginBuilder.getConfig()`. Typed getters: `getBooleanParam`/`getDoubleParam`/`getIntegerParam`/`getLongParam`/`getStringParam(key[, ifNull])`, plus `getConfigMap()` and `getConfigAsJSON()`. See [Configuration](../getting-started/configuration.md).

## Messaging (`io.cresco.library.messaging`)

- **`MsgEvent`** — the universal Cresco message: a typed header (src/dst region/agent/plugin + regional/global flags), a `params` string map, and an internal `msgparams` control map. Types: `CONFIG, DISCOVER, ERROR, EXEC, GC, INFO, KPI, LOG, WATCHDOG`. Supports compressed params, binary data params, and file-transfer attachments. Detailed on the [MsgEvent](../api/msgevent.md) page.
- **`RPC`** — the blocking request/reply helper owned by `PluginBuilder` (used through `sendRPC`): sends with a generated `callId`, polls the reply map (default 30s), and returns the reply or `null`.

The messaging fabric that actually routes these is described in [Messaging & Routing](../architecture/messaging.md).

## Agent-facing services (`io.cresco.library.agent`)

| Interface | Role |
|-----------|------|
| `AgentService` | Implemented by the controller, injected into every `PluginBuilder`: `getAgentState()`, `getDataPlaneService()`, `getCLogger(...)`, `msgOut(id, MsgEvent)`, `setLogLevel(...)`, `getAgentDataDirectory()`. |
| `AgentState` | Serializable snapshot delegating to a `ControllerState`: region/agent identities at each tier, `isActive()`, `getControllerState(): ControllerMode`. |
| `ControllerState` | Interface the controller implements: `isActive()`, `getControllerState()`, `isRegionalController()`/`isGlobalController()`, tier identities, and the global/regional/agent paths. |
| `ControllerMode` | Enum of lifecycle states — `PRE_INIT`; `STANDALONE(_INIT/_FAILED/_SHUTDOWN)`; `AGENT(_INIT/_FAILED/_SHUTDOWN)`; `REGION(_INIT/_FAILED)`, `REGION_GLOBAL(_INIT/_FAILED)`, `REGION_SHUTDOWN`; `GLOBAL(_INIT/_FAILED/_SHUTDOWN)`. See [Health & State](../architecture/health.md). |

## Data plane (`io.cresco.library.data`)

`DataPlaneService` is implemented by the controller and obtained via `AgentService.getDataPlaneService()`. It provides topic pub/sub (`TopicType` = `AGENT`/`REGION`/`GLOBAL`), listener registration with JMS selectors, JMS message factories, Siddhi CEP creation (`createCEP`/`inputCEP`/`removeCEP`), and chunked file transfer (`FileObject`, `splitFile`, `mergeFiles`, `downloadRemoteFile`). See [Data Plane](../architecture/dataplane.md).

## Metrics (`io.cresco.library.metrics`)

`MeasurementEngine` is the Micrometer façade a plugin instantiates with its `PluginBuilder` to register and update metrics; it wraps the plugin's `CrescoMeterRegistry` (a Dropwizard registry exposed as JMX MBeans via `CrescoReporter`). Metric classes: `GAUGE_INT/LONG/DOUBLE/AUTO`, `TIMER`, `DISTRIBUTION_SUMMARY`, `COUNTER`. See [Metrics & Measurements](../architecture/metrics.md).

## Security & identity

Identity in Cresco is PKI-based. In the standard profile the controller's `CertificateManager` (see [controller](controller.md)) makes each node's X.509 chain (or enrolls a leaf from its region CA) and keeps PKCS12 key and trust stores (BCFKS in FIPS mode); in the production profile it loads the identity the operator's PKI issued. Discovery establishes trust with an authenticated exchange (discovery v2: HMAC-SHA-384 keyed by the discovery secret, or signatures under the operator's PKI in production) before any certificate is recorded. Every message is authenticated end to end ([Message authentication](../security/message-auth.md)); the library's own security types are listed in the [Library API](../api/library-api.md#security-primitives). External identity — a client's region/agent/plugin — is read from the TLS peer certificate DN by [wsapi](wsapi.md) and the clients. The end-to-end model, including tenant namespaces and multi-tenancy, is documented in [Security & Identity](../architecture/security.md) and [Multi-Tenancy](../architecture/tenancy.md).

## Application description types (`io.cresco.library.app`)

Plain graph POJOs consumed by the controller's global scheduler and Derby DB: `gPayload` (a pipeline), `gNode` (a node), `gEdge` (an edge), and `pNode` (a plugin descriptor: name / jarfile / md5 / version / repoServers).

## Utilities and core

- `CLogger` (`io.cresco.library.utilities`) — the logging interface handed to plugins. Levels: `None(-1), Error(0), Warn(1), Info(2), Debug(4), Trace(8)`; `error`/`warn`/`info`/`debug`/`trace(msg[, params][, Throwable])`.
- `CoreState` (`io.cresco.library.core`) — the coarse lifecycle interface implemented by [core](core.md): `updateController(jarPath)`, `stopController()`, `restartController()`, `restartFramework()`, `killJVM()`.

## See also

- [Library API](../api/library-api.md) · [MsgEvent](../api/msgevent.md)
- [controller](controller.md) — the implementer of `AgentService`, `DataPlaneService`, and `ControllerState`.
- [Configuration](../getting-started/configuration.md)
