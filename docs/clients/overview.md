# Client Libraries

Cresco ships three external client SDKs for driving a mesh from outside the OSGi
runtime:

- **[Java — `clientlib`](java.md)** — the Java SDK (`io.cresco:clientlib`).
- **[Python — `pycrescolib`](python.md)** — the Python SDK.
- **[C++ / Arduino — `cppcrescolib`](cpp.md)** — the embedded SDK for ESP32 (ESP32-S3) edge nodes.

All three connect to a running Cresco agent that hosts the **`wsapi`** plugin over a
secure WebSocket (`wss://host:8282`) and drive the fabric — controlling agents,
deploying plugins, submitting pipelines, streaming the data plane, and tailing
logs.

## The shared model

Every interaction follows the same shape regardless of language:

1. **Connect once.** Construct a client with the target host and port, how to
   trust wsapi's certificate (its pinned fingerprint or a CA file) and a
   credential (a user and password, an API token, the legacy service key, or a
   client certificate; see [API authentication](../security/api-auth.md)), then
   `connect()`. The client opens a single authenticated control connection to
   the `wsapi` endpoint.
2. **Use submodule handles.** The connected client exposes a fixed set of
   submodules, each grouping a family of operations:

    | Submodule | Purpose |
    |---|---|
    | `api` | Local and global identity lookups (region/agent/plugin names). |
    | `admin` | Node lifecycle on a target agent (stop/restart controller, restart framework, kill JVM). |
    | `agents` | Per-agent operations: plugin add/remove/list/status, uploads, logs, CEP. |
    | `globalcontroller` | Mesh-wide inventory and application ops: regions, agents, resources, pipelines, repositories, metric/capability inventories. |
    | `messaging` | Low-level MsgEvent routing (RPC and fire-and-forget) to controllers, agents, and plugins. |
    | `stunnel` | Secure TCP tunnels through the [stunnel](../plugins/stunnel.md) plugin (Python and Java). |
    | `executor` | OS processes and their stdin through the [executor](../plugins/executor.md) plugin (Python and Java). |
    | `filerepo` | Verified file downloads from the [filerepo](../plugins/filerepo.md) plugin (Python and Java). |

3. **Open streams as needed.** Data-plane and log-streamer channels are
   *separate* WebSocket connections opened on demand (`get_dataplane(...)`,
   `get_logstreamer(...)`). They are tracked by name on the client, delivered to
   a callback, and auto-reconnect if the socket drops.

4. **Close.** `close()` tears down the control connection and every managed
   stream.

### RPC vs. fire-and-forget

Messaging calls carry an `is_rpc` flag. When `true`, the call blocks for the
single reply and returns it; when `false`, the message is sent without waiting.
wsapi may complete RPCs out of order, so the clients correlate them: pycrescolib
stamps each request with a `client_rpc_id` that the reply echoes, and drops a late
reply to a call that already timed out instead of reading it as the next call's
answer ([wsapi › API RPCs](../plugins/wsapi.md#api-rpcs)).

## Feature parity

!!! note "Standardized clients"
    The Python and Java clients are kept **feature- and name-identical**: the same
    submodules (`api`, `admin`, `agents`, `globalcontroller`, `messaging`, `stunnel`,
    `executor`, `filerepo`), the same method names (snake_case in both), the same
    behavior, TLS and credential options, and the same worked examples. The C++
    client shares the first five submodules and their names, with the service key
    only (see [TLS verification](#tls-verification)).

Parity is enforced by test: the clients ship a self-contained suite that
asserts they emit **identical wire messages for identical API calls**, verified
against the same generated golden corpus. A green run proves the clients produce
byte-for-byte equivalent results.

Beyond that, the differences are the ones each platform forces:

| | Python (`pycrescolib`) | Java (`clientlib`) | C++ (`cppcrescolib`) |
|---|---|---|---|
| Concurrency | asyncio under a blocking surface | background reader threads | single cooperative `loop()` |
| Structured returns | `dict` / `list` | `Map` / `List` / JSON | `ArduinoJson::JsonDocument` |
| Target | server / desktop | server / desktop / JVM | ESP32-S3 edge node |

## TLS verification

wsapi serves its own certificate (written to `ws.crt.pem`, with its SHA-256 and
SHA-384 fingerprints in `ws.fingerprint`, in its plugin data directory). The
**Python (1.4.0+) and Java clients verify it on every connection by default** and
send nothing to a server they do not trust: give them the fingerprint
(`fingerprint=` / `pinnedFingerprint(...)`, or `CRESCO_TLS_FINGERPRINT`) or the PEM
file (`ca_file=` / `caFile(...)`). `verify_ssl=False` (Java: `.insecure()`) turns
verification off and logs a WARNING on every connection. See
[API authentication](../security/api-auth.md).

!!! warning "The C++ / Arduino client"
    `cppcrescolib` still takes only the service key and defaults to
    `verify_ssl=false`; it has none of the 1.3 credentials (users, tokens, client
    certificates), so it cannot reach a wsapi running the `production` profile.
    See [C++ / Arduino](cpp.md).

## Where to go next

- **[Java (clientlib)](java.md)** — installation, connection, full interface
  reference, and worked examples.
- **[Python (pycrescolib)](python.md)** — the equivalent Python SDK.
- **[C++ / Arduino (cppcrescolib)](cpp.md)** — the embedded ESP32 SDK.
- **[MsgEvent](../api/msgevent.md)** — the message envelope and routing model
  the `messaging` submodule builds on.
- **[Plugin Actions](../api/plugin-actions.md)** — the action catalog exposed by
  controllers and plugins.
