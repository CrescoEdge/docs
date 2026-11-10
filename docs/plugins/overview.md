# Modules & Plugins Overview

Cresco is assembled from a fixed set of OSGi bundles (`io.cresco.*`) hosted inside a single [Apache Felix](https://felix.apache.org/) container. This page explains the module/plugin model — how the bundles are packaged, how the [agent](agent.md) uber-jar bootstraps them, and the distinction between the [controller](controller.md) (the fabric brain) and the functional plugins that run on top of it.

## The OSGi bundle model

Every Cresco module is an **OSGi bundle**: a JAR whose manifest carries a `Bundle-SymbolicName`, an `Export-Package` list, and (for most modules) an `Embed-Dependency` set that packages third-party libraries inside the bundle. The symbolic name of each module is `io.cresco.<artifactId>` (group id `io.cresco`).

!!! warning "The `bundle:bundle` build gotcha"
    Each module declares `<packaging>jar</packaging>`, but the artifact must be a real OSGi bundle or Felix cannot start it. The `maven-bundle-plugin` only rewrites the JAR when its goal is invoked **explicitly**:

    ```bash
    mvn clean package bundle:bundle      # NOT just "mvn install"
    ```

    A plain `mvn package`/`mvn install` produces a non-bundle JAR that the framework rejects. See [Installation & Build](../getting-started/installation.md).

Bundles fall into three roles:

| Role | Bundles | Loaded |
|------|---------|--------|
| **Host** | `agent` | The executable uber-jar; boots Felix and installs everything else. |
| **Framework services** | `logger`, `library`, `core` | Installed and started at boot, before the controller. |
| **The controller** | `controller` | The fabric brain — installed and started at boot. |
| **Functional plugins** | `repo`, `filerepo`, `sysinfo`, `wsapi`, `stunnel`, `executor` | Embedded in the uber-jar but installed **at runtime** by the controller, not at boot. |

## How the agent uber-jar bootstraps the bundles

The [agent](agent.md) is not itself an OSGi bundle — it is the Felix **host** jar with Main-Class `io.cresco.main.AgentEngine`. The component JARs are embedded **inside** the agent uber-jar under `src/main/resources/` and installed from the classpath by `HostApplication`. The boot sequence installs and starts a fixed ordered set:

```
configadmin → logger → (metatype, scr, gogo, healthcheck.api) → library → core → controller
                                          (+ optional Jetty web console when enable_console=true)
```

`healthcheck.api` is installed but **not** started at boot (the controller drives the Felix Health Check subsystem itself). Once the controller reaches `ACTIVE`, it brings up the fabric — the embedded [ActiveMQ](../architecture/messaging.md) broker, the embedded Apache Derby state database, TCP/UDP [discovery](../architecture/discovery.md), the [data plane](../architecture/dataplane.md), the [health](../architecture/health.md) subsystem — and only then loads the functional plugins. See [agent](agent.md) for the full boot detail.

## Controller vs. functional plugins

The **[controller](controller.md)** is fundamentally different from the other plugins. It implements the `AgentService` that every plugin depends on, owns the messaging fabric, the state machine, discovery, persistence, and the three role tiers (agent / region / global). It is the *host environment* that all functional plugins run inside.

A **functional plugin** is a much thinner thing: an OSGi `@Component` implementing `PluginService` (`scope=PROTOTYPE`, `configurationPolicy=REQUIRE`), holding a `@Reference AgentService`. Each registers an [`Executor`](library.md) that handles only the [`MsgEvent`](../api/msgevent.md) types it uses (all others return `null`). Plugins reach the mesh exclusively through the [`PluginBuilder`](library.md) API defined by [library](library.md).

### Plugin lifecycle

The controller's `StaticPluginLoader` (see [controller](controller.md)) waits for the controller to become active, then loads the enabled system plugins and restarts any persisted ones. Lifecycle is driven by `PluginAdmin`:

1. **Resolve** the plugin JAR (bundle / absolute path / local cache / embedded / remote [repo](repo.md)). A jar that is not embedded in the agent jar must verify against `plugin_signing_truststore` ([Signed plugins](../security/signed-plugins.md)); jars are named by a SHA-256 content hash (still carried in a field called `md5`).
2. **Install & start** the OSGi bundle.
3. **Register** a `PluginNode` (in-memory record + Derby row) and poll for the `PluginService` to appear.
4. **Run** — the plugin reports a status code; `10` = working, which the controller's `PluginHealthCheck` rolls into the `plugins` verdict. In addition, each functional plugin **registers its own `local`-tagged Felix Health Check** (executor/filerepo/repo/sysinfo/wsapi/stunnel), discovered by `CrescoHealthExecutor` alongside the controller's built-in checks and queryable via `gethealthinventory`. See [Health & State](../architecture/health.md).
5. **Stop** — the bundle is stopped and the `PluginNode` pruned.

Plugins are identified by a generated `pluginID` (e.g. `system-<UUID>` for system plugins) and addressed on the mesh via their region/agent/plugin tuple.

## Module summary

| Module | Bundle symbolic name | Purpose | Page |
|--------|----------------------|---------|------|
| agent | *(host jar — not a bundle)* | OSGi host bootstrap + uber-jar (Main-Class `io.cresco.main.AgentEngine`). | [agent](agent.md) |
| library | `io.cresco.library` | Shared plugin & controller API plus embedded runtime dependencies. | [library](library.md) |
| core | `io.cresco.core` | `CoreState` service — stop/restart/update the controller & JVM. | [core](core.md) |
| logger | `io.cresco.logger` | Logging bootstrap (Pax Logging / log4j2 via ConfigAdmin). | [logger](logger.md) |
| controller | `io.cresco.controller` | **The fabric brain** — state machine, messaging, discovery, persistence, health, roles. | [controller](controller.md) |
| repo | `io.cresco.repo` | Plugin/JAR repository — stores, reports, and serves signed plugin jars. | [repo](repo.md) |
| filerepo | `io.cresco.filerepo` | Distributed file/artifact repository — catalog + any-size transfer/streaming/sync. | [filerepo](filerepo.md) |
| sysinfo | `io.cresco.sysinfo` | Host metrics (OSHI) + on-demand SciMark2 CPU benchmark. | [sysinfo](sysinfo.md) |
| wsapi | `io.cresco.wsapi` | WebSocket/HTTPS API — external client entrypoint over `wss://…:8282`. | [wsapi](wsapi.md) |
| stunnel | `io.cresco.stunnel` | Netty TCP tunnels over the data plane. | [stunnel](stunnel.md) |
| executor | `io.cresco.executor` | Process/shell execution — dataplane-attached stdio + per-process metrics. | [executor](executor.md) |

## See also

- [Architecture Overview](../architecture/overview.md)
- [Library API](../api/library-api.md) — the surface plugins are written against.
- [MsgEvent](../api/msgevent.md) — the universal wire message.
- [Configuration](../getting-started/configuration.md) — the `-D`/`CRESCO_*`/`agent.ini` inputs each module reads.
