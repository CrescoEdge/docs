# Dynamic Cost-Aware Routing

Cresco decides message paths **itself** — it does not defer path selection to ActiveMQ. Every controller
learns a live, mesh-wide picture of link latency, computes the lowest-latency path to any destination, and
enforces that path on real traffic. When a link slows down, when a faster alternative appears, or when two
regions that were never wired together turn out to be reachable, the routing follows automatically.

This page documents the shipped mechanism. It is the implementation of the
[Optimal Global Routing plan](../design/optimal-global-routing-plan.md) phases A/C/D (distributed-cost-first
per deviation **D1**), plus a self-organizing link-inference capability beyond the original plan. It builds
directly on the per-edge measurements in the [Network Link Metrics design](../design/link-metrics-design.md).

!!! info "Gating flags"
    The routing behaviours below are gated and can be enabled independently:
    `net_source_routing` (carry/honor a waypoint stack), `net_cost_routing` (measure, select, and inject the
    cheaper path), `net_autotune` (drive the advertise/probe/scale control loop). See the
    [Configuration reference](../reference/configuration.md#network-link-metrics-auto-tuning).

## Why ActiveMQ alone is not enough

The [broker federation](messaging.md#federation-network-of-brokers) resolves *reachability*: a
demand-forwarded message will find *a* path to a subscriber. But ActiveMQ pins one arbitrary bridge per
destination and never re-routes around a slow link or load-balances across alternatives. In a redundant mesh
— say region **R1** reachable both directly and via the global **G** — that arbitrary choice is frequently
the *worse* path, and nothing corrects it.

Cresco corrects it. The controller measures the real cost of each candidate path and, at the message's
origin, attaches the waypoint stack for the cheaper one. ActiveMQ still moves the bytes one broker hop at a
time; **Cresco decides which hops.**

## The pieces

All of the routing classes live in the controller's `netmetrics` package
([`controller` plugin](../plugins/controller.md#netmetrics-measurement)).

| Class | Role |
|---|---|
| `LinkMetrics` / `LinkMetricsRegistry` | per-edge measured latency (Jacobson/Karels smoothed RTT), jitter, backlog, throughput, and the composite `cost()` — see [link-metrics design](../design/link-metrics-design.md) |
| `RouteAdvertiser` | **pushes** this node's link-state onto the data plane and subscribes to every peer's |
| `RouteView` | the mesh-wide link-state graph assembled from those pushed advertisements |
| `RouteComputer` | Dijkstra over the `RouteView`, weighted by measured RTT → lowest-latency path + waypoint stack |
| `PathTable` | per-peer direct-vs-via-global decision from explicit probing, with anti-flap hysteresis |
| `MsgRouter` | enforces the chosen path at ingress and relays transit traffic between regions |
| `RegionHealthWatcher` | the control loop: probes paths, infers new links, verifies computed routes |
| `NetworkStateJson` | serializes the live view for the [dashboard / observability](#observability) |

## How a path is chosen and enforced

### 1. Measure — every node probes its neighbours

`RegionHealthWatcher` periodically times each neighbour with a lightweight `ping` action
([plugin action](../api/plugin-actions.md)) and records the round-trip as the edge's smoothed RTT. It times
several candidate paths to each connected region:

- the **direct** region↔region bridge,
- the **via-global** path (2-hop, forced with a source-route waypoint; only with `net_source_routing` on, since a
  global with it off does not relay the probe),
- and any **steered** multi-hop path the graph suggests.

!!! note "RTT must be measured, not polled"
    RPC replies complete **event-driven** (a `CompletableFuture` per call), not on a 100 ms poll. The old
    poll quantized every measurement up to the next 100 ms boundary, so a 10 ms link read as ~100 ms and the
    cost model could not tell fast links from slow ones. See [messaging → RPC](messaging.md#rpc).

### 2. Share — link-state is PUSHED over the data plane

Each node's `RouteAdvertiser` publishes its own neighbour edges (smoothed RTT, composite cost, and the live
AutoTuner connector count so [dynamic scaling](#dynamic-link-scaling) shows up in the graph) plus its
dialable addresses onto the `GLOBAL` [data-plane topic](dataplane.md#internal-use-link-state-advertisements),
and subscribes to everyone else's. Every controller thus assembles the same mesh-wide `RouteView`.

This is **publish/subscribe, not an RPC fan-out**. Metrics are pushed as they change; a pull that asked every
node for its state on every routing decision would not scale. RPC stays reserved for configuration, queries,
and one-time reconciliation. Advertisements are `NON_PERSISTENT`, lowest priority, and short-TTL, so they
never sit in a queue, never compete with data traffic, and a missed one is simply refreshed next tick — and
a node that goes silent ages out of the view.

### 3. Compute — Dijkstra over the learned graph

`RouteComputer` runs Dijkstra from this node to the destination over the `RouteView`, weighting each edge by
its **measured RTT** — so the result is the genuinely lowest-latency path, not the fewest broker hops. The
graph is treated as undirected (an advertised edge A→B implies B→A). For a destination two or more hops away
it emits the waypoint stack `"region,agent;region,agent;…"`; for a trivial direct destination it returns
nothing and lets default routing carry it.

`PathTable` holds the simpler per-peer direct-vs-via-global decision as a fast fallback and applies
**hysteresis**: a path must beat the incumbent by a configurable margin
(`net_route_hysteresis_ms`) before the selection flips, so a near-tie or a noisy sample cannot make the route
flap.

### 4. Enforce — source routing at the ingress

`MsgRouter.maybeInjectCostRoute()` runs at the origin region. If this node originated the message, it is
bound for another region, it carries no explicit route yet, and a cheaper path exists, the router attaches
that waypoint stack (`srcroute` param; the head becomes the message's `forwardDst`). From there the
source-routing machinery carries it:

`advanceSourceRoute()` runs at every hop. The `srcroute` param is an ordered `;`-separated stack of
`region,agent` waypoints still to visit; the message's `forwardDst` is always the head, so ActiveMQ delivers
it to that node because that node **is** the head. The node pops itself and:

- if waypoints remain → re-address `forwardDst` to the next waypoint and forward (one ActiveMQ leg);
- if the stack is now empty → restore the true destination, drop the routing headers, deliver locally.

Only locally-originated traffic is steered (a transiting message is never re-injected, which would loop), the
stack is length-capped, and the message's `BrokerPath` + TTL still bound total hops as defense-in-depth.

### Relaying: a region acts as a router

Steering a flow across `R1 → G → R2` requires **G to relay** traffic that is neither from nor for itself.
`MsgRouter` resolves any message it cannot match to a static delivery case **by destination**: addressed to
this node → deliver locally; addressed elsewhere → **relay it toward its destination**. That relay *is* the
point of a bridge — a region forwarding other regions' traffic so the mesh can carry, and steer, cross-region
flows.

!!! warning "Relaying is bounded, not an open relay"
    Every message that reaches the relay path already arrived over an mTLS-authenticated, secret-gated broker
    bridge (trusted fabric traffic). It is forwarded only toward the destination named in its own header, and
    the TTL drops it past a hop bound so a loop cannot amplify. Which bridge a transit hop takes is the path
    lookup itself (gated by `net_source_routing` / the cost selector).

### Which broker relays between bridges

The relaying above is done by controllers, hop by hop. Left to itself the broker underneath also forwards on demand
along every path of up to five brokers, so in a fabric whose regions are bridged to each other every status message
arrived once per path, and a message could take the way over another region whenever its own bridge was away. A
broker does not vouch for a caller of another region, so those second-hand copies lost their caller stamp (one
`STRIPPED a forged caller` line per controller every ten seconds at rest, and unsigned messages refused after a
restart of the global). From agent release 1.3.379 it is decided who relays (`broker_bridge_transit`):

| Value | Who relays between bridges |
|---|---|
| `auto` (default) | The global's broker. A region's broker only while its fabric has no global: it is configured with none (`global_optional=true` and no `global_controller_host`) and has not joined, or begun to join, one since it started. Once a region has had a global it does not relay again until it restarts, also while that global is down. |
| `true`, `false` | This broker relays, or does not, whatever it is. |

A broker that does not relay hands a bridge no topic message that reached it over another bridge and does not pass one
bridge's demand on to another. What waits in a region's queue still goes to whatever consumes that queue (a node that
moved to another region gets what waited for it).

!!! warning "What a fabric with a global gives up"
    A region whose own link to the global is down is no longer reached through a neighbouring region: its direct
    peers still reach it over their own bridges, the global does not until the link is back (a request for one of its
    nodes is answered `route unavailable`). Two globals joined only through bridges between their regions carry nothing
    between them unless those regions set `broker_bridge_transit=true`. A fabric without a global is unchanged: its
    regions relay for each other and learn of each other through the relayed advertisements.

## Self-organizing link inference

A computational mesh is not wired fully; it must **learn who can connect**. Because every advertisement
carries the sender's dialable addresses, a node can discover that a region it was never configured to peer
with is nonetheless reachable. `RegionHealthWatcher.inferConnections()` looks for a region it is
(transitively) connected to but not *directly* linked to, learns its address from the `RouteView`, and — where
physically possible — forms a direct bridge. It then probes the new link and **adopts it only if it is
faster** than the incumbent path; the hysteresis rule keeps a marginal new link from causing churn.

This was proven in the reference mesh: R1, with no configured peering to R4, learned R4's address from pushed
link-state, formed a direct link, and adopted it (≈16 ms direct vs ≈44 ms via-global) — confirmed by R4's own
receiver-side hop stamps showing zero transit hops.

!!! note "Not full mesh, and not region-under-region"
    Inference adds *direct region↔region* links where they help; it does not force a full mesh, and it does
    not create control-hierarchy nesting (a region's controller still uplinks to a global — see the
    [region federation design](../design/region-federation-design.md)).

## Dynamic link scaling

Independently of path selection, the [`AutoTuner`](../design/link-metrics-design.md) scales the number of
parallel bridge connectors on a link up or down with load (backlog / send-latency thresholds). That connector
count is advertised as part of each edge's link-state, so capacity changes are visible in the `RouteView` and
feed back into the cost picture. Scaling and routing are complementary: scaling adds throughput to a chosen
link; routing chooses which link.

## How this is proven

The routing claims are validated with evidence that does **not** come from the node that made the decision:

- **Independent receiver stamps.** Intermediate nodes stamp `srcroute-hop-*` params as a message transits
  them; the destination logs the trail on `PING-RECEIVED`. An empty trail means a direct single-broker hop; a
  trail naming the global proves the message physically went via-G — evidence written by other nodes, not the
  sender.
- **Physics.** A message cannot cross a 300 ms link in 30 ms. Latency measurements are cross-checked against
  the `tc netem` delays injected per link in the simulation.
- **Causal intervention.** Flipping or spiking a link's latency (via `netem`) and observing the path change
  in the next probe cycle demonstrates the routing is reacting to the live metric, not a static config.

See [Operations → Testing](../operations/testing.md) and the
[containerlab simulation](../operations/deployment.md) for the reference mesh used to produce this evidence.

## Observability

`getnetworkstate` ([plugin action](../api/plugin-actions.md), served by `AgentExecutor`/`GlobalExecutor`)
returns the live dynamic topology as JSON via `NetworkStateJson`: every fresh node, every advertised edge with
its `rtt`/`cost`/`conns`, and the observer's own per-peer path choices. Invoked on the global — which receives
every node's advertisement — it renders the whole mesh. A dashboard consumes this to show the learned graph,
inferred links appearing, connector scaling, and path flips as they happen. Because the underlying transport
is data-plane pub/sub, the same state can be **streamed** to a browser over the
[wsapi WebSocket bridge](../plugins/wsapi.md) rather than polled.

## Configuration

| Parameter | Default | Effect |
|---|---|---|
| `net_source_routing` | `false` | Carry and honor the `srcroute` waypoint stack (source routing). Off, a node starts no source route and relays none: a message source-routed through it is not run there (an RPC gets a `route_unavailable` error); one whose route ends there is delivered. Set it on every node. |
| `net_cost_routing` | `false` | Measure path cost, select the cheaper path, and inject its route at ingress (with `net_source_routing` on). |
| `net_route_advertise_interval_sec` | `5` | How often each node pushes its link-state advertisement. |
| `net_route_stale_sec` | `20` | Age after which a silent node's advertisement is dropped from the view. |
| `net_route_hysteresis_ms` | `10.0` | Margin by which a path must beat the incumbent before the route flips. |
| `broker_bridge_decrease_consumer_priority` | `true` | Make ActiveMQ default to the direct 1-hop bridge so the cost selector has a stable baseline to override. |

Full list in the [Configuration reference](../reference/configuration.md#network-link-metrics-auto-tuning).

## See also

- [Messaging & Routing](messaging.md) — addressing, the forwarders, QoS tiers
- [Data Plane](dataplane.md) — the pub/sub transport link-state rides on
- [Network Link Metrics & Auto-Tuning](../design/link-metrics-design.md) — the cost model and tuner
- [Optimal Global Routing plan](../design/optimal-global-routing-plan.md) — the design this implements
