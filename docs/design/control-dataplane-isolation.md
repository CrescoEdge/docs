# Control / Data Plane Isolation

Control-plane messages (liveness, agent registration, RPC) must not be delayed by bulk dataplane
traffic. This documents how that is enforced, and — just as importantly — the two layers where it
is *not* enough on its own.

## Three layers, and what each one actually buys

**1. Message priority (queue layer).** `MsgQoS` classifies every MsgEvent into a tier —
`LIVENESS` (JMS priority 9, persistent), `CONTROL` (7, persistent), `TELEMETRY` (4,
non-persistent), `BULK` (1). `AgentProducer` routes the control-plane tiers to a dedicated
`ControlPlaneSender`. The broker has `prioritizedMessages` on and producer flow control off, so a
backed-up destination never blocks a producer.

This orders messages **within one destination**. It does nothing across destinations, and nothing
at all below the queue layer.

**2. Dedicated sockets (transport layer).** Priority is powerless once frames are on the wire: a
256KB payload frame occupies the socket for its full transmit time, and TCP backpressure throttles
whatever shares that connection. So control gets its own TCP connections:

- `ControlPlaneSender` sends on a dedicated connection (`controlplane_dedicated_connection`).
- The agent's inbox consumer — which carries *all* inbound control, including RPC replies — has
  its own connection (`agentconsumer_dedicated_connection`).
- Between brokers, the bridge group is split by destination type
  (`broker_control_bridge_split`): connector 0 forwards **queues only** (the whole MsgEvent
  fabric), connectors 1..N forward **topics only** (all dataplane). The `>` wildcards match
  tenant-qualified names, so the split holds under tenant namespacing.

Both are `vm://`-aware: a controller talking to its own in-JVM broker has no socket to isolate and
stays pooled.

**3. Bounded prefetch.** Prefetch is the batch already committed toward a consumer or bridge ahead
of anything later. Data bridge connectors use a small `broker_bridge_prefetch_data` so at most a
few MB of bulk is ever queued ahead of a control frame on the data socket, while control keeps the
larger `broker_bridge_prefetch`. Note `queue_prefetch_limit` had been applied via
`setTopicPrefetch` and silently overwritten, leaving control inbox queues at the ActiveMQ default
of 1000; it now sets queue prefetch as intended.

## Measured effect

On a two-node fabric with deliberately constrained socket buffers (64KB, emulating a slow link),
control-plane RPC latency during a saturating stunnel transfer:

| | p50 | p99 |
|---|---|---|
| Shared sockets | 484 ms | 530 ms |
| Isolated | 2.4 ms | 5.2 ms |

Throughput is unaffected — the split bridge still carries >1.2 GB/s.

## Consequences worth knowing

**`dataplane_shards > 1` does not isolate stunnel.** Shard connectors carry only their own shard
topic, but connector 0 keeps all control queues *plus* the base topic — and stunnel publishes on
the unsharded topic. Sharding is a throughput feature; it is not a substitute for the split above.

**Health checks must probe the right connection.** `isFaultURIActive()` reports the *control*
connection once isolation is on. `DataPlaneHealthCheck` therefore probes the dataplane connection
separately (`DataPlaneServiceImpl.isDataPlaneConnectionHealthy()`); without that it reports green
while the dataplane is unusable.

**Nothing else exercises the dataplane connection any more.** Control traffic used to keep it
honest — if it broke, control broke and recovery fired. Now it can wedge alone, so
`getSession()` drops a transport-failed connection for rebuild and waits only
`dataplane_session_wait_sec` before returning `null` rather than parking its caller forever.

## Liveness paths

A region's liveness at the global is control-plane only: `RegionHealthWatcher` sends its watchdog
tick to the global (`GlobalExecutor.executeWATCHDOG` applies it), rather than depending on the
dataplane state update. Because config exports ride best-effort topics, `stateUpdateTask` forces a
full resend every `state_update_full_resend_ticks` ticks so a lost first export self-heals instead
of leaving a node's rows missing at its parent indefinitely.

Staleness detection uses two multipliers that were historically one key with two different
defaults: `watchdog_scan_multiplier` (scan cadence) and `watchdog_stale_multiplier` (staleness
threshold). The legacy `period_multiplier` is still honoured as a fallback for existing configs.

## Configuration

| Key | Default | Effect |
|-----|---------|--------|
| `controlplane_dedicated_connection` | `true` | Control-plane sender gets its own TCP connection (non-`vm://`). |
| `agentconsumer_dedicated_connection` | `true` | Agent inbox consumer gets its own TCP connection (non-`vm://`). |
| `broker_control_bridge_split` | `true` | Split broker bridges into queues-only and topics-only connectors. |
| `broker_bridge_prefetch` | `100` | Prefetch on the control bridge connector. |
| `broker_bridge_prefetch_data` | `10` | Prefetch on data bridge connectors. |
| `queue_prefetch_limit` | `100` | Queue consumer prefetch (control inboxes). |
| `topic_prefetch_limit` | `100` | Topic consumer prefetch. |
| `dataplane_session_wait_sec` | `30` | Bound on waiting for the messaging plane in `getSession()`. |
| `state_update_full_resend_ticks` | `4` | Force a full config export every N state-update ticks. |
| `watchdog_scan_multiplier` | `3` | Node-status scan cadence multiplier. |
| `watchdog_stale_multiplier` | `10` | Staleness threshold multiplier (`watchdog_period` × N). |
| `controlplane_ttl` | `300000` | TTL on control-plane messages. |

All flags default on; setting the two `*_dedicated_connection` flags and the bridge split to
`false` restores the pre-isolation behaviour exactly.

## See also

- [stunnel Data Path](stunnel-dataplane-design.md) — the bulk traffic this isolates against.
- [Health & State](../architecture/health.md) — where these checks surface.
