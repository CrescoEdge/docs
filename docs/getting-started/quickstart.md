# Quickstart

This walks through launching a Cresco fabric and connecting a client. It assumes you have
[built the agent jar](installation.md).

## The shared environment

The launch scripts source `run/cresco.env`, which fixes the values every node in one fabric must agree on:

| Variable | Purpose |
|----------|---------|
| `JAVA_HOME`, `PATH` | JDK 21 |
| `AGENT_JAR` | the staged `agent-1.3-SNAPSHOT.jar` |
| `CRESCO_DISCOVERY_SECRET_GLOBAL` / `_REGION` / `_AGENT` | join secrets — a node presents the secret for the tier it joins |
| `CRESCO_SERVICE_KEY` | wsapi's legacy API credential: a client presenting it acts as an `admin` of the agent's tenant (standard profile only; see [API authentication](../security/api-auth.md)) |
| `CRESCO_XMX` | JVM heap |

## 1. Start a global controller

A global controller is also its own region and agent, so a single one is a complete, self-contained fabric.

```bash
cd run
./run-global.sh
```

This launches `region=global-region agent=global-controller` with:

| Port | Service |
|------|---------|
| `32005` | TCP discovery listener |
| `32010` | embedded ActiveMQ broker |
| `8282`  | **[wsapi](../plugins/wsapi.md)** — the secure WebSocket endpoint clients connect to (`wss://host:8282`) |
| `8080`  | Felix web console, only with `enable_console=true` (bound to loopback by default) |

Under the hood it sets `-Dis_global=true -Denable_wsapi=true` plus the discovery secrets and service key.

## 2. Attach a region (optional)

A regional controller runs its own broker and **federates up** to the global via a broker-to-broker bridge.
On the same host it uses non-default ports to avoid clashing with the global:

```bash
./run-region.sh          # region=edge-region agent=edge-controller -> global@127.0.0.1
```

Key flags: `-Dis_region=true -Dglobal_controller_host=127.0.0.1 -Dnetdiscoveryport=32015 -Dbroker_port=32020`.
The agents in this region connect to *its* broker (`32020`), and the region bridges to the global.

## 3. Attach a plain agent (optional)

A plain agent runs no broker; it connects **directly** to a controller's broker.

```bash
./run-agent.sh           # region=global-region agent=agent-001 -> regional controller@127.0.0.1
```

Key flags: `-Dis_agent=true -Dregional_controller_host=127.0.0.1 -Dregional_controller_port=32005`.

!!! tip "Two-node helper"
    `run/launch_2node.sh` brings up a global + a second node in one step for quick testing.

## 4. Connect a client

Clients connect to the **global controller's wsapi** at `wss://host:8282`. Both clients verify wsapi's TLS
certificate by default, so give them its fingerprint: wsapi writes it to `ws.fingerprint` in its plugin data directory
and logs it at every start. The quickstart authenticates with the service key; users, API tokens and client
certificates are in [API authentication](../security/api-auth.md).

=== "Python"
    ```python
    from pycrescolib.clientlib import clientlib

    client = clientlib("localhost", 8282, "<CRESCO_SERVICE_KEY>", fingerprint="sha384:AB:CD:...")
    if client.connect():
        # list the mesh
        print(client.globalcontroller.get_agent_list())
        client.close()
    ```

=== "Java"
    ```java
    CrescoClient client = CrescoClient.builder("localhost", 8282)
            .serviceKey("<CRESCO_SERVICE_KEY>")
            .pinnedFingerprint("sha384:AB:CD:...")
            .build();
    if (client.connect()) {
        System.out.println(client.globalcontroller.get_agent_list(null));
        client.close();
    }
    ```

See the [Java](../clients/java.md) and [Python](../clients/python.md) client references for the full API,
and [Plugin Actions](../api/plugin-actions.md) for everything you can invoke.

## 5. Shut down and reset

```bash
./stop.sh                       # or: pkill -9 -f agent-1.3-SNAPSHOT.jar
rm -rf run/cresco-data run/nodes   # wipe state before the next run
```

## Where to go next

- **[Configuration](configuration.md)** — the flags that shape a node, including enabling security.
- **[Architecture Overview](../architecture/overview.md)** — how the mesh actually works.
- **[Operations › Deployment](../operations/deployment.md)** — multi-host and secured deployments.
