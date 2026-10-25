# C++ / Arduino Client (`cppcrescolib`)

`cppcrescolib` is the official **C++/Arduino** SDK for driving a Cresco fabric from an
embedded edge node. It opens a single authenticated, secure WebSocket control connection to
an agent running the [`wsapi`](../plugins/wsapi.md) plugin (`wss://host:8282`, authenticated
with the `cresco_service_key` header) and exposes the same submodule handles as the Python
and Java clients — so you can inspect the mesh, deploy plugins, submit pipelines, build
stunnel tunnels, and open separate stream sockets for the data plane and for live logs, all
from a microcontroller.

It is targeted at **ESP32 (ESP32-S3)** class devices: the reference node is a Heltec
ESP32-S3 that scans the RF spectrum and reports sweeps over a Cresco data plane.

The C++, Python, and Java clients are kept **feature- and name-identical**: the same
submodules, the same method names (snake_case in all three languages), the same wire
messages, and the same worked examples. Anything you can do in one you can do in the others
with the same call — see [Python Client](python.md) and [Java Client](java.md).

!!! info "Single-threaded `loop()` model"
    Unlike the Python client (async under a blocking surface) or the Java client (background
    reader threads), `cppcrescolib` runs on a **single cooperative task**. You call
    `client.loop()` from the sketch's `loop()`; it pumps the control socket **and** every
    registered stream — incoming frames, ping/pong, and reconnect. **RPC calls block this
    same task**: they spin the transport loop internally until the reply arrives or the
    timeout elapses. There is no separate event loop or thread to manage.

---

## Installation

