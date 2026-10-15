# Java Client (`clientlib`)

`clientlib` is the external Java SDK for Cresco. It connects to an agent running
the **`wsapi`** plugin over a secure WebSocket (`wss://host:8282`, authenticated
with the `cresco_service_key` header) and drives the fabric: controlling agents,
deploying plugins, submitting pipelines, streaming the data plane, and tailing
logs.

For the shared connect-then-use-submodules model and the Java/Python/C++ parity
guarantee, see the [Client Libraries overview](overview.md). The equivalent SDKs
are the [Python client](python.md) and the [C++ / Arduino client](cpp.md).

## Installation

The client is published as `io.cresco:clientlib`. Add it to a Maven project:

```xml
<dependency>
    <groupId>io.cresco</groupId>
    <artifactId>clientlib</artifactId>
    <version>1.3-SNAPSHOT</version>
</dependency>
```

To build from source, run `mvn package` in `code/clientlib`. The build produces
a shaded, self-contained jar at `target/clientlib-1.3-SNAPSHOT.jar`.

### Requirements

- **JDK 21.**
- Network reachability to an agent hosting the `wsapi` plugin (default port
  `8282`).
- The `wsapi` **service key** configured on that agent.

The transport is built on Netty (TLS + WebSocket). By default it uses the JDK
TLS provider; set `-Dcresco.client.ssl_provider=OPENSSL` to use native
BoringSSL (via `netty-tcnative`) when the native library is present.

## Connecting

The entry point is `crescoclient.CrescoClient`.

```java
import crescoclient.CrescoClient;

// host/ip of an agent running wsapi, its port, the wsapi service key,
// and whether to verify the TLS certificate.
CrescoClient client = new CrescoClient("localhost", 8282, "your-service-key", false);

if (client.connect()) {
    System.out.println(client.api.get_api_region_name()
        + " " + client.api.get_api_agent_name());
    System.out.println(client.globalcontroller.get_agent_list(null));
    client.close();
}
```

### Constructors

| Constructor | Description |
|---|---|
| `CrescoClient(String host, int port, String service_key)` | Create a client with SSL verification disabled. |
| `CrescoClient(String host, int port, String service_key, boolean verify_ssl)` | Create a client, choosing whether to verify the TLS certificate. |

Cresco agents use self-signed certificates, so `verify_ssl` defaults to
`false`. The `host` and `port` identify the agent running the `wsapi` plugin;
`service_key` is sent as the `cresco_service_key` header on the WebSocket
upgrade.

### Lifecycle methods

| Method | Description |
|---|---|
| `boolean connect()` | Connect to the `wsapi` endpoint, blocking until connected (or the connect timeout elapses). Returns `true` on success. |
| `boolean connect(boolean block_on_connect)` | Connect; block until connected only when `block_on_connect` is `true`. |
| `boolean connected()` | Return `true` if currently connected to the `wsapi` endpoint. |
| `boolean close()` | Close the control connection and all managed streams. |
| `MsgEventInterface connection()` | Return the underlying WebSocket transport interface. |
| `boolean verify_ssl()` | Whether SSL certificate verification is enabled for this client. |

`connect()` throws `InterruptedException`. The default connect timeout is 5
seconds; the RPC reply timeout is 30 seconds.

### Stream management

The client tracks data-plane and log-streamer connections by name so they can be
closed and enumerated centrally.

| Method | Description |
|---|---|
| `DataPlaneInterface get_dataplane(String stream_name)` | Get (and register) a data plane for the given stream query. |
| `DataPlaneInterface get_dataplane(String stream_name, OnMessageCallback cb)` | As above, delivering received messages to `cb`. |
| `boolean close_dataplane(String stream_name)` | Close and deregister the named data plane. |
| `List<String> get_active_dataplanes()` | Names of currently registered data planes. |
| `LogStreamerInterface get_logstreamer()` | Get (and register) a log streamer with an auto-generated name. |
| `LogStreamerInterface get_logstreamer(OnMessageCallback cb)` | As above, delivering received log lines to `cb`. |
| `LogStreamerInterface get_logstreamer(String name, OnMessageCallback cb)` | Get (and register) a log streamer under `name`. |
| `boolean close_logstreamer(String name)` | Close and deregister the named log streamer. |
| `List<String> get_active_logstreamers()` | Names of currently registered log streamers. |

