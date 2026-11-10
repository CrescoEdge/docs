# wsapi — WebSocket API

The `wsapi` plugin (bundle `io.cresco.wsapi`) is the fabric's external client entrypoint. It embeds a Netty HTTPS/WSS server that exposes three authenticated WebSocket endpoints — control, data plane, and log streaming — over `wss://…:8282`. This is the endpoint the [Java](../clients/java.md), [Python](../clients/python.md), and [C++ / Arduino](../clients/cpp.md) client libraries connect to.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/wsapi` |
| Bundle symbolic name | `io.cresco.wsapi` |
| Java files | 34 |
| Loaded | At runtime by the controller's `StaticPluginLoader` (default-on for a global). |
| Type | Functional plugin (`@Component` implementing `PluginService`). |
| Server | Embedded **Netty** (`NettyWsServer`) on NIO event loops (1 boss + default worker group). |
| Default port | `8282` (`wss://`). |

## Endpoints

All three endpoints sit behind the `WsRouter` handler, which runs first (right after HTTP aggregation) and authenticates the WebSocket upgrade: every connection is authenticated, by a user and password, an API token or the legacy `cresco_service_key` header in the standard profile, and by a client certificate in production (see [API authentication](../security/api-auth.md)). It then routes the upgrade to the handler for the requested path.

| Handler | Path | Purpose |
|----------|------|---------|
| `ApiSocketWsHandler` | `/api/apisocket` | Control-plane / RPC. Deserializes `{message_info, message_payload}`, builds the matching [`MsgEvent`](../api/msgevent.md) by `message_type` (e.g. `global_controller_msgevent`, `global_agent_msgevent`, `global_plugin_msgevent`, `kpi_msgevent`, `agent_msgevent`, `plugin_msgevent`), and either `sendRPC` (returning response params as JSON) or `msgOut`, per the `is_rpc` flag. |
| `DataPlaneWsHandler` | `/api/dataplane` | Bridges a client WebSocket to the global JMS [data plane](../architecture/dataplane.md). The first message subscribes (creates a JMS listener from `ident_key`/`ident_id`/`io_type_key`/`output_id`/`input_id`, or a raw `stream_query`); subsequent text → publish and whole-binary → `BytesMessage` on `TopicType.GLOBAL` (with optional `seq_num`/`transfer_id` framing). |
| `LogStreamerWsHandler` | `/api/logstreamer` | Subscribes to agent logger data-plane messages (filtered by session) and RPCs `setloglevel` to set remote agent log levels. |

## Key classes

| Class | Role |
|-------|------|
| `Plugin` | Builds and manages the `NettyWsServer`; ensures its managed TLS keystore `ws.keystore` in the plugin data directory (key and signature from the fabric crypto policy: EC P-384 with `SHA384withECDSA` by default; a random password in `ws.keystore.pass`; BCFKS in FIPS mode) or loads a CA-issued one (`wsapi_keystore_file`); reads the port/buffer config and starts the server. |
| `NettyWsServer` | The Netty bootstrap: NIO `MultiThreadIoEventLoopGroup`s (1 boss + default workers), the TLS + WebSocket pipeline, and adaptive receive buffers. |
| `WsRouter` | First pipeline handler after HTTP aggregation: authenticates the upgrade (`Authenticator`, `auth` package) and routes it to the per-path handler. |
| `ApiSocketWsHandler` / `DataPlaneWsHandler` / `LogStreamerWsHandler` | The three endpoint handlers (control-plane RPC, data-plane bridge, log streamer). |
| `PluginExecutor` | EXEC action `globalinfo` (sets `global_region`/`global_agent`). |
| `StreamInfo` | Per-stream data holder (in the `websockets` package). |
| `Activator` | Routes third-party logging through SLF4J. |

## TLS and performance tuning

`NettyWsServer` is tuned for high-throughput streaming:

- TLS via an SSL context whose provider is selectable with `wsapi_ssl_provider` (default `JDK`; `OPENSSL` is refused in FIPS mode).
- Adaptive receive-buffer allocation (`AdaptiveRecvByteBufAllocator`, 2 KB … 64 KB … up to `wsapi_read_chunk_bytes`) and `wsapi_socket_buffer_bytes` (4 MB) socket buffers, with a `wsapi_write_high_water_bytes` (2 MB) write watermark for backpressure.
- **`permessage-deflate` is disabled** (`WsRouter` sets `allowExtensions(false)`) — WebSocket compression was the single biggest data-plane CPU bottleneck, so it is deliberately off. See [Data Plane](../architecture/dataplane.md) and [Metrics & Measurements](../architecture/metrics.md).

