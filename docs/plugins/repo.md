# repo — Plugin/JAR Repository

The `repo` plugin (bundle `io.cresco.repo`) is a functional plugin that stores, reports on, and serves other plugins' JARs within the fabric. It is the source `PluginAdmin` pulls from when a plugin is scheduled onto an agent that does not already have the JAR cached.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/repo` |
| Bundle symbolic name | `io.cresco.repo` |
| Java files | 3 |
| Loaded | At runtime by the controller's `StaticPluginLoader` (default-on for a global). |
| Type | Functional plugin (`@Component` implementing `PluginService`). |

## What it does

`repo` serves and stores plugin JARs from a per-plugin data directory. It stores and serves only jars that verify against `plugin_signing_truststore` ([Signed plugins](../security/signed-plugins.md)); a jar is named by its SHA-256 content hash, carried in the field still called `md5` (advisory: a mismatch is refused, a match proves nothing). Multiple repo instances can exist across the mesh; the [controller](controller.md)'s global tier tracks them (`listrepoinstances`) and a `savetorepo` operation fans a `putjar` RPC out to every repo instance so a plugin JAR is replicated fabric-wide.

## Executor actions

`ExecutorImpl` handles EXEC actions ([`MsgEvent`](../api/msgevent.md) type `EXEC`):

| Action | Behaviour |
|--------|-----------|
| `repolist` | Return a compressed `repolist` — the JAR inventory plus this repo's region/agent/pluginID. |
| `getjar` | Find a JAR by `action_pluginname` + `action_pluginmd5` (its content hash) and attach its bytes as `jardata`. |
| `putjar` | Verify the supplied JAR's signature, write exactly the verified bytes to the repo directory, recompute its content hash, and set `uploaded` / `md5-confirm`; an unsigned or foreign-signed jar is refused (`status 9`, `refusal=signature`). |

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `repo_dir` | `repo` | Directory (under the plugin data directory) where JARs are stored. |

See [Configuration](../getting-started/configuration.md).

## Where it fits

When the global scheduler places a plugin on an agent, the agent's `PluginAdmin` resolves the JAR — checking bundle / absolute path / local cache / embedded resources, and finally a **remote repo** via an RPC `getjar` (the fetched jar is installed only if its signature verifies). `repo` is the endpoint that answers those calls, and `putjar`/`savetorepo` is how new plugin artifacts are published into the fabric. See [controller](controller.md) for the plugin lifecycle and the global tier.

## Observability

- **Metrics** — `getmetrics` exposes `MeasurementEngine` gauges `repo.plugin.count` (jars in the
  repository) and `repo.bytes` (total on-disk size); aggregated by the controller's
  `getmetricinventory`. See [Metrics & Measurements](../architecture/metrics.md#plugin-repo).
- **Health** — registers the `repo` Felix HealthCheck (tag `local`) reporting the plugin-jar count.
  See [Health & State](../architecture/health.md#plugin-checks).

## See also

- [Overview](overview.md) — plugin lifecycle and JAR resolution.
- [controller](controller.md) — `PluginAdmin` (JAR resolution) and the global `savetorepo` fan-out.