## Submodule handles

A connected client exposes six submodule fields — `api`, `admin`, `agents`,
`globalcontroller`, `messaging`, and `stunnel` — documented below. Every public
class and method also carries Javadoc (see [API reference (Javadoc)](#api-reference-javadoc)).

### `api`

Identity lookups for the local `wsapi` host and the global controller. Local
identity is read from the negotiated TLS peer certificate; global identity is
resolved with a one-time `globalinfo` RPC.

| Method | Signature | Description |
|---|---|---|
| `get_api_region_name` | `String get_api_region_name()` | Region name of the agent hosting `wsapi`. |
| `get_api_agent_name` | `String get_api_agent_name()` | Agent name of the agent hosting `wsapi`. |
| `get_api_plugin_name` | `String get_api_plugin_name()` | Plugin name of the `wsapi` plugin. |
| `get_global_region` | `String get_global_region()` | Region of the global controller. |
| `get_global_agent` | `String get_global_agent()` | Agent of the global controller. |
| `get_global_info` | `Map.Entry<String,String> get_global_info()` | Pair of `(global_region, global_agent)`: `getKey()` is the region, `getValue()` the agent. |

### `admin`

Node lifecycle actions on a target agent. These are fire-and-forget `CONFIG`
messages routed through the global controller.

| Method | Signature | Description |
|---|---|---|
| `stopcontroller` | `void stopcontroller(String dst_region, String dst_agent)` | Stop the agent's controller plugin. |
| `restartcontroller` | `void restartcontroller(String dst_region, String dst_agent)` | Restart the agent's controller plugin. |
| `restartframework` | `void restartframework(String dst_region, String dst_agent)` | Restart the underlying OSGi framework. |
| `killjvm` | `void killjvm(String dst_region, String dst_agent)` | Force-terminate the agent JVM. |

### `agents`

Per-agent operations: controller status, plugin lifecycle, uploads, logs, and
complex-event-processing queries. All are routed to the target agent through the
global controller.

| Method | Signature | Description |
|---|---|---|
| `is_controller_active` | `boolean is_controller_active(String dst_region, String dst_agent)` | Whether the target controller is active. |
| `get_controller_status` | `String get_controller_status(String dst_region, String dst_agent)` | Controller status string. |
| `add_plugin_agent` | `Map<String,String> add_plugin_agent(String dst_region, String dst_agent, Map<String,String> configparams, Map<String,String> edges)` | Add a plugin to an agent (`edges` may be `null`). |
| `remove_plugin_agent` | `Map<String,String> remove_plugin_agent(String dst_region, String dst_agent, String plugin_id)` | Remove a plugin from an agent. |
| `list_plugin_agent` | `List<Map<String,String>> list_plugin_agent(String dst_region, String dst_agent)` | List plugins on an agent. |
| `status_plugin_agent` | `Map<String,String> status_plugin_agent(String dst_region, String dst_agent, String plugin_id)` | Get a plugin's status. |
| `get_agent_info` | `Map<String,String> get_agent_info(String dst_region, String dst_agent)` | Get agent information. |
| `get_agent_log` | `Map<String,String> get_agent_log(String dst_region, String dst_agent)` | Get agent logs. |
| `repo_pull_plugin_agent` | `Map<String,String> repo_pull_plugin_agent(String dst_region, String dst_agent, String jar_file_path)` | Pull a plugin jar into the agent's repo (by manifest metadata). |
| `upload_plugin_agent` | `Map<String,String> upload_plugin_agent(String dst_region, String dst_agent, String jar_file_path)` | Upload a plugin jar to the agent. |
| `update_plugin_agent` | `void update_plugin_agent(String dst_region, String dst_agent, String jar_file_path)` | Update a plugin on the agent. |
| `get_broadcast_discovery` | `Map<String,String> get_broadcast_discovery(String dst_region, String dst_agent)` | Get broadcast discovery from an agent. |
| `cepadd` | `Map<String,String> cepadd(String input_stream, String input_stream_desc, String output_stream, String output_stream_desc, String query, String dst_region, String dst_agent)` | Add a complex-event-processing query on an agent. |

### `globalcontroller`

Mesh-wide inventory and application operations, routed to the global controller.

| Method | Signature | Description |
|---|---|---|
| `submit_pipeline` | `Map<String,String> submit_pipeline(Map<String,Object> cadl, String tenantId)` | Submit a CADL pipeline (also accepts a JSON string). Returns the reply, e.g. `gpipeline_id`. |
| `remove_pipeline` | `boolean remove_pipeline(String pipeline_id)` | Remove a pipeline. |
| `get_pipeline_list` | `List<Map<String,String>> get_pipeline_list()` | List pipelines. |
| `get_pipeline_info` | `gPayload get_pipeline_info(String pipeline_id)` | Get full pipeline info as a `gPayload`. |
| `get_pipeline_status` | `int get_pipeline_status(String pipeline_id)` | Get a pipeline's status code (`10` = active). |
| `get_pipeline_id_by_name` | `String get_pipeline_id_by_name(String pipeline_name)` | Look up a pipeline ID by name. |
| `get_pipeline_export` | `Map<String,String> get_pipeline_export(String pipeline_id)` | Export a pipeline. |
| `get_pipeline_is_assignment_info` | `Map<String,String> get_pipeline_is_assignment_info(String inode_id, String resource_id)` | Get inode-to-resource assignment info. |
| `get_agent_list` | `Map<String,List<Map<String,String>>> get_agent_list(String dst_region)` | List agents (optionally in one region; pass `null` for all). |
| `get_agent_resources` | `Map<String,List<Map<String,String>>> get_agent_resources(String dst_region, String dst_agent)` | Get an agent's resource info. |
| `get_region_resources` | `Map<String,List<Map<String,String>>> get_region_resources(String dst_region)` | Get a region's resource info. |
| `get_region_list` | `Map<String,List<Map<String,String>>> get_region_list()` | List regions. |
| `get_plugin_repo_list` | `Map<String,List<Map<String,String>>> get_plugin_repo_list()` | List plugins available in the repositories. |
| `get_repo_plugins` | `Map<String,List<Map<String,String>>> get_repo_plugins()` | List plugins known to the repo plugin. |
| `upload_plugin_global` | `Map<String,String> upload_plugin_global(String jar_file_path)` | Upload a plugin jar to the global repo. |
| `get_metric_inventory` | `String get_metric_inventory(String dst_region, String dst_agent, String scope, boolean include_plugins, boolean include_resource)` | Pull the fabric's unified metric inventory as JSON. |
| `get_capability_inventory` | `String get_capability_inventory(String scope, boolean include_plugins, boolean include_osgi)` | Pull the fabric's self-describing capability catalog (LLM tool descriptors) as JSON. |

!!! note "Inventory scope and targeting"
    `scope` is one of `"node"`, `"region"`, or `"global"`. For
    `get_metric_inventory`, passing both `dst_region` and `dst_agent` targets a
    single agent's controller directly (node scope); otherwise the query goes to
    the global controller. No-argument convenience overloads exist:
    `get_metric_inventory()` (whole mesh, plugins + resource on) and
    `get_capability_inventory()` (whole mesh, plugins on, MsgEvent actions only).
    Only MsgEvent actions in the capability inventory are callable tools; the
    optional OSGi surface is informational.

### `messaging`

The low-level MsgEvent layer that every other submodule builds on. Each method
takes `(boolean is_rpc, String message_event_type, Map<String,Object>
message_payload, ...)`. When `is_rpc` is `true` the call blocks and returns the
reply `Map<String,String>`; when `false` it is sent without waiting and returns
`null`. `message_event_type` is typically `"EXEC"` or `"CONFIG"`.

| Method | Routes to |
|---|---|
| `global_controller_msgevent(is_rpc, type, payload)` | the global controller |
| `regional_controller_msgevent(is_rpc, type, payload)` | the regional controller |
| `global_agent_msgevent(is_rpc, type, payload, dst_region, dst_agent)` | an agent via the global controller |
| `regional_agent_msgevent(is_rpc, type, payload, dst_agent)` | an agent in the local region |
| `agent_msgevent(is_rpc, type, payload)` | the local agent |
| `global_plugin_msgevent(is_rpc, type, payload, dst_region, dst_agent, dst_plugin)` | a plugin via the global controller |
| `regional_plugin_msgevent(is_rpc, type, payload, dst_agent, dst_plugin)` | a plugin in the local region |
| `plugin_msgevent(is_rpc, type, payload, dst_plugin)` | a local plugin |

The submodule also provides (de)compression and JSON helpers used to marshal
large parameters into the envelope: `setCompressedParam` / `getCompressedParam`,
`setCompressedDataParam` / `getCompressedDataParam`, and
`getMapFromString` / `getListMapFromString` / `getMapListMapFromString`.

See [MsgEvent](../api/msgevent.md) for the envelope structure and routing, and
[Plugin Actions](../api/plugin-actions.md) for the action names available in
`message_payload`.

### `stunnel`

Secure TCP tunnels through the [stunnel plugin](../plugins/stunnel.md), managed
entirely from the client: no CADL pipeline and no hand-built `configsrctunnel`
messages. A tunnel listens on `src_port` at the source agent and forwards every
accepted connection over the data plane to `dst_host:dst_port` as seen from the
destination agent. Plugin ids are resolved by name; every method with a trailing
`plugin_id` argument also accepts `null` to resolve it.

| Method | Plugin action | Description |
|---|---|---|
| `String find_plugin(region, agent)` | — | The `io.cresco.stunnel` plugin id on an agent, or `null`. |
| `String deploy_plugin(region, agent)` / `deploy_plugin(region, agent, jar_file_path)` | `pluginadd` (+ `pluginupload`) | Ensure the plugin runs on an agent: no-op if loaded, else start the copy embedded in the agent (or upload and start a local jar). Returns the plugin id. |
| `Map<String,String> create_tunnel(stunnel_id, src_region, src_agent, src_port, dst_region, dst_agent, dst_host, dst_port, buffer_size)` | CONFIG `configsrctunnel` | Create a tunnel. Overloads add `src_plugin_id, dst_plugin_id` and then `watchdog_timeout, performance_report_rate` (ms). A `null` id generates a UUID. Returns the resolved tunnel config on success, the plugin reply (`status`, `status_desc`) on failure. |
| `List<Map<String,String>> get_tunnel_list(region, agent)` | EXEC `listtunnels` | `[{stunnel_id, status}]` for the tunnels on an agent. |
| `String get_tunnel_status(region, agent, stunnel_id)` | EXEC `gettunnelstatus` | `ACTIVE`, `RECOVERING`, `DOWN` or `UNKNOWN`; `null` if not configured there. |
| `Map<String,String> get_tunnel_config(region, agent, stunnel_id)` | EXEC `gettunnelconfig` | The persisted tunnel config. |
| `boolean tunnel_health_check(region, agent, stunnel_id)` | EXEC `tunnelhealthcheck` | Whether the tunnel is configured on that agent. |
| `Map<String,Object> get_metrics(region, agent)` | EXEC `getmetrics` | The plugin's live metrics. |
| `Map<String,String> set_net_tuning(region, agent, tuning)` | CONFIG `nettuning` | Plugin-wide socket tuning: `net_socket_buffer_bytes`, `net_read_chunk_bytes`, `net_write_high_water_bytes`, `net_fc_window_bytes`, `net_trace_sample_n`. |
| `Map<String,Object> remove_tunnel(stunnel_id, src_region, src_agent, dst_region, dst_agent)` | `removesrctunnel` + `removedsttunnel` | Remove both ends; returns `{stunnel_id, src_removal, dst_removal, fully_removed}`. |
| `remove_src_tunnel(region, agent, stunnel_id)` / `remove_dst_tunnel(region, agent, stunnel_id)` | CONFIG `removesrctunnel` / `removedsttunnel` | Remove one end (removing the source also cascades to the destination). |

Plugin replies carry `status` `"10"` (success), `"9"` (failure / not found),
`"400"` (bad request) or `"500"` (internal error), plus `status_desc`.

### Data plane — `DataPlaneInterface`

Returned by `client.get_dataplane(...)`. A data plane is its own WebSocket
connection (`/api/dataplane`); it sends its stream handshake on connect and delivers
received messages to the supplied `OnMessageCallback`.

The handshake (the `stream_name` argument) is either:

- a **JSON stream config** such as
  `{"ident_key":"stream_name","ident_id":"demo","io_type_key":"type","input_id":"in"}` —
  the stream receives messages whose `ident_key` property equals `ident_id`, and
  messages it sends are tagged with those properties; or
- a **selector** such as `stream_name='demo'` (`ident = 'value'` terms joined by
  `AND`, at least one equality) — receive only.

`wsapi` refuses selectors that could match other streams (`OR`, `IS NULL`, ...)
and closes the socket.

| Method | Signature | Description |
|---|---|---|
| `send` | `void send(String message)` | Send a text message (needs a JSON stream config handshake). |
| `send` / `send_binary` | `void send(ByteBuffer buf)` / `void send_binary(ByteBuffer buf)` | Send a binary message. |
| `send_binary_file` | `void send_binary_file(String file_path)` | Read a file and send it as a binary message. |
| `send_partial` | `void send_partial(ByteBuffer buf, boolean complete)` | Send a payload as a fragmented message (flushed when `complete` is `true`). |
| `update_config` | `void update_config(String dst_region, String dst_agent)` | Update the stream's target configuration. |
| `connect` / `start` | `void connect()` / `void start()` | Open the stream (and start the auto-reconnect monitor). |
| `connected` | `boolean connected()` | Whether the stream is connected. |
| `close` | `void close()` | Close the stream. |
| `get_metrics` | `Map<String,Object> get_metrics()` | Client-side counters for this stream (messages/bytes sent and received, active). |

### Log streamer — `LogStreamerInterface`

Returned by `client.get_logstreamer(...)`. A log streamer is its own WebSocket
connection (`/api/logstreamer`); received log lines are delivered to the
supplied `OnMessageCallback` (the default callback prints them to stdout).

| Method | Signature | Description |
|---|---|---|
| `send` | `void send(String message)` | Send a message over the log stream. |
| `update_config` | `void update_config(String dst_region, String dst_agent)` | Point the log stream at an agent (Trace, default base class). |
| `update_config_class` | `void update_config_class(String dst_region, String dst_agent, String loglevel, String baseclass)` | Set the log level for one base class. |
| `connect` / `start` | `void connect()` / `void start()` | Open the stream (and start the auto-reconnect monitor). |
| `connected` | `boolean connected()` | Whether the stream is connected. |
| `close` | `void close()` | Close the stream. |

### `OnMessageCallback`

Streams deliver received data through the `crescoclient.core.OnMessageCallback`
interface:

```java
public interface OnMessageCallback {
    void onMessage(String msg);
    void onMessage(byte[] b, int offset, int length);
}
```

## Worked examples

The following examples are drawn from
`example/standard/StandardExamples.java`; the connect, pipeline, plugin, data
plane, log and admin examples have byte-for-byte counterparts in the Python
client's `standard_examples.py`. Run one with

```bash
export CRESCO_HOST=localhost CRESCO_PORT=8282 CRESCO_SERVICE_KEY=...
java -cp target/clientlib-1.3-SNAPSHOT.jar example.standard.StandardExamples connect
# others: pipeline | plugin_lifecycle <region> <agent> <jar> | dataplane <stream> |
#         logstreamer <region> <agent> | admin <region> <agent> | messaging <region> <agent> |
#         stunnel <region> <agent> <src_port> <dst_host> <dst_port>
```

### Connect and list the mesh

```java
import crescoclient.CrescoClient;

CrescoClient client = new CrescoClient("127.0.0.1", 8282, "a-service-key", false);
if (!client.connect()) {
    System.out.println("Failed to connect");
    return;
}

// API identity of the agent hosting wsapi.
System.out.println("region: " + client.api.get_api_region_name());
System.out.println("agent : " + client.api.get_api_agent_name());
System.out.println("plugin: " + client.api.get_api_plugin_name());

// Global controller identity.
System.out.println("global region: " + client.api.get_global_region());
System.out.println("global agent : " + client.api.get_global_agent());

// List regions and agents.
System.out.println("regions: " + client.globalcontroller.get_region_list());
System.out.println("agents : " + client.globalcontroller.get_agent_list(null));

client.close();
```

### Submit a pipeline and poll its status

```java
import java.util.HashMap;
import java.util.Map;

// 1. Define a minimal CADL pipeline.
Map<String, Object> cadl = new HashMap<>();
cadl.put("pipeline_id", "0");
cadl.put("pipeline_name", "example-pipeline");

// 2. Submit for tenant 0.
Map<String, String> reply = client.globalcontroller.submit_pipeline(cadl, "0");
String pipeline_id = reply.get("gpipeline_id");
System.out.println("submitted pipeline: " + pipeline_id);

// 3. Poll until the pipeline is active (status code 10).
for (int i = 0; i < 30; i++) {
    int status = client.globalcontroller.get_pipeline_status(pipeline_id);
    if (status == 10) break;
    Thread.sleep(1000);
}

// 4. Fetch info and an export, then remove it.
System.out.println("info  : " + client.globalcontroller.get_pipeline_info(pipeline_id));
System.out.println("export: " + client.globalcontroller.get_pipeline_export(pipeline_id));
client.globalcontroller.remove_pipeline(pipeline_id);
```

### Plugin lifecycle on an agent

```java
import java.util.HashMap;
import java.util.Map;

// 1. Pull the jar into the agent repo, then add it as a plugin.
client.agents.repo_pull_plugin_agent(dst_region, dst_agent, jar_file_path);

Map<String, String> configparams = new HashMap<>();
configparams.put("pluginname", "example");
configparams.put("jarfile", jar_file_path);
Map<String, String> added =
    client.agents.add_plugin_agent(dst_region, dst_agent, configparams, null);
String plugin_id = added.get("pluginid");

// 2. List and check status.
System.out.println("plugins: " + client.agents.list_plugin_agent(dst_region, dst_agent));
System.out.println("status : " + client.agents.status_plugin_agent(dst_region, dst_agent, plugin_id));

// 3. Remove it.
client.agents.remove_plugin_agent(dst_region, dst_agent, plugin_id);
```

### Send a MsgEvent RPC

Every submodule call is ultimately a `messaging` call. To invoke a plugin action
directly, build the payload and issue an RPC. This mirrors how `api` resolves the
global controller location:

```java
import java.util.HashMap;
import java.util.Map;

Map<String, Object> payload = new HashMap<>();
payload.put("action", "globalinfo");

// Blocking RPC to the local wsapi plugin; returns the reply map.
Map<String, String> reply = client.messaging.plugin_msgevent(
        true,                                 // is_rpc
        "EXEC",                               // message_event_type
        payload,                              // message_payload
        client.api.get_api_plugin_name());    // dst_plugin

System.out.println("global_region=" + reply.get("global_region"));
System.out.println("global_agent =" + reply.get("global_agent"));
```

Pass `false` for `is_rpc` to send without waiting for a reply. See
[MsgEvent](../api/msgevent.md) and [Plugin Actions](../api/plugin-actions.md)
for available `message_event_type` values and action names.

### Subscribe to the data plane

```java
import crescoclient.core.OnMessageCallback;
import crescoclient.dataplane.DataPlaneInterface;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;

// 1. Open a data plane for a stream query, printing anything received.
OnMessageCallback onMessage = new OnMessageCallback() {
    @Override public void onMessage(String message) {
        System.out.println("dataplane recv: " + message);
    }
    @Override public void onMessage(byte[] b, int offset, int length) {
        System.out.println("dataplane recv: " + length + " bytes");
    }
};

String streamQuery =
    "{\"ident_key\":\"stream_name\",\"ident_id\":\"demo\",\"io_type_key\":\"type\",\"input_id\":\"in\"}";
DataPlaneInterface dp = client.get_dataplane(streamQuery, onMessage);
dp.connect();
Thread.sleep(2000);

// 2. Send text, binary, and a fragmented (partial) message.
dp.send("hello dataplane");
dp.send_binary(ByteBuffer.wrap(new byte[]{0, 1, 2, 3}));
dp.send_partial(ByteBuffer.wrap("part-1".getBytes(StandardCharsets.UTF_8)), false);
dp.send_partial(ByteBuffer.wrap("part-2".getBytes(StandardCharsets.UTF_8)), true);
Thread.sleep(2000);

// 3. Close the stream.
client.close_dataplane(streamQuery);
```

The handshake selects which data-plane traffic the stream carries (here, every
message tagged `stream_name=demo`, including this stream's own); it is also the
registry key used by `close_dataplane` / `get_active_dataplanes`. See
the [Data Plane architecture](../architecture/dataplane.md) for the query model.

### Stream logs

```java
import crescoclient.core.OnMessageCallback;
import crescoclient.logstreamer.LogStreamerInterface;

// 1. Attach a log streamer, printing anything received.
OnMessageCallback onMessage = new OnMessageCallback() {
    @Override public void onMessage(String message) {
        System.out.println("log: " + message);
    }
    @Override public void onMessage(byte[] b, int offset, int length) {
        System.out.println("log: " + length + " bytes");
    }
};

LogStreamerInterface ls = client.get_logstreamer("example", onMessage);
ls.connect();
Thread.sleep(2000);

// 2. Point the stream at an agent, then raise one base class to Trace.
ls.update_config(dst_region, dst_agent);
ls.update_config_class(dst_region, dst_agent, "Trace", "io.cresco.agent");
Thread.sleep(5000);

// 3. Close the stream.
client.close_logstreamer("example");
```

### Build a tunnel (no CADL)

```java
// 1. Make sure the stunnel plugin runs on both ends.
client.stunnel.deploy_plugin("global-region", "agent-a");
client.stunnel.deploy_plugin("global-region", "agent-b");

// 2. Listen on agent-a:2222, forward to localhost:22 as seen from agent-b.
Map<String, String> config = client.stunnel.create_tunnel("ssh",
        "global-region", "agent-a", "2222",
        "global-region", "agent-b", "localhost", "22", "65536");

// 3. Inspect it.
client.stunnel.get_tunnel_status("global-region", "agent-a", "ssh");  // ACTIVE
client.stunnel.get_tunnel_list("global-region", "agent-a");           // [{stunnel_id=ssh, status=ACTIVE}]
client.stunnel.get_tunnel_config("global-region", "agent-a", "ssh");

// 4. Remove both ends.
client.stunnel.remove_tunnel("ssh", "global-region", "agent-a", "global-region", "agent-b");
```

`example.Launcher` runs the same flow for a list of tunnels read from a JSON
config file (copy `config.example.json` to `config.json`, which is git-ignored,
and set `service_key`); with `"echo_test": true` it also pushes bytes through
each tunnel and checks they come back:

```bash
java -cp target/clientlib-1.3-SNAPSHOT.jar example.Launcher config.json
```

### Node administration

```java
// Only act if the target agent's controller is active.
if (client.agents.is_controller_active(dst_region, dst_agent)) {
    System.out.println("controller status: "
        + client.agents.get_controller_status(dst_region, dst_agent));

    // Restart the OSGi framework on the target agent.
    client.admin.restartframework(dst_region, dst_agent);
}
```

## API reference (Javadoc)

Every public class and method in `crescoclient` is documented with Javadoc
(parameters, return values, the wire action each call sends). To build the HTML
reference from a checkout:

```bash
mvn -q dependency:build-classpath -Dmdep.outputFile=target/cp.txt
javadoc -d target/apidocs -sourcepath src/main/java -subpackages crescoclient -cp "$(cat target/cp.txt)"
```

The repository README has the same usage guide in condensed form:
[CrescoEdge/clientlib](https://github.com/CrescoEdge/clientlib/tree/1.3#usage-guide).

## Testing and parity

The client ships a self-contained test suite (no live mesh required) that
asserts it emits **identical wire messages for identical API calls** as the
[Python](python.md) and [C++ / Arduino](cpp.md) clients, verified against a
shared golden corpus:

```bash
mvn test
```

A green run across the Java, Python, and C++ suites proves the clients produce
the same results. See the [Client Libraries overview](overview.md#feature-parity)
for the parity guarantee.
