# Glossary

Core Cresco terms, in one place.

| Term | Meaning |
|------|---------|
| **Agent** | A leaf node — a JVM running the Cresco host + plugins, attached as a client to a regional controller's broker. Also, the identity of any node (`region_agent`). |
| **Region / Regional controller** | A controller that runs its own embedded ActiveMQ broker; the agents in its region attach to it. Federates up to a global. |
| **Global / Global controller** | The top of the hierarchy; federates regions (and other globals) together. Serves clients via [wsapi](../plugins/wsapi.md). |
| **Controller mode** | A node's role: `STANDALONE`, `AGENT`, `REGION`, `REGION_GLOBAL`, or `GLOBAL`. |
| **Controller** | The [fabric-brain module](../plugins/controller.md) present on every node (messaging, discovery, security, health, db, data plane). |
| **Plugin** | An OSGi bundle providing functionality and exposing [actions](../api/plugin-actions.md) (e.g. `sysinfo`, `wsapi`, `stunnel`, `repo`). |
| **MsgEvent** | The unit of the [control plane](../architecture/messaging.md) — a typed, addressed, routed message. See [MsgEvent](../api/msgevent.md). |
| **Action** | A callable operation a plugin exposes, declared with `@CrescoAction`; invoked by sending a MsgEvent. |
| **Capability inventory** | The fabric-wide catalog of all actions (`getcapabilityinventory`). |
| **Data plane** | The [pub/sub streaming channel](../architecture/dataplane.md) over ActiveMQ topics (`agent.event` / `region.event` / `global.event`). |
| **Control plane** | The addressed [MsgEvent](../architecture/messaging.md) channel over ActiveMQ queues. |
| **MsgRouter** | The controller component that makes the deterministic per-hop routing decision. |
| **Federation / Network of brokers** | Broker-to-broker ActiveMQ bridges that connect regions to globals (and globals to each other). |
| **Bridge** | A duplex, demand-forwarding ActiveMQ network connector between two brokers. |
| **QoS tier** | The priority class of a message: `LIVENESS` > `CONTROL` > `TELEMETRY` > `BULK`. |
| **Discovery** | Finding a parent controller (UDP broadcast or static TCP) and establishing trust with it over an authenticated exchange: keyed by a tier [join secret](../architecture/discovery.md) (standard profile) or signed under the operator's PKI (production). |
| **Join secret** | The pre-shared `discovery_secret_{agent,region,global}` a node must hold to join a tier (standard profile; refused in production). |
| **Service key** | `cresco_service_key` — wsapi's legacy API credential (an `admin` of the agent's tenant); refused in production. |
| **Message authentication** | Every MsgEvent authenticated by its origin and checked by its recipient before dispatch: a signature, a session-key MAC, or first hand on a direct link. See [Message authentication](../security/message-auth.md). |
| **Recipient binding** | The signed recipient of a message (`cresco_sig_to`): a node refuses a message that does not name it. |
| **Direct link** | A message between a node and its own parent or child: no relay, so it travels unsigned and the receiving broker attests the sender (`cresco_direct`). |
| **Session key** | An end-to-end key two nodes agree with ECDH P-384 to MAC (HMAC-SHA-384) their messages instead of signing each. |
| **Trust bundle** | The signed, versioned list of region and global controller keys the global sends down on its pongs. |
| **Backpressure** | On the data plane, a subscriber that falls behind slows its topic's writers (producer flow control) instead of losing messages. See [Data Plane](../architecture/dataplane.md#backpressure-nothing-lost-order-kept). |
| **Paced listener** | A dataplane listener (`PacedListener`) that may wait in `onMessage` to slow its stream; it gets a JMS session of its own. |
| **Security profile** | `standard` (default) or `production` (operator PKI, FIPS 140-3 approved-only, no passwords). See [Security profiles](../security/profiles.md). |
| **CrescoIdentity** | A node's identity `tenant/region/agent(/uid)`, bound into its X.509 leaf DN (`CN=agent, OU=region, O=tenant`). |
| **Regional CA** | A region/global controller acting as a certificate authority, signing joiners' leaves — distributed trust (O(regions)); standard profile only. |
| **Tenant** | An isolation domain (`O=` in the cert DN). Multiple tenants can share one fabric; see [Multi-Tenancy](../architecture/tenancy.md). |
| **Tenant namespacing** | Qualifying destinations `T.<tenant>.<raw>` so isolation is enforceable by a local prefix check at every broker. |
| **Role** | An authorization role: `SUPERUSER` (all tenants) or `TENANT` (own tenant only). |
| **Health check** | A Felix `HealthCheck` contributing a pass/fail verdict; run centrally by `CrescoHealthExecutor`. |
| **MeasurementEngine** | The per-bundle Micrometer registry; metrics are aggregated mesh-wide via `getmetricinventory`. |
| **LinkMetrics** | Per-edge network quality (RTT/jitter/throughput/backlog) in the `netlink` metric group. |
| **Cost-aware routing** | Dynamic path selection over a pushed link-state view with per-hop cost, choosing the cheapest path across the federated mesh. Gated by `net_cost_routing`. See [Dynamic Cost-Aware Routing](../architecture/dynamic-routing.md). |
| **RouteView / LSA** | The pushed link-state advertisement (`route_lsa`) each node shares so peers build a topology graph and compute paths — pushed as it changes, not polled. |
| **Source route** | A pre-computed hop sequence stamped on a message (`srcroute`) so it follows a chosen path instead of per-hop re-selection. Gated by `net_source_routing`. |
| **Coordinator** | A controller elected to coordinate a scope. Multiple globals can coexist and regions can boot/run without one (`global_optional`). See [Coordinator Decentralization](../architecture/coordinator-decentralization.md). |
| **Quorum / epoch** | Coordinator elections use a majority quorum (`f+1` of `2f+1`) with epoch fencing, so a stale coordinator cannot act. |
| **Failure detector** | φ-accrual suspicion + SWIM indirect probing that decides when a peer is dead (`failure_phi_suspect`/`failure_phi_dead`, `failure_swim_k`). |
| **Tunnel tracing** | End-to-end [stunnel](../plugins/stunnel.md) path tracing: brokers stamp `cresco_hops` on marked bytes, streamed live (throughput + path) on the subscribable `stunnel_trace` channel. See [Tunnel Path Tracing](../architecture/tunnel-tracing.md). |
| **wsapi** | The [WebSocket API plugin](../plugins/wsapi.md) that bridges external [clients](../clients/overview.md) to the fabric (`wss://host:8282`). |
| **clientlib / pycrescolib / cppcrescolib** | The external [Java](../clients/java.md), [Python](../clients/python.md), and [C++ / Arduino (ESP32)](../clients/cpp.md) client SDKs. |
| **CADL / pipeline** | A described application (a graph of plugins/edges) submitted to and orchestrated by the global controller. |