`cppcrescolib` is a [PlatformIO](https://platformio.org/) library. Add it to your
`lib_deps`, or symlink the repo into `lib/`:

```ini
; platformio.ini
[env:esp32-s3]
platform  = espressif32
board     = esp32-s3-devkitc-1
framework = arduino
lib_deps  =
    https://github.com/CrescoEdge/cppcrescolib
    bblanchon/ArduinoJson @ ^7.0.0
    links2004/WebSockets  @ ^2.7.0
```

### Requirements

- **Platform:** `espressif32` (the ESP32 Arduino core supplies `MD5Builder` and mbedTLS for
  the TLS handshake). ESP32-S3 is the reference target.
- **Dependencies** (resolved automatically by PlatformIO's `deep+` LDF):

    | Dependency | Purpose |
    |---|---|
    | [`ArduinoJson`](https://arduinojson.org/) (`^7`) | Structured message bodies and reply documents (`JsonDocument`). |
    | [`arduinoWebSockets`](https://github.com/Links2004/arduinoWebSockets) (`^2.7`) | The `wss://` transport for the control connection and every stream. |
    | `miniz` | gzip for the `compress_param` / `decompress_param` message options (bundled). |

- A reachable Cresco agent hosting the [`wsapi`](../plugins/wsapi.md) plugin and a valid
  `cresco_service_key`.

---

## Connecting

```cpp
#include <WiFi.h>
#include <cppcrescolib.h>
using namespace cresco;

CrescoClient client("host", 8282, "your-service-key", /*verify_ssl=*/false);

void setup() {
    Serial.begin(115200);
    WiFi.begin("SSID", "PASS");
    while (WiFi.status() != WL_CONNECTED) delay(400);

    if (client.connect()) {                       // blocks until connected
        Serial.println(client.api.get_api_region_name());
        serializeJson(client.globalcontroller.get_agent_list(), Serial);
    }
}

void loop() {
    client.loop();                                // pump control + all streams
}
```

Bring up Wi-Fi yourself (the library does not manage the radio), then `connect()`. The
client opens one authenticated control connection to the `wsapi` endpoint; identity (region,
agent, plugin) is read from the negotiated TLS peer certificate, exactly as with the other
clients.

`verify_ssl` defaults to `false` (self-signed mesh certs on constrained devices); set it
`true` when the device is provisioned to validate the peer chain.

### Constructor & lifecycle

| Method | Description |
|---|---|
| `CrescoClient(host, port, service_key, verify_ssl=false)` | Create the client for a `wsapi` endpoint. |
| `connect(block_on_connect=true) -> bool` | Connect; block until connected when `block_on_connect`. |
| `connected() -> bool` | True if the control connection is up. |
| `close() -> bool` | Close the control connection **and every managed stream**. |
| `connection() -> WsTransport*` | The underlying websocket transport. |
| `loop()` | Pump the control socket and all registered streams. **Call every `loop()`.** |

### Stream management

Data-plane and log-streamer channels are *separate* WebSocket connections, tracked by name
on the client and auto-reconnecting if the socket drops.

| Method | Description |
|---|---|
| `get_dataplane(stream_name, text_cb=nullptr, binary_cb=nullptr) -> dataplane*` | Open & register a data plane for a stream query. |
| `close_dataplane(stream_name) -> bool` | Close & deregister a named data plane. |
| `get_active_dataplanes() -> std::vector<String>` | Names of registered data planes. |
| `get_logstreamer(name="", callback=nullptr) -> logstreamer*` | Open & register a log streamer. |
| `close_logstreamer(name) -> bool` | Close & deregister a named log streamer. |
| `get_active_logstreamers() -> std::vector<String>` | Names of registered log streamers. |

---

## Submodule handles

The connected client exposes the same five submodules as the Python and Java clients.
Structured returns are `ArduinoJson::JsonDocument` (a flat string→string object for dicts, an
array for lists); primitives (`bool`, `int`, `String`) where the spec says so.

### `api`

Local and global identity lookups.

| Method | Description |
|---|---|
| `get_api_region_name()` | Region of the agent hosting the `wsapi` plugin. |
| `get_api_agent_name()` | Agent hosting the `wsapi` plugin. |
| `get_api_plugin_name()` | The `wsapi` plugin name. |
| `get_global_region()` | Global controller region. |
| `get_global_agent()` | Global controller agent. |
| `get_global_info()` | Reply doc carrying `global_region` / `global_agent`. |

### `admin`

Node lifecycle on a target agent.

| Method | Description |
|---|---|
| `stopcontroller(dst_region, dst_agent)` | Stop the agent's controller plugin. |
| `restartcontroller(dst_region, dst_agent)` | Restart the controller plugin. |
| `restartframework(dst_region, dst_agent)` | Restart the OSGi framework. |
| `killjvm(dst_region, dst_agent)` | Force-terminate the agent JVM. |

### `agents`

Per-agent operations: plugin lifecycle, uploads, logs, discovery, CEP.

| Method | Description |
|---|---|
| `is_controller_active(dst_region, dst_agent) -> bool` | Whether the target controller is active. |
| `get_controller_status(dst_region, dst_agent)` | Controller status. |
| `add_plugin_agent(dst_region, dst_agent, configparams, edges=null)` | Add a plugin to an agent. |
| `remove_plugin_agent(dst_region, dst_agent, plugin_id)` | Remove a plugin. |
| `list_plugin_agent(dst_region, dst_agent)` | List plugins on an agent (JSON array). |
| `status_plugin_agent(dst_region, dst_agent, plugin_id)` | Plugin status. |
| `get_agent_info(dst_region, dst_agent)` | Agent information. |
| `get_agent_log(dst_region, dst_agent)` | Agent logs. |
| `repo_pull_plugin_agent(dst_region, dst_agent, jar_file_path)` | Pull a plugin jar to the agent repo. |
| `upload_plugin_agent(dst_region, dst_agent, jar_file_path)` | Upload a plugin jar. |
| `update_plugin_agent(dst_region, dst_agent, jar_file_path)` | Update a plugin. |
| `get_broadcast_discovery(dst_region, dst_agent)` | Broadcast discovery from an agent. |
| `cepadd(input_stream, input_stream_desc, output_stream, output_stream_desc, query, dst_region, dst_agent)` | Add a complex-event-processing query. |

### `globalcontroller`

Mesh-wide inventory and application operations.

| Method | Description |
|---|---|
| `submit_pipeline(cadl, tenant_id="0")` | Submit a CADL pipeline. |
| `remove_pipeline(pipeline_id)` | Remove a pipeline. |
| `get_pipeline_list()` | List pipelines (JSON array). |
| `get_pipeline_info(pipeline_id)` | Pipeline information. |
| `get_pipeline_status(pipeline_id) -> int` | Pipeline status code. |
| `get_pipeline_id_by_name(pipeline_name, out_id) -> bool` | Look up a pipeline ID by name. |
| `get_pipeline_export(pipeline_id)` | Export a pipeline. |
| `get_pipeline_is_assignment_info(inode_id, resource_id)` | inode→resource assignment info. |
| `get_agent_list(dst_region="")` | List agents (optionally in one region). |
| `get_agent_resources(dst_region, dst_agent)` | An agent's resource info. |
| `get_region_resources(dst_region)` | A region's resource info. |
| `get_region_list()` | List regions. |
| `get_plugin_repo_list()` | Plugins available in the repositories. |
| `get_repo_plugins()` | Plugins known to the repositories. |
| `upload_plugin_global(jar_file_path)` | Upload a plugin jar to the global controller's repository. |
| `get_metric_inventory(scope="global", dst_region="", dst_agent="", include_plugins=true, include_resource=true, timeout_ms=45000)` | Unified metric inventory. `scope` is `node`/`region`/`global`; with `dst_region`+`dst_agent` set it targets that agent's controller (node scope), otherwise the global controller. |
| `get_capability_inventory(scope="global", dst_region="", dst_agent="", include_plugins=true, include_osgi=false, timeout_ms=45000)` | Unified capability inventory (self-describing actions), scoped the same way as `get_metric_inventory`. |

### `messaging`

Low-level `MsgEvent` routing (RPC and fire-and-forget) to controllers, agents, and plugins —
the same primitive the other submodules are built on. Messaging calls take an `is_rpc` flag:
when true the call **blocks** (pumping the transport loop until the reply arrives or the
timeout elapses); when false it is sent without waiting.

---

## Worked examples

The runnable sketches in the repo's [`examples/`](https://github.com/CrescoEdge/cppcrescolib/tree/1.3/examples)
are identical in shape to the Python/Java `example_*` set: `example_connect`,
`example_pipeline`, `example_plugin_lifecycle`, `example_dataplane`, `example_logstreamer`,
`example_admin`, plus `esp_spectrum_reporter`.

### Connect and list the mesh

```cpp
#include <WiFi.h>
#include <cppcrescolib.h>
using namespace cresco;

CrescoClient client("host", 8282, "your-service-key", /*verify_ssl=*/false);

void setup() {
    Serial.begin(115200);
    WiFi.begin("SSID", "PASS");
    while (WiFi.status() != WL_CONNECTED) delay(400);

    if (client.connect()) {
        Serial.printf("api region=%s agent=%s plugin=%s\n",
                      client.api.get_api_region_name().c_str(),
                      client.api.get_api_agent_name().c_str(),
                      client.api.get_api_plugin_name().c_str());

        JsonDocument regions = client.globalcontroller.get_region_list();
        serializeJson(regions, Serial); Serial.println();

        JsonDocument agents = client.globalcontroller.get_agent_list();
        serializeJson(agents, Serial);  Serial.println();
    }
}

void loop() { client.loop(); }
```

### Subscribe to a data-plane stream

Data-plane channels are separate auto-reconnecting sockets, delivered to callbacks. Register
the stream in `setup()`, then keep `client.loop()` running so frames are pumped:

```cpp
CrescoClient client("host", 8282, "your-service-key");
dataplane* dp = nullptr;

void on_text(const String& msg)                        { Serial.printf("[dp text] %s\n", msg.c_str()); }
void on_binary(const uint8_t* data, size_t len)        { Serial.printf("[dp bin] %u bytes\n", (unsigned)len); }

void setup() {
    Serial.begin(115200);
    WiFi.begin("SSID", "PASS");
    while (WiFi.status() != WL_CONNECTED) delay(400);
    if (!client.connect()) return;

    dp = client.get_dataplane("region0_agent0", on_text, on_binary);
    if (dp->connect()) {
        dp->update_config("region0", "agent0");
        dp->send("hello text");
        uint8_t bin[4] = {1, 2, 3, 4};
        dp->send_binary(bin, 4);
    }
}

void loop() { client.loop(); }   // frames arrive on the callbacks from here
```

### Submit a pipeline and poll status

```cpp
// cadl is a serialized CADL document (String / JsonDocument)
JsonDocument res = client.globalcontroller.submit_pipeline(cadl, /*tenant_id=*/"0");
String pid;
if (client.globalcontroller.get_pipeline_id_by_name("my-pipeline", pid)) {
    int status = client.globalcontroller.get_pipeline_status(pid);
    Serial.printf("pipeline %s status=%d\n", pid.c_str(), status);
}
```

### The reference node — `esp_spectrum_reporter`

`esp_spectrum_reporter` is the flagship example: a Heltec ESP32-S3 RF spectrum scanner that
connects to the mesh, opens a data plane, and streams RF sweep frames to a Cresco stream —
a complete embedded producer feeding the fabric with the same client every other language
uses.

---

## Cross-language parity and testing

`cppcrescolib` is validated against the **same** behavioral contract as the Python and Java
clients: identical submodules, identical method names (snake_case), identical wire messages,
and the same `example_*` set. Porting a control script between the three languages is a
mechanical translation — the calls and their arguments are the same.

The differences are only at the edges the platform forces:

| | Python (`pycrescolib`) | Java (`clientlib`) | C++ (`cppcrescolib`) |
|---|---|---|---|
| Concurrency | asyncio under a blocking surface | background reader threads | single cooperative `loop()` |
| Structured returns | `dict` / `list` | `Map` / `List` / JSON | `ArduinoJson::JsonDocument` |
| Streams | separate auto-reconnecting sockets | separate auto-reconnecting sockets | separate auto-reconnecting sockets |
| Target | server / desktop | server / desktop / JVM | ESP32-S3 edge node |

## See also

- [Client Libraries overview](overview.md) — the shared model across all three SDKs
- [Python Client](python.md) · [Java Client](java.md) — the equivalent SDKs
- [`wsapi` plugin](../plugins/wsapi.md) — the endpoint every client connects to
- [Data Plane](../architecture/dataplane.md) · [Messaging & Routing](../architecture/messaging.md)
