# Library API

The `library` module (`io.cresco.library`) is the API surface that **plugins and the controller** build
on. If you are writing a Cresco plugin, this is your SDK; if you are integrating, it's the shared vocabulary
the whole fabric speaks. For the module overview see [library](../plugins/library.md).

## PluginBuilder — the plugin's handle to the fabric

`PluginBuilder` is injected into a plugin and is its gateway to everything: identity, config, logging,
messaging, RPC, and services.

| Area | Key methods |
|------|-------------|
| Identity | `getRegion()`, `getAgent()`, `getPluginID()` |
| Config | `getConfig()` → [`Config`](#config) |
| Logging | `getLogger(name, level)` → `CLogger` |
| Messaging | `msgIn(MsgEvent)`, `msgOut(MsgEvent)`, `sendRPC(MsgEvent)` |
| Services | `getAgentService()` → `AgentService` (→ `getDataPlaneService()`), `getCrescoMeterRegistry()` |

### Message factories

`PluginBuilder` builds a correctly-addressed, correctly-scoped `MsgEvent` for you — one factory per
(tier × target). Pass a [`MsgEvent.Type`](msgevent.md) and, for the non-controller targets, the
destination identity:

| Factory | Destination |
|---------|-------------|
| `getGlobalControllerMsgEvent(type)` | the global controller |
| `getGlobalAgentMsgEvent(type, region, agent)` | a specific agent, routed via global |
| `getGlobalPluginMsgEvent(type, region, agent, plugin)` | a specific plugin, routed via global |
| `getRegionalControllerMsgEvent(type)` | this node's region controller |
| `getRegionalAgentMsgEvent(type, region, agent)` | an agent in this region |
| `getRegionalPluginMsgEvent(type, region, agent, plugin)` | a plugin in this region |
| `getAgentMsgEvent(type)` | this agent's controller |
| `getKPIMsgEvent()` | the KPI/metrics sink |

## Config

`Config` provides typed access to a node's [configuration](../getting-started/configuration.md):

```java
Config c = plugin.getConfig();
int    p = c.getIntegerParam("broker_port", 32010);
long   t = c.getLongParam("controlplane_ttl", 300000L);
boolean s = c.getBooleanParam("broker_security_enabled", true);
String  n = c.getStringParam("tenant_id", "default");
double  d = c.getDoubleParam("prefetch_rate_multiplier", 2.5);
```

Resolution is **system property → config map → default**; the default preserves shipped behavior. See the
full [Configuration Parameters reference](../reference/configuration.md).

## MsgEvent — the message

The unit of the control plane. Full structure and method reference on its own page: **[MsgEvent](msgevent.md)**.
In brief: a typed message (`EXEC`, `CONFIG`, `INFO`, `KPI`, `WATCHDOG`, …) with source/destination
`(region, agent, plugin)`, scope flags, string parameter maps, optional compressed params, and a file
list. Build one from a `PluginBuilder` factory, set params (including `action` for a plugin
[action](plugin-actions.md)), and `msgOut()` or `sendRPC()` it.

## DataPlaneService — streaming

The pub/sub streaming interface (`io.cresco.library.data.DataPlaneService`); see
[Data Plane](../architecture/dataplane.md) for concepts. Reached via
`plugin.getAgentService().getDataPlaneService()`.

**Subscribe / publish**

| Method | Purpose |
|--------|---------|
| `addMessageListener(TopicType, MessageListener, selector)` → `String id` | subscribe to a scope with an optional JMS selector |
| `addMessageListener(TopicType, MessageListener, selector, int shard)` | subscribe on a specific shard (for high-throughput streams) |
| `removeMessageListener(String id)` | unsubscribe |
| `sendMessage(TopicType, Message)` | publish to a scope |
| `sendMessage(TopicType, Message, int deliveryMode, int priority, int timeToLive)` | publish with JMS delivery options |
| `sendMessage(TopicType, Message, int deliveryMode, int priority, int timeToLive, int shard)` | publish to a specific shard |
| `sendMessage(MsgEvent.Type, TopicType, Message)` | publish with an explicit message type |
| `shardFor(String routingKey)` → `int` | deterministic shard for a stream/routing key (pair with the sharded overloads) |
| `isFaultURIActive()` → `boolean` | is this node's messaging-plane connection up (the signal behind the `dataplane` health check) |

**Message factories** (all no-arg unless noted)

`createTextMessage()`, `createBytesMessage()`, `createMapMessage()`, `createObjectMessage()`,
`createStreamMessage()`, `createMessage()`, `createMessage(InputStream)`, and
`getInputMessageStream(Message)` to read a streamed message back.

**Embedded CEP** (Complex-Event-Processing over dataplane streams; the engine runs in the controller)

| Method | Purpose |
|--------|---------|
| `createCEP(inputSchema, inputStream, outputStream, outputAttributes, query)` → `String cepId` | register a Siddhi streaming query |
| `inputCEP(streamName, jsonPayload)` | feed one event into a CEP input stream |
| `removeCEP(String cepId)` → `boolean` | tear a query down |

`TopicType` is `AGENT` (node-local), `REGION`, or `GLOBAL` (mesh-reachable via wsapi).

**Paced listeners (backpressure).** A listener that implements `io.cresco.library.data.PacedListener` (a marker
sub-interface of `MessageListener`) may block in `onMessage` to slow its stream: what it has not taken stays at the
broker, whose producer flow control slows the stream's writers. The data plane gives each paced listener a JMS session
of its own (consumer prefetch `dataplane_paced_prefetch`, 16), so a wait holds back only its own messages, in order. It
must not wait forever: give up on a consumer that reads nothing, return, and remove the listener. wsapi's WebSocket
consumers are paced listeners. See [Data Plane](../architecture/dataplane.md#backpressure-nothing-lost-order-kept).

## MeasurementEngine — metrics

Register and update metrics; see [Metrics & Measurements](../architecture/metrics.md).

| Method | Purpose |
|--------|---------|
| `setTimer` / `updateTimer` | latency timers (nanoTime deltas) |
| `setGauge` / `updateIntGauge` / `updateLongGauge` / `updateDoubleGauge` | point-in-time values |
| `setDistributionSummary` / `updateDistributionSummary` | value distributions |
| `getAllMetrics()` | `Map<group, List<Map<String,String>>>` — the wire schema behind `getmetrics` |

## Security primitives

Under `io.cresco.library.security` (see [Security & Identity](../architecture/security.md) and
[Multi-Tenancy](../architecture/tenancy.md)):

| Type | Purpose |
|------|---------|
| `CrescoIdentity` | identity ⇄ cert DN (`tenant`/`region`/`agent`/`uid`) |
| `Caller` | the verified caller of a message (kind, principal, tenant, roles), stamped by infrastructure, never by a sender |
| `ActionAuthorizer` | decides whether a message's verified caller may run its action (`@CrescoAction(roles=...)`); refusal answers `403` |
| `SecurityProfile` | `PRODUCTION`, `STANDARD` or `INSECURE` (debug mode); plugins read the node's with `PluginBuilder.getSecurityProfile()` |
| `TenantNamespace` | qualify/parse tenant-namespaced destinations (`T.<tenant>.<raw>`) |
| `JarSignatures` | verify a plugin jar's signature against a trust store (`verify`; `verifyCms` through the installed providers only, used in production) |
| `OwnerOnlyFile` | read a secret only from an owner-only file (the `<name>_file` secret sources) |
| `MessageSigner` | sign/verify a byte payload with a private key / a certificate's public key |
| `CertTrust` | PKIX chain-verify a leaf to a trusted region CA |

## Capability annotations

Under `io.cresco.library.capability`. Actions are declared with annotations that make a plugin
self-describing and aggregatable into the fabric-wide [capability inventory](plugin-actions.md):

| Annotation | On | Purpose |
|------------|----|---------|
| `@CrescoCapabilities` | the Executor class | declares the plugin's `namespace`, `target`, routing params, and summary |
| `@CrescoActions` | the Executor class | container holding the `@CrescoAction` list |
| `@CrescoAction` | (inside `@CrescoActions`) | one callable action — `name`, `type`, `summary`, `why` |
| `@CrescoParam` | inside `@CrescoAction` | one action parameter — name, required, type, description |
| `@CrescoReturn` | inside `@CrescoAction` | one reply field — name, type, description, `compressed` |

At runtime `CapabilityResponder.respond(incoming, this)` answers the `getcapabilities` action by
scanning these annotations (`CapabilityScanner`) into a `CapabilityDocument`; the controller's
`getcapabilityinventory` aggregates every plugin's document into one tool catalog. Versions come from
the OSGi manifest — never hard-coded.

## Exported packages

The `library` bundle exports the whole `io.cresco.library.*` API — `agent` (`AgentService`),
`plugin` (`PluginBuilder`, `Config`, `PluginService`, `Executor`), `messaging` (`MsgEvent`),
`data` (`DataPlaneService`, `TopicType`), `metrics` (`MeasurementEngine`, `CMetric`), `capability`,
`security`, `core`, `app`, and `utilities` (`CLogger`) — plus the shared embedded runtime it
re-exports for plugins (Guava, jakarta.jms, Micrometer, Siddhi). See [library](../plugins/library.md).