## Authentication and identity

Every connection is authenticated on the WebSocket upgrade; the credentials, roles and limits are in
[API authentication](../security/api-auth.md). wsapi stamps each request with the verified API user as its caller and
removes the fabric's own stamps (`cresco_sig*`, `cresco_caller*`, routing annotations, RPC plumbing) from what a client
sends and receives. On the return path, external identity — a client's region/agent/plugin — is read from the TLS peer
certificate DN. See [Security & Identity](../architecture/security.md).

## API RPCs

An RPC from `/api/apisocket` (`is_rpc`) waits for its reply on a bounded pool, never on a Netty event loop: an idle thread
takes the next RPC at once (the most recently idle first), with at most 640 RPCs in flight (128 + 512). A full pool is
answered at once with `503 busy`, and a caller past its own in-flight limit (`wsapi_rpc_max_inflight_user`, 8;
`wsapi_rpc_max_inflight_admin`, 32) with `429 busy`. Replies may complete out of order; clients correlate them
(pycrescolib stamps `client_rpc_id`, echoed back).

A message sent without waiting for a reply (`is_rpc` false in `message_info`) has no reply to count, so it counts
against a rate: `wsapi_send_max_per_second_user` (50) or `wsapi_send_max_per_second_admin` (500) a second for each
caller, across its connections. Such sends go to the controller off the network threads, one at a time for each caller,
in the order they arrived; what arrives faster waits (wsapi stops reading from a connection with 64 sends, or 1 MiB of
them, waiting) and nothing is dropped. A send and an RPC that follows it on the same connection may reach the
controller in either order.

What a client may not set: every parameter infrastructure writes is removed from a client's payload -- the caller
stamp, signatures, routing parameters, call ids, and the hop count `ttl` and the RPC flag `is_rpc` (whether a request
is an RPC is said in its `message_info`). A `user` addresses only its own region; that is decided from the message
wsapi builds, so a message for the global controller is one whatever `region_id` it names, and is refused (403) when
the global controller is in another region.

## Dataplane backpressure

A dataplane consumer (`/api/dataplane`) is a `PacedListener`: the controller gives it a JMS session of its own with a
consumer prefetch of 16 (`dataplane_paced_prefetch`). Frames queue on the WebSocket past Netty's write high-water mark up
to the consumer's budget (`wsapi_dataplane_egress_buffer_bytes`, 32 MiB; all consumers together
`wsapi_dataplane_egress_global_bytes`, a quarter of the heap, at most 512 MiB, shared equally by the consumers that hold
a backlog). A consumer past its own budget, or past its share when all together are past theirs, **waits** until its
backlog is back to half; a consumer that keeps up never waits for another's backlog. Meanwhile the rest of the stream stays at the broker, whose flow control slows the
stream's writers ([Data Plane](../architecture/dataplane.md#backpressure-nothing-lost-order-kept)). Nothing is lost and
the order is kept.

A held consumer that reads less than half its budget in `wsapi_dataplane_stall_close_ms` (16 MiB in 30 s by default,
about 0.5 MiB/s) is closed with WebSocket status **1013** (`stalled: <n> B read in 30000 ms, less than <m>`; counted in
`wsapi.dataplane.egress.stalled`): a reader that stopped, or one that takes a trickle, would otherwise hold every
writer of its topic on that broker. A link that is legitimately slower than that needs a smaller
`wsapi_dataplane_egress_buffer_bytes` or a longer `wsapi_dataplane_stall_close_ms`. Client frames are published by send
threads that belong to a dataplane shard: a stream whose topic is full holds only the threads of its own shard. `wsapi_dataplane_backpressure=false` restores the
earlier budget, past which frames are dropped (counted in `wsapi.dataplane.egress.dropped`).

Client frames are published off the Netty event loop, on a per-connection send thread (order kept; up to
`wsapi_dataplane_send_threads` threads, the core count from 2 to 32), with reading paused while more than
`wsapi_dataplane_ingress_queue_bytes` (16 MiB) wait to be published and resumed at half of it, so a send that waits on a
full topic never stalls the event loop or the other connections it serves. A frame the controller does not take is
counted (`wsapi.dataplane.ingress.failed`) and logged.

