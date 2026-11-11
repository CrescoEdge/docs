# Python Client (`pycrescolib`)

`pycrescolib` is the official Python SDK for driving a Cresco fabric. It opens a single
authenticated, secure WebSocket control connection to an agent running the
[`wsapi`](../plugins/wsapi.md) plugin (`wss://host:8282`) and exposes a set of interface
objects that let you inspect the mesh, deploy plugins, submit pipelines, and open
separate stream sockets for the data plane and for live logs.

The Python, Java, and C++ clients are kept **feature- and name-identical**: the same
submodules, the same method names (snake_case in all three languages), the same wire messages,
and the same worked examples. Anything you can do in one you can do in the others with the same
call — see the [Java Client](java.md) and [C++ / Arduino Client](cpp.md) for the equivalents.

!!! info "Async underneath, blocking on the surface"
    The client was migrated to `asyncio` + [`websockets`](https://websockets.readthedocs.io/):
    every socket operation runs on a dedicated background event-loop thread. The **public API
    that you call is synchronous** — control-plane methods block until the reply arrives (RPC)
    or until the message is flushed (fire-and-forget). You do **not** `await` `clientlib`
    methods. The `async`/`await` mechanics are internal (`ws_interface`, `messaging`), so the
    client is safe to use from ordinary synchronous scripts, notebooks, and test suites without
    an event loop of your own. See [Async model](#async-model) for details.

---

## Installation

```bash
pip install pycrescolib
```

From a source checkout:

```bash
pip install .          # install into the current environment
python -m build        # build sdist + wheel into dist/
```

| Requirement | Value |
|---|---|
| Package name | `pycrescolib` |
| Version | 1.5.0 |
| Python | 3.9+ (tested through 3.12) |
| License | Apache-2.0 |

### Dependencies

| Dependency | Constraint | Used for |
|---|---|---|
| [`websockets`](https://pypi.org/project/websockets/) | `>=14.0` | The WebSocket transport for the control connection and stream sockets. |
| [`cryptography`](https://pypi.org/project/cryptography/) | `>=36.0.0` | Parsing the server X.509 certificate to extract the agent's region / agent / plugin identity. |
| [`backoff`](https://pypi.org/project/backoff/) | `>=2.0.0` | Exponential-backoff retry on transient connection / timeout errors. |

The optional `test` extra (`pip install pycrescolib[test]`) adds `pytest` for the
cross-language conformance suite.

---

## Connection

The single entry point is the `clientlib` class:

```python
from pycrescolib.clientlib import clientlib

client = clientlib(host, port, service_key=None, verify_ssl=None, *, ca_file=None, fingerprint=None,
                   check_hostname=True, user=None, password=None, token=None,
                   client_cert=None, client_key=None, client_key_password=None)
```

| Argument | Type | Description |
|---|---|---|
| `host` | `str` | Host or IP of an agent running the `wsapi` plugin. |
| `port` | `int` | `wsapi` WebSocket port (typically `8282`). |
| `service_key` | `str` | The legacy `wsapi` service key (an `admin` of the agent's tenant), sent as the `cresco_service_key` header. Omit it when giving another credential. |
| `fingerprint` / `ca_file` | `str` | How to trust wsapi's certificate: its pinned SHA-256 or SHA-384 fingerprint (from `ws.fingerprint`; several allowed for a rotation), or a PEM file (`ws.crt.pem` or a CA bundle). Unset: `CRESCO_TLS_FINGERPRINT` / `CRESCO_TLS_CA_FILE`, then the system trust store. |
| `verify_ssl` | `bool` | Not given (the default): verify. `False` turns verification off; the credentials then go to whoever answers and every connection logs a WARNING. |
| `user` / `password` | `str` | A per-user credential from wsapi's credential file (HTTP Basic inside TLS). |
| `token` | `str` | An API token (`cwt1_...`), sent as `Authorization: Bearer`; the choice for tools and automation. Unset with no other credential: `CRESCO_API_TOKEN`. |
| `client_cert` / `client_key` / `client_key_password` | `str` | A client certificate (PEM with its key, or PKCS12) presented on every connection; the only credential wsapi accepts under the `production` profile. |

See [API authentication](../security/api-auth.md) for the credentials and how wsapi checks them.

Constructing a `clientlib` wires up the transport and the interface objects but does **not**
open the socket. Call `connect()` to establish the connection:

```python
if client.connect():
    print("connected:", client.connected())
    # ... use the client ...
    client.close()
else:
    print("failed to connect")
```

Under the hood, `connect()` checks wsapi's certificate, opens `wss://{host}:{port}/api/apisocket`, authenticates with
the client's credential, verifies the socket is live, and reads the mesh identity (region / agent / plugin)
out of the server certificate's Distinguished Name (`O`=region, `OU`=agent, `CN`=plugin).

| Method | Signature | Description |
|---|---|---|
| Construct | `clientlib(host, port, service_key=None, verify_ssl=None, *, ...)` | Create the client for a `wsapi` endpoint (does not connect). |
| Connect | `connect() -> bool` | Open and verify the control connection. Returns `True` on success. |
| Status | `connected() -> bool` | Whether the control connection is currently live. |
| Close | `close() -> None` | Close the control connection and every managed stream. |

### Context management

`clientlib.connection()` is a context manager that connects on entry and always closes on
exit (including on exception). It raises `ConnectionError` if the connection cannot be
established:

```python
from pycrescolib.clientlib import clientlib

client = clientlib("localhost", 8282, "your-service-key", fingerprint="sha384:AB:CD:...")
with client.connection() as c:
    print(c.globalcontroller.get_region_list())
# connection is closed automatically here
```

### TLS verification

Since 1.4.0 every connection (the API socket, every dataplane and every log stream) verifies wsapi's certificate
before it sends anything; 1.3.2 and earlier did not verify. wsapi makes its own certificate, so pin it or trust its PEM
file:

```python
client = clientlib("localhost", 8282, "your-service-key", fingerprint="sha384:AB:CD:...")   # pin it
client = clientlib("localhost", 8282, "your-service-key", ca_file="ws.crt.pem")             # or trust it
```

With neither (and no `CRESCO_TLS_*` variable) the system trust store applies, which does not trust a self-signed
wsapi. A server that does not match gets nothing: `connect()` returns `False` and `client.last_error` says why
(`pycrescolib.tls`: `CertificateMismatch`, `CertificateVerificationError`, `AuthenticationError`, `TLSConfigError`).

---

## Async model

Although the calls you make are blocking, it is worth understanding the machinery so you can
reason about threads and timeouts:

- **Dedicated event loop.** The transport (`ws_interface`) starts its own `asyncio` event
  loop on a background daemon thread the first time you `connect()`. All WebSocket I/O runs on
  that loop.
- **Blocking wrappers.** The control-plane interfaces use `messaging_sync`, which submits
  coroutines to the background loop with `asyncio.run_coroutine_threadsafe(...)` and waits on
  the result. That is why `client.globalcontroller.get_region_list()` returns a plain `list`
  rather than a coroutine.
- **RPC vs. fire-and-forget.** Each messaging call carries an `is_rpc` flag. When `True`, the
  call blocks for the reply and returns it as a `dict`. When `False`, the message is flushed
  and the call returns `None` without waiting.
- **Timeouts.** RPC calls take a `timeout` (default `8.0` seconds). A control socket the server
  closed raises `ConnectionError` and `connected()` turns `False`, so reconnect with `connect()`.
  Inventory queries raise (`CrescoQueryError`, `TimeoutError`, `ConnectionError`) when they could not
  ask, and return an empty list or `None` only when the fabric answered "none" (since 1.3.2).
- **Retries.** Transient `ConnectionError` / `TimeoutError` failures are retried with
  exponential backoff (up to 3 attempts) via `backoff`.
- **Stream sockets self-manage.** Each dataplane and logstreamer spins up its own event loop
  thread, delivers inbound frames to your callback via an executor, and includes an automatic
  reconnect monitor.

You call the client from ordinary synchronous code; no `await` or `asyncio.run(...)` is
required on your side.

---

## Interfaces

After `connect()`, a `clientlib` exposes the following interface objects as attributes. Each
groups a set of related operations.

| Attribute | Class | Purpose |
|---|---|---|
| `client.api` | `api` | Identity of the connected agent and of the global controller. |
| `client.globalcontroller` | `globalcontroller` | Fabric-wide operations: regions, agents, pipelines, repos, resources, inventories. |
| `client.agents` | `agents` | Per-agent plugin lifecycle, agent info/logs, controller status, CEP. |
| `client.admin` | `admin` | Administrative controls: restart / stop controller, restart framework, kill JVM. |
| `client.messaging` | `messaging_sync` | Low-level typed message routing (the transport all interfaces build on). |
| `client.stunnel` | `stunnel` | Secure TCP tunnels through the [stunnel](../plugins/stunnel.md) plugin. |
| `client.executor` | `executor` | OS processes and their stdin through the [executor](../plugins/executor.md) plugin. |
| `client.filerepo` | `filerepo` | Verified file downloads from the [filerepo](../plugins/filerepo.md) plugin. |
| `get_dataplane(...)` | `dataplane` | Open a data-plane stream socket (created on demand). |
| `get_logstreamer(...)` | `logstreamer` | Open a live-log stream socket (created on demand). |

The method tables for `stunnel`, `executor` and `filerepo` are in the
[pycrescolib README](https://github.com/CrescoEdge/pycrescolib#api-reference); the names match the Java client's
(see [Java › stunnel](java.md#stunnel)).

### `api`

Identity of the agent hosting the `wsapi` plugin and of the global controller. The API
identity values are read from the server certificate at connect time; the global identity is
fetched lazily on first access.

| Method | Signature | Description |
|---|---|---|
| `get_api_region_name()` | `-> str` | Region of the agent hosting the `wsapi` plugin. |
| `get_api_agent_name()` | `-> str` | Agent name of the agent hosting the `wsapi` plugin. |
| `get_api_plugin_name()` | `-> str` | Plugin name of the `wsapi` plugin. |
| `get_global_region()` | `-> str \| None` | Region of the global controller. |
| `get_global_agent()` | `-> str \| None` | Agent of the global controller. |
| `get_global_info()` | `-> tuple[str \| None, str \| None]` | `(global_region, global_agent)` in one call. |

### `globalcontroller`

Fabric-wide operations routed through the global controller.

| Method | Signature | Description |
|---|---|---|
| `get_region_list()` | `-> list[dict]` | List all regions in the mesh. |
| `get_agent_list(dst_region=None)` | `-> list[dict]` | List agents, optionally filtered to one region. |
| `get_agent_resources(dst_region, dst_agent)` | `-> dict` | Resource/perf summary for one agent. |
| `get_region_resources(dst_region)` | `-> dict` | Resource summary for one region. |
| `submit_pipeline(cadl, tenant_id='0')` | `-> dict` | Submit a CADL pipeline; reply carries `gpipeline_id`. |
| `remove_pipeline(pipeline_id)` | `-> dict` | Tear down a pipeline. |
| `get_pipeline_list()` | `-> list[dict]` | List all pipelines. |
| `get_pipeline_info(pipeline_id)` | `-> dict` | Full pipeline record. |
| `get_pipeline_status(pipeline_id)` | `-> int` | Pipeline status code (`10` = active). |
| `get_pipeline_id_by_name(pipeline_name)` | `-> str \| None` | Resolve a pipeline ID from its name. |
| `get_pipeline_export(pipeline_id)` | `-> dict` | Exportable representation of a pipeline. |
| `get_pipeline_is_assignment_info(inode_id, resource_id)` | `-> dict` | inode-to-resource assignment info. |
| `get_plugin_repo_list()` | `-> dict` | Plugins available in each configured repository. |
| `get_repo_plugins()` | `-> dict` | Plugins known to the repositories. |
| `upload_plugin_global(jar_file_path)` | `-> dict` | Upload a plugin jar to the global repository. |
| `get_metric_inventory(scope='global', dst_region=None, dst_agent=None, include_plugins=True, include_resource=True, timeout=45.0)` | `-> dict` | Unified metric inventory: controller + plugin metrics, optional resource summary. See [Metrics & Measurements](../architecture/metrics.md). |
| `get_capability_inventory(scope='global', dst_region=None, dst_agent=None, include_plugins=True, include_osgi=False, timeout=45.0)` | `-> dict` | Self-describing capability catalog (LLM-facing tool descriptors). See [Plugin Actions](../api/plugin-actions.md). |

!!! note "Inventory scope"
    `get_metric_inventory` and `get_capability_inventory` take `scope` = `'node'`, `'region'`,
    or `'global'`. Pass both `dst_region` and `dst_agent` to target a single agent's controller
    directly (node scope). Whole-mesh (`'global'`) queries fan out, so `timeout` defaults high.

### `agents`

Per-agent plugin lifecycle and inspection. `dst_region` / `dst_agent` name the target agent.

| Method | Signature | Description |
|---|---|---|
| `is_controller_active(dst_region, dst_agent)` | `-> bool` | Whether the target agent's controller is active. |
| `get_controller_status(dst_region, dst_agent)` | `-> dict` | Controller status of the target agent. |
| `get_agent_info(dst_region, dst_agent)` | `-> dict` | Agent metadata. |
| `get_agent_log(dst_region, dst_agent)` | `-> dict` | Recent agent log data. |
| `list_plugin_agent(dst_region, dst_agent)` | `-> list[dict]` | Plugins currently on the agent. |
| `status_plugin_agent(dst_region, dst_agent, plugin_id)` | `-> dict` | Status of one plugin. |
| `add_plugin_agent(dst_region, dst_agent, configparams, edges=None)` | `-> dict` | Add a plugin; reply carries `pluginid`. |
| `remove_plugin_agent(dst_region, dst_agent, plugin_id)` | `-> dict` | Remove a plugin. |
| `repo_pull_plugin_agent(dst_region, dst_agent, jar_file_path)` | `-> dict` | Pull a plugin jar from a repo onto the agent. |
| `upload_plugin_agent(dst_region, dst_agent, jar_file_path)` | `-> dict` | Upload a plugin jar directly to the agent. |
| `update_plugin_agent(dst_region, dst_agent, jar_file_path)` | `-> dict` | Update a plugin on the agent. |
| `get_broadcast_discovery(dst_region, dst_agent)` | `-> dict` | Broadcast-discovery info from the agent. |
| `cepadd(input_stream, input_stream_desc, output_stream, output_stream_desc, query, dst_region, dst_agent)` | `-> dict` | Register a Complex-Event-Processing query on the agent. |

### `admin`

Administrative controls for a target agent. These are fire-and-forget (`CONFIG`) actions and
return `None`.

| Method | Signature | Description |
|---|---|---|
| `stopcontroller(dst_region, dst_agent)` | `-> None` | Stop the agent's controller plugin. |
| `restartcontroller(dst_region, dst_agent)` | `-> None` | Restart the agent's controller plugin. |
| `restartframework(dst_region, dst_agent)` | `-> None` | Restart the underlying OSGi framework. |
| `killjvm(dst_region, dst_agent)` | `-> None` | Force-terminate the agent JVM. |

!!! warning
    `restartframework` and `killjvm` disrupt the target agent. Guard them behind an
    `is_controller_active(...)` check (as the [admin example](#restart-an-agent-guarded) does).

### `messaging`

The low-level typed message router that every other interface builds on. Each method takes
`(is_rpc, message_event_type, message_payload, ...)` plus optional routing fields, and returns
the reply `dict` for RPC calls or `None` otherwise. `message_event_type` is a
[MsgEvent](../api/msgevent.md) type such as `'EXEC'` or `'CONFIG'`; `message_payload` is a
`dict` whose `action` selects the operation. See [MsgEvent](../api/msgevent.md) for the wire
contract.

| Method | Routes to |
|---|---|
| `global_controller_msgevent(is_rpc, event_type, payload, timeout=8.0, ...)` | the global controller |
| `regional_controller_msgevent(is_rpc, event_type, payload, timeout=8.0, ...)` | the regional controller |
| `global_agent_msgevent(is_rpc, event_type, payload, dst_region, dst_agent, timeout=8.0)` | an agent, via the global controller |
| `regional_agent_msgevent(is_rpc, event_type, payload, dst_agent, timeout=8.0)` | an agent in the local region |
| `agent_msgevent(is_rpc, event_type, payload, timeout=8.0)` | the local agent |
| `global_plugin_msgevent(is_rpc, event_type, payload, dst_region, dst_agent, dst_plugin, timeout=8.0)` | a plugin, via the global controller |
| `regional_plugin_msgevent(is_rpc, event_type, payload, dst_agent, dst_plugin, timeout=8.0)` | a plugin in the local region |
| `plugin_msgevent(is_rpc, event_type, payload, dst_plugin, timeout=8.0)` | a local plugin |

Every method raises what went wrong: `ConnectionError` when the socket is gone, `TimeoutError` when no answer came in
time. An empty `dict` is an answer, never an error. (Before 1.5.0 `agent_msgevent`, `regional_agent_msgevent` and
`regional_plugin_msgevent` returned `{}` for both.) A reply may be up to 64 MiB, wsapi's own limit for the API socket
(`CRESCO_API_MAX_MESSAGE` changes the client's side); before 1.5.0 a reply over 1 MiB closed the connection.

Example of a direct RPC through `messaging` (equivalent to the higher-level helpers):

```python
reply = client.messaging.global_controller_msgevent(
    True,                       # is_rpc: block for the reply
    "EXEC",                     # MsgEvent type
    {"action": "listregions"},  # payload
)
```

### `dataplane`

A data-plane stream is a separate WebSocket (`wss://host:port/api/dataplane`) obtained from
the client. It is created on demand, tracked by stream name, auto-reconnects, and delivers
inbound frames to the callbacks you supply.

```python
dp = client.get_dataplane(stream_name, callback=None, binary_callback=None)
```

| Method | Signature | Description |
|---|---|---|
| `client.get_dataplane(stream_name, callback=None, binary_callback=None)` | `-> dataplane` | Create or fetch the dataplane for a stream query. `callback(msg)` handles text frames, `binary_callback(bytes)` handles binary frames. |
| `connect()` | `-> bool` | Open the stream and block (up to 5 s) for activation. `False` with the reason in `last_error` when the connection failed or wsapi refused the handshake; raises `DataplaneSubscriptionRefused` for a receiver whose subscription the broker refused. |
| `connected()` / `is_active()` | `-> bool` | Whether the stream is active. |
| `send(data, timeout=None)` | `-> SendResult` | Send a text or bytes payload. |
| `send_binary(data, timeout=None)` | `-> SendResult` | Send a `bytes` payload (raises `TypeError` if not bytes). |
| `send_binary_file(file_path, timeout=None)` | `-> SendResult` | Read a file and send it as one binary message. |
| `send_partial(data, complete, timeout=None)` | `-> SendResult` | Append a binary fragment (`PENDING`); when `complete` is `True`, flush the accumulated fragments as one fragmented message (`SENT`). |
| `update_config(dst_region, dst_agent)` | `-> SendResult` | Reconfigure the stream to target an agent. |
| `get_metrics()` | `-> dict` | Client-side counters: messages/bytes sent and received, and `active`. |
| `close()` | `-> None` | Close the stream. |
| `client.close_dataplane(stream_name)` | `-> bool` | Close and deregister a named dataplane on the client. |
| `client.get_active_dataplanes()` | `-> list[str]` | Names of currently registered dataplanes. |

**What a send returns (1.5.0).** A `SendResult`: `SENT` (handed to the socket; not an acknowledgement from a
receiver), `PENDING` (a fragment waiting for the final one), `DROPPED` (the stream was not active: nothing was written)
or `FAILED` (the write raised or did not finish in time: the stream reconnects). It is true in a condition unless the
data was lost, and `dp.last_send_error` says why:

```python
if not dp.send_binary(chunk):
    raise RuntimeError(f"chunk not sent: {dp.last_send_error}")
```

- A send may take as long as the stream's `send_timeout` (seconds; `CRESCO_DATAPLANE_SEND_TIMEOUT`, default 30) or
  `timeout=` on the call. wsapi slows a sender down when the fabric cannot keep up
  ([back-pressure](../architecture/dataplane.md)), so a slow send is not a broken one. Before 1.5.0 the limit was a
  fixed 5 s, after which the stream was torn down.
- A fragmented message goes out whole or not at all: when one fragment is not taken, the rest of that message is
  `DROPPED` up to and including its final fragment.
- When wsapi refuses the handshake (the caller's role or tenant, a stream it does not open to this caller),
  `connect()` returns `False` at once, `dp.handshake_refused` holds wsapi's reason, the `refused_callback` is called
  with it, and the stream tries again after 5, 10, 20, 40 and then every 60 s. Before 1.5.0 the refusal looked like a
  timeout and the stream reconnected every 5 s for ever.

See [Data Plane](../architecture/dataplane.md) for the fabric-side model.

### `logstreamer`

A log stream is a separate WebSocket (`wss://host:port/api/logstreamer`) that delivers live
log lines from a target agent to your callback.

```python
ls = client.get_logstreamer(name=None, callback=None)
```

| Method | Signature | Description |
|---|---|---|
| `client.get_logstreamer(name=None, callback=None)` | `-> logstreamer` | Create or fetch a logstreamer. `callback(msg)` receives each log line. A name is auto-generated if omitted. |
| `connect()` | `-> bool` | Open the stream and block (up to 5 s) for activation. |
| `connected()` | `-> bool` | Whether the stream is active. |
| `update_config(dst_region, dst_agent)` | `-> None` | Point the log stream at an agent (default level). |
| `update_config_class(dst_region, dst_agent, loglevel, baseclass)` | `-> None` | Set the log level (e.g. `'Trace'`) for one base class (e.g. `'io.cresco.agent'`). |
| `send(message)` | `-> None` | Send a raw config/control message over the stream. |
| `close()` | `-> None` | Close the stream. |
| `client.close_logstreamer(name)` | `-> bool` | Close and deregister a named logstreamer on the client. |
| `client.get_active_logstreamers()` | `-> list[str]` | Names of currently registered logstreamers. |

---

## Worked examples

These mirror the runnable, standardized examples in
`examples/standard_examples.py`, which have byte-for-byte counterparts in the Java client
(`StandardExamples.java`). Set `HOST`, `PORT`, `SERVICE_KEY` and `FINGERPRINT` for your environment.

```python
from pycrescolib.clientlib import clientlib

HOST = "127.0.0.1"
PORT = 8282
SERVICE_KEY = "a-service-key"
FINGERPRINT = "sha384:AB:CD:..."   # from ws.fingerprint in the wsapi plugin data directory


def new_client():
    """Create a connected client, or None if the connection failed."""
    client = clientlib(HOST, PORT, SERVICE_KEY, fingerprint=FINGERPRINT)
    if not client.connect():
        print("Failed to connect")
        return None
    return client
```

### Connect and list the mesh

```python
def example_connect():
    """Connect, print API identity, list regions and agents, disconnect."""
    client = new_client()
    if client is None:
        return

    # Identity of the agent hosting the wsapi plugin.
    print("region:", client.api.get_api_region_name())
    print("agent :", client.api.get_api_agent_name())
    print("plugin:", client.api.get_api_plugin_name())

    # Identity of the global controller.
    print("global region:", client.api.get_global_region())
    print("global agent :", client.api.get_global_agent())

    # List the regions and the agents in the mesh.
    print("regions:", client.globalcontroller.get_region_list())
    print("agents :", client.globalcontroller.get_agent_list())

    client.close()
```

### Submit a pipeline (CADL) and poll status

```python
import time


def example_pipeline():
    """Submit a CADL pipeline, poll until active, fetch info/export, remove it."""
    client = new_client()
    if client is None:
        return

    # A minimal CADL pipeline.
    cadl = {"pipeline_id": "0", "pipeline_name": "example-pipeline",
            "nodes": [], "edges": []}

    reply = client.globalcontroller.submit_pipeline(cadl, "0")  # tenant 0
    pipeline_id = reply.get("gpipeline_id")
    print("submitted pipeline:", pipeline_id)

    # Poll until the pipeline is active (status code 10).
    for _ in range(30):
        status = client.globalcontroller.get_pipeline_status(pipeline_id)
        print("status:", status)
        if status == 10:
            break
        time.sleep(1)

    print("info  :", client.globalcontroller.get_pipeline_info(pipeline_id))
    print("export:", client.globalcontroller.get_pipeline_export(pipeline_id))

    client.globalcontroller.remove_pipeline(pipeline_id)
    client.close()
```

### Send a message / RPC

Use the `messaging` interface directly for actions not wrapped by a helper. This RPC lists the
regions, exactly like `globalcontroller.get_region_list()` does internally:

```python
def example_rpc():
    """Send an EXEC RPC to the global controller and read the reply dict."""
    client = new_client()
    if client is None:
        return

    reply = client.messaging.global_controller_msgevent(
        True,                        # is_rpc -> block for the reply
        "EXEC",                      # MsgEvent type
        {"action": "listregions"},   # payload
    )
    print("reply keys:", list(reply.keys()))

    # Fire-and-forget (is_rpc=False) returns None and does not wait.
    client.messaging.global_agent_msgevent(
        False, "CONFIG", {"action": "restartframework"},
        "my-region", "my-agent",
    )
    client.close()
```

### Manage a plugin on an agent

```python
def example_plugin_lifecycle(dst_region, dst_agent, jar_file_path):
    """Pull + add a plugin to an agent, inspect it, then remove it."""
    client = new_client()
    if client is None:
        return

    # Upload the plugin jar to the agent's repo, then add it as a plugin.
    print("repo pull:", client.agents.repo_pull_plugin_agent(
        dst_region, dst_agent, jar_file_path))
    configparams = {"pluginname": "example", "jarfile": jar_file_path}
    added = client.agents.add_plugin_agent(dst_region, dst_agent, configparams, None)
    plugin_id = added.get("pluginid")
    print("added plugin:", plugin_id)

    # Inspect the agent's plugins.
    print("plugins:", client.agents.list_plugin_agent(dst_region, dst_agent))
    print("status :", client.agents.status_plugin_agent(dst_region, dst_agent, plugin_id))

    client.agents.remove_plugin_agent(dst_region, dst_agent, plugin_id)
    client.close()
```

### Subscribe to a dataplane stream

```python
import time


def example_dataplane(stream_query):
    """Open a dataplane, receive via callback, send text + binary + fragmented data."""
    client = new_client()
    if client is None:
        return

    def on_message(message):
        print("dataplane recv:", message)

    dp = client.get_dataplane(stream_query, on_message)
    dp.connect()
    time.sleep(2)  # allow the stream to activate

    dp.send("hello dataplane")            # text frame
    dp.send_binary(b"\x00\x01\x02\x03")   # binary frame
    dp.send_partial(b"part-1", False)     # append a fragment
    dp.send_partial(b"part-2", True)      # flush the fragmented message
    time.sleep(2)

    print("metrics:", dp.get_metrics())
    client.close_dataplane(stream_query)
    client.close()
```

### Stream logs from an agent

```python
import time


def example_logstreamer(dst_region, dst_agent):
    """Attach a log streamer, set the config, raise one base class to Trace."""
    client = new_client()
    if client is None:
        return

    def on_message(message):
        print("log:", message)

    ls = client.get_logstreamer("example", on_message)
    ls.connect()
    time.sleep(2)

    ls.update_config(dst_region, dst_agent)
    ls.update_config_class(dst_region, dst_agent, "Trace", "io.cresco.agent")
    time.sleep(5)

    client.close_logstreamer("example")
    client.close()
```

### Restart an agent (guarded)

```python
def example_admin(dst_region, dst_agent):
    """Check the target controller, then restart its framework (guarded)."""
    client = new_client()
    if client is None:
        return

    if client.agents.is_controller_active(dst_region, dst_agent):
        print("controller status:", client.agents.get_controller_status(dst_region, dst_agent))
        client.admin.restartframework(dst_region, dst_agent)
    else:
        print("controller not active on", dst_region, "/", dst_agent)

    client.close()
```

---

## Cross-language parity and testing

The clients ship a self-contained conformance suite (no live mesh required) that asserts they
emit **identical wire messages for identical API calls**, sharing one generated golden corpus.

- **Python:** `pip install -e .[test] && pytest`
- **Java:** `mvn test` — see [Java Client (feature parity)](java.md).
- **C++ / Arduino:** the [C++ client](cpp.md) mirrors the same submodules, method names, and
  `example_*` set for ESP32-S3 edge nodes.

---

## See also

- [Client Libraries — Overview](overview.md)
- [Java Client (feature parity)](java.md)
- [C++ / Arduino Client (feature parity)](cpp.md)
- [MsgEvent — the message wire contract](../api/msgevent.md)
- [Plugin Actions (Capability Inventory)](../api/plugin-actions.md)
- [Data Plane architecture](../architecture/dataplane.md)
- [`wsapi` plugin](../plugins/wsapi.md)
