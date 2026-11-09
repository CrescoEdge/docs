# Messaging & Routing

The control plane moves [MsgEvent](../api/msgevent.md) objects between agents, regions, and globals over a
federated network of ActiveMQ brokers. This page covers addressing, routing, the broker fabric,
federation, and quality-of-service.

## Addressing

Every node has an identity `region_agent`, and every plugin `region_agent_plugin`. A MsgEvent carries a
source and destination `(region, agent, plugin)` plus two scope flags, `isRegional` and `isGlobal`. The
per-node inbox is a JMS **queue** named `<region>_<agent>` (or `T.<tenant>.<region>_<agent>` under
[tenant namespacing](tenancy.md)).

## Routing

`MsgRouter.route()` makes a **deterministic, local** forwarding decision. It computes a 16-bit route path
from the message's scope flags and destination against the local controller's state (am I the region? the
global? is the destination local?), then dispatches to one of a fixed set of forwarders:

| Forwarder | Meaning |
|-----------|---------|
| `forwardToLocalAgent` / `forwardToLocalPlugin` | deliver in-process |
| `forwardToLocalRegionalController` | up to this region's controller |
| `forwardToRemoteRegionalController` / `forwardToRemoteRegion` | across a broker to another region |
| `forwardToLocalGlobal` / `forwardToRemoteGlobal` | up to / across to a global |

Loop safety is provided by a **BrokerPath** trail on each message plus a TTL cap.

By default the message follows the tree (agent → region → global → region → agent). When
[**cost-aware routing**](dynamic-routing.md) is enabled, `MsgRouter` additionally measures the real latency
of candidate paths, learns a mesh-wide link-state graph pushed over the data plane, computes the
lowest-latency path (Dijkstra), and **injects a source-route waypoint stack at the origin** so a flow can be
steered onto a faster multi-hop path (e.g. `R1 → G → R2` around a slow direct link) instead of ActiveMQ's
arbitrary default. Transit regions then **relay** the flow toward its destination. See
[Dynamic Cost-Aware Routing](dynamic-routing.md) for the full mechanism.

!!! note "Relays preserve, they do not re-address"
    A message keeps its `(dst_region, dst_agent, dst_plugin)` end-to-end; each relay recomputes only the
    *next hop* from `getForwardDst()`. Under [tenant namespacing](tenancy.md) the origin tenant is stamped
    once and preserved, so every relay qualifies the destination queue identically.

## Authentication end to end

Every broker stamps the verified caller of each hop (`cresco_caller`). Beyond one hop, the origin's word is carried by
the message itself: every MsgEvent a node originates is authenticated by its origin and checked by its recipient before
dispatch ([Message authentication](../security/message-auth.md)):

- between a node and its own parent or child (no relay) it travels unsigned, attested first hand by the receiving
  broker (`cresco_direct`);
- otherwise it carries a signature (RSA-3072/SHA-384, batched under load) or, between two nodes that hold a session key,
  an HMAC-SHA-384.

The signature or MAC covers the **recipient** the origin's router chose (`cresco_sig_to`): a relay can carry a message
on, but a copy re-sent to any other node is refused there. Relays do not sign what they carry, and a message a node only
carries on is checked wherever it is finally dispatched.

## The broker fabric

Region and global controllers run an embedded ActiveMQ (`SslBrokerService`) via `ActiveBroker`. Agents
connect as clients via `ActiveClient` using a `failover:(nio+ssl://host:port)` URL; the in-JVM controller
connects over `vm://localhost`. Producers and consumers are managed by `AgentProducer` / `AgentConsumer`.

Notable broker configuration (all tunable — see the
[Configuration reference](../reference/configuration.md)):

- **KahaDB** persistence with journal disk-syncs off by default (persistence is used as a flow-control
  mechanism, not for crash durability).
- Prioritized messages on, per-destination memory limits, large socket buffers. Producer flow control is **off on
  queues** (the control plane: a liveness ping must never wait behind bulk) and **on for topics** (the
  [data plane](dataplane.md#backpressure-nothing-lost-order-kept): a subscriber that falls behind slows the topic's
  writers instead of losing messages; `activemq_topic_producer_flow_control`, `dataplane_backpressure`).
- **Network bridges** (`AddNetworkConnector` / `buildConnector`) are duplex, demand-forwarding, with a
  configurable `networkTTL`. Each bridge group is split by destination type (`broker_control_bridge_split`): connector 0
  forwards queues only (the MsgEvent control fabric, prefetch `broker_bridge_prefetch`, 100), connectors 1..N topics only
  (the data plane, prefetch `broker_bridge_prefetch_data`, 64, sending synchronously so the prefetch is the bridge's
  credit window). Parallel connectors per peer can be added/removed at runtime (`addBridgeConnections`) for throughput.

## Federation (network of brokers)

Regions federate to globals (and globals to each other) via ActiveMQ **network bridges**. A bridge only
forwards a message when the far side has advertised **demand** (a consumer subscription); loop prevention
uses the message's BrokerPath. ActiveMQ resolves *reachability* across the mesh but does **not** select
paths by latency/throughput — Cresco owns any smarter routing at the `MsgRouter` layer.

## Quality of service

To guarantee that a flood of bulk data can never starve agent-liveness traffic, outbound MsgEvents are
classified by `MsgQoS` into four tiers:

| Tier | JMS priority | Delivery | Traffic |
|------|-------------|----------|---------|
| **LIVENESS** | 9 | persistent | ping/pong, watchdog |
| **CONTROL** | 7 | persistent | config, commands |
| **TELEMETRY** | 4 | non-persistent | info/log/KPI (evictable under pressure) |
| **BULK** | 1 | persistent | file/binary transfers |

Liveness + control ride a **dedicated, isolated producer/session** (`ControlPlaneSender`), separate from
the per-destination telemetry workers and the bulk executor, so control traffic is never serialized behind
or evicted by a flood. See the broker-performance design doc via [Design Docs](../reference/design-docs.md).

## RPC

Request/response is built on MsgEvent: a sender sets a reply destination and blocks for the correlated
reply (`PluginBuilder.sendRPC(msg, timeoutMs)`). The [client libraries](../clients/overview.md) expose this
as ordinary method calls.

The wait is **event-driven**: each outstanding call registers a `CompletableFuture` keyed by its call id and
the reply completes it directly (the no-timeout overload defaults to 30 s). An earlier implementation polled
in 100 ms increments, which quantized every measured round-trip up to the next 100 ms boundary — harmless for
ordinary RPC but fatal for the latency measurements that feed [cost-aware routing](dynamic-routing.md), where
a 10 ms link must not read as ~100 ms.