Close codes a client may see:

| Code | Reason | When |
|---|---|---|
| 1013 | `stalled: <n> B read in <ms> ms, less than <m>` | a held dataplane consumer read less than half its budget in `wsapi_dataplane_stall_close_ms` while frames waited |
| 1013 | `too many message bytes in flight (...); retry shortly` | a caller (`wsapi_max_inflight_bytes_per_user`, 256 MiB) or all callers (`wsapi_max_inflight_bytes`) held too many inbound message bytes |
| 1008 | `Subscription refused: ...` | a dataplane subscription the caller may not make |
| 1008 | `credentials revoked or changed: authenticate again` | the connection's credential was revoked or changed (checked every `wsapi_revocation_check_ms`, 1 s) |
| 1009 | `message larger than <n> bytes (...)` | an inbound message over its size cap |

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `wsapi_port` | `8282` | WSS/HTTPS listen port. |
| `wsapi_keystore_file`, `wsapi_keystore_password` | — | A CA-issued PKCS12 keystore and its password (or `wsapi_keystore_password_file`); unset: wsapi's managed `ws.keystore`. |
| `cresco_service_key` | — | The legacy service key (an `admin` of the agent's tenant); refused in production. |
| `wsapi_ssl_provider` | `JDK` | Netty SSL provider (`JDK` or `OPENSSL`; `OPENSSL` refused in FIPS mode). |
| `dataplane_io_buffer_bytes` | `65536` | Data-plane I/O buffer size (64 KB). |
| `wsapi_read_chunk_bytes` | `262144` | Max adaptive read chunk (256 KB). |
| `wsapi_socket_buffer_bytes` | `4194304` | Socket send/receive buffer (4 MB). |
| `wsapi_write_high_water_bytes` | `2097152` | Netty write-buffer high-water mark (2 MB). |
| `wsapi_dataplane_backpressure` | `true` | a consumer past its budget waits (its stream slows); `false`: frames past it are dropped, counted |
| `wsapi_dataplane_egress_buffer_bytes` | `33554432` | one consumer's egress budget (32 MiB) |
| `wsapi_dataplane_egress_global_bytes` | min(512 MiB, heap / 4) | all consumers' egress budget |
| `wsapi_dataplane_stall_close_ms` | `30000` | a held consumer that reads less than half its budget in this long is closed (1013); at least 1000 |
| `wsapi_send_max_per_second_user` / `wsapi_send_max_per_second_admin` | `50` / `500` | messages without a reply (`is_rpc` false) per caller per second; more wait, none is dropped |
| `wsapi_dataplane_send_threads` | cores, 2 to 32 | threads publishing client frames (one per connection, order kept); `0`: publish on the event loop |
| `wsapi_dataplane_ingress_queue_bytes` | `16777216` | client bytes waiting to be published before reading pauses (16 MiB; at least 1 MiB) |

The authentication settings (`wsapi_users_file`, tokens, lockouts, client certificates) are in
[API authentication](../security/api-auth.md).

See [Configuration](../getting-started/configuration.md).

## Observability

- **Metrics** — `getmetrics` exposes `MeasurementEngine` gauges for the dataplane (`wsapi.dataplane.connections`,
  `.bytes`, `.messages`, `.egress.pending.bytes`, `.egress.max.pending.bytes`, `.egress.paused`, `.egress.stalled`,
  `.egress.dropped`, `.ingress.failed`), authentication (`wsapi.auth.failures`, `.servicekey.failures`, `.refused`,
  `.revoked.connections`) and inbound memory (`wsapi.inbound.inflight.bytes`, `wsapi.inbound.budget.closed`),
  aggregated by `getmetricinventory`. See
  [Metrics & Measurements › wsapi](../architecture/metrics.md#plugin-wsapi).
- **Health** — registers the `wsapi` Felix HealthCheck (tag `local`), **only after the Netty server
  binds**, so an OK reports the live wss port. See [Health & State](../architecture/health.md#plugin-checks).

## See also

- [Client Libraries Overview](../clients/overview.md) · [Java (clientlib)](../clients/java.md) · [Python (pycrescolib)](../clients/python.md)
- [Data Plane](../architecture/dataplane.md) — what `/api/dataplane` bridges to.
- [Security & Identity](../architecture/security.md) — the auth header and TLS-cert identity model.
