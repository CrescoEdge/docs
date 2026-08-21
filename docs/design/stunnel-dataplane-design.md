# stunnel Data Path Design

How a tunnel session is established, how payload is carried and demultiplexed, and what the
failure modes are. This documents the design as it exists in `code/stunnel` today; every
behaviour below is implemented in the classes named next to it.

## Two planes, two lifetimes

A **tunnel** is long-lived configuration: a SRC listener bound to a local port, paired with a DST
side that dials the real target. A **session** is one client TCP connection through that tunnel,
identified by a `client_id` (a UUID minted per accepted socket in
`SrcChannelInitializer.initChannel`).

Payload does not travel over the MsgEvent control plane. Bytes read from a socket are published as
JMS `BytesMessage` on `TopicType.GLOBAL` carrying three string properties — `stunnel_id`,
`client_id`, and `direction` (`src` or `dst`) — plus `eos` on the end-of-stream marker. Control for
a session (open/close/failure) travels as `MapMessage` with a `status` field on the same topic.

## One consumer per tunnel, demultiplexed by client

`TunnelDemux` opens **one** dataplane consumer per tunnel per direction, at tunnel setup, with the
selector:

```
stunnel_id='<id>' AND direction='dst'     # on the DST side
stunnel_id='<id>' AND direction='src'     # on the SRC side
```

Its listener reads the `client_id` property and dispatches to the handler registered for that
client. **Opening a session is a map insert — the session path makes no JMS call at all.**

This is deliberate and load-bearing. Creating a consumer per session (the previous design) meant
calling `MessageConsumer.setMessageListener` per client connection, and in ActiveMQ that calls
`ActiveMQSession.stop()`, which stops **every consumer on the shared dataplane session** and needs
each one's dispatch mutex. Three consequences, all observed in production:

- every new client connection briefly stalled the whole node's dataplane — wsapi, CEP, and every
  other tunnel, not just the tunnel being connected;
- any lock held across that call deadlocked the agent permanently, because a dispatch thread
  holding a dispatch mutex would wait on that lock while the attach waited on the mutex;
- listener add/remove churn serialized through the controller's listener map.

With the demux, JMS listener operations drop from one per connection to one per tunnel.

## Ordering

Per-client ordering is exact, and the non-obvious part is the handoff.

A session's payload can arrive **before** its handler is registered — on the DST side the SRC is
released to send as soon as the `configdstsession` RPC returns, which is before the target TCP
connect completes. `TunnelDemux.expect(clientId)` therefore starts buffering for a client before
the connect is initiated, and `register(clientId, handler)` drains that backlog when the session
goes live.

The drain cannot simply run after publishing the handler: a message arriving in that window would
be delivered live, ahead of the buffered backlog, reordering the stream. `register` instead marks
the client **draining** and delivers the backlog in passes; `dispatch` keeps buffering behind that
flag, so anything arriving mid-drain is picked up by the next pass and delivered after the
backlog. The flag is cleared only when a pass finds the buffer empty.

Delivery always happens **outside** the demux lock. A JMS consumer dispatches serially, so once a
client is live its messages are delivered in arrival order by construction.

## Bounds and failure modes

Every wait on the session path is bounded, and every failure is loud:

| Situation | Behaviour |
|-----------|-----------|
| Payload for a client that was never announced via `expect` | Dropped (debug log). No buffer is allocated, so stray or late delivery cannot grow the buffer map. |
| Buffered backlog exceeds `stunnel_demux_buffer_max` | Buffer dropped, client marked overflowed, `register` returns `false`; the handler closes the target channel and notifies the SRC. Fails the session rather than serving a truncated stream. |
| `demux.open()` blocked (dataplane unavailable) | Bounded by `stunnel_demux_open_timeout_ms`. Tunnel setup fails cleanly instead of parking the plugin message thread — `configdsttunnel` is retried every 10s by `ReconnectTask`, so an unbounded wait here would leak a thread per retry. |
| DST target connect fails | `connectWithRetry` retries 3 times, then discards the client's demux state and sends session status `9` to the SRC. |
| SRC-side session init RPC | Bounded by `stunnel_dst_init_timeout_ms`, run off the Netty event loop, with reads held (`autoRead(false)`) until the DST confirms. Concurrent inits bounded by `stunnel_dst_init_max_concurrent`. |

## Session close

Close is relayed in both directions so a half-close propagates rather than stranding the peer.
`allowHalfClosure` is on, so a peer EOF surfaces as `ChannelInputShutdownEvent`; the handler sends
an `eos` marker, the far side drains pending writes and calls `shutdownOutput`, and the channel
fully closes once both directions are done. `eos` sends are idempotent. The SRC also notifies the
DST on `channelInactive` even when the session never completed setup, because `configdstsession` is
a persistent message that can be delivered late and would otherwise orphan a target connection.

## Tunnel health check

`SocketController.startHealthCheck` runs per SRC tunnel on a **fixed delay** (not fixed rate, so a
slow probe cannot queue an overdue one behind it). Recent byte activity is proof-of-life and skips
the probe. Otherwise it sends a `tunnelhealthcheck` EXEC RPC to the DST plugin with an explicit
short timeout.

On reaching the failure threshold it sends **one more verification probe** before tearing down,
because teardown closes every client session on the tunnel and the counted failures may all be
probes sent into a stall that has since cleared. Teardown does not delete the persisted tunnel
config, so `ReconnectTask` rebuilds the tunnel automatically.

## Configuration

| Key | Default | Effect |
|-----|---------|--------|
| `stunnel_health_check_interval_sec` | `5` | Health-check period (fixed delay). |
| `stunnel_health_failure_threshold` | `3` | Consecutive failures before the verification probe and teardown. |
| `stunnel_health_probe_timeout_ms` | `10000` | RPC budget for a health probe. Per-tunnel `watchdog_timeout` (ms) overrides it. |
| `stunnel_dst_init_timeout_ms` | `10000` | RPC budget for `configdstsession`. |
| `stunnel_dst_init_max_concurrent` | `64` | Concurrent DST session initiations; excess connects fail fast. |
| `stunnel_demux_buffer_max` | `256` | Messages buffered per client before its handler registers. |
| `stunnel_demux_open_timeout_ms` | `15000` | Bound on opening a tunnel's demux consumer. |
| `stunnel_socket_buffer_bytes` | `4194304` | SO_RCVBUF/SO_SNDBUF on tunnel sockets. |
| `stunnel_read_chunk_bytes` | `262144` | Max bytes per read, and so per dataplane message. |
| `stunnel_write_high_water_bytes` | `2097152` | Netty write-buffer high-water mark. |
| `stunnel_beacon_ms` | `3000` | Tunnel existence-beacon interval. |

Per-tunnel keys (in the tunnel config map) are `stunnel_id`, `src_port`, `dst_host`, `dst_port`,
`dst_region`/`dst_agent`/`dst_plugin`, `src_region`/`src_agent`/`src_plugin`, plus optional
`buffer_size`, `watchdog_timeout` (health-probe budget, ms) and `performance_report_rate`.

## See also

- [Control/Data Plane Isolation](control-dataplane-isolation.md) — why control traffic no longer
  shares a socket with this payload.
- [stunnel plugin](../plugins/stunnel.md) — actions, classes, observability.
- [Data Plane](../architecture/dataplane.md) — the JMS transport underneath.
