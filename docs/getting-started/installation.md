# Installation & Build

Cresco builds from source into a single runnable **agent uber-jar**. This page covers requirements,
the build, and the resulting artifacts.

## Requirements

| Requirement | Version / note |
|-------------|----------------|
| **JDK** | 21 (every module targets `<jdk>21</jdk>`) |
| **Maven** | 3.8+ |
| **Python** | 3.8+ (only for the [Python client](../clients/python.md)) |
| **OS** | Linux / macOS (developed on both) |

## Get a release

Every commit of the agent's `main` branch is a release with a name of its own, `1.3.<n>` (n: the number of commits up
to it), published at [github.com/CrescoEdge/agent/releases](https://github.com/CrescoEdge/agent/releases) and never
replaced. Each one holds the agent jar (`agent-1.3.<n>.jar`), its SHA-256, `bundles.lock` (every embedded Cresco
bundle with its SHA-256 and the published snapshot it is) and the SBOM. The release `1.3-SNAPSHOT` carries the latest
jar under the old name `agent-1.3-SNAPSHOT.jar` and is replaced each time: take it to try the latest 1.3, and name a
`1.3.<n>` release for anything you deploy, so that every node runs the same known build.

```bash
shasum -a 256 -c agent-1.3.376.jar.sha256   # the download is the published file
java -jar agent-1.3.376.jar --version       # the release, the build, every embedded bundle (no node is started)
```

`--version` answers "what does this node run" from the jar alone. To check a jar against the source it names: in a
checkout of the tag, `scripts/rebuild-check.sh agent-1.3.376.jar` verifies the embedded bundles against `bundles.lock`,
and with `--rebuild` compares every entry with a build of that checkout (identical with the same JDK build, but for
the four entries that record the build itself; another 21.x patch release of the JDK changed one debug table in one
class in our test). A release signed by an operator's key is verified with `scripts/verify-release.sh` (agent
repository, README).

## Build from source

Cresco is a set of OSGi bundles — one module (and one git repository) per directory under `code/`. The
`build.sh` driver builds each in dependency order and stages the runnable jar into `run/`.

```bash
./build.sh            # build everything: components + clients + agent uber-jar
./build.sh components # just the OSGi bundles (logger, library, core, repo, sysinfo, wsapi, stunnel, controller)
./build.sh agent      # stage bundles + assemble the agent uber-jar into run/
./build.sh clients    # build the Java clientlib + set up the Python venv
```

!!! warning "The `bundle:bundle` gotcha"
    Each module declares `<packaging>jar</packaging>`, and the `maven-bundle-plugin` only rewrites the
    artifact into a **real OSGi bundle** when its `bundle:bundle` goal runs explicitly. The build recipe
    is therefore `mvn clean package bundle:bundle` — a bare `mvn install` produces a non-bundle jar that
    Felix cannot start. Always build through `build.sh` (or run `mvn ... bundle:bundle` yourself).

Build order (from `build.sh`): base layer `logger`, `library` (no Cresco deps) → plugin layer `core`,
`repo`, `sysinfo`, `wsapi`, `stunnel`, `controller` → then the `agent` uber-jar last.

## Artifacts

| Artifact | Location | What it is |
|----------|----------|-----------|
| `agent-1.3-SNAPSHOT.jar` | `run/` | The runnable Cresco node (Felix host + all bundles). This is what you launch. |
| Per-module bundles | `agent/src/main/resources/` | Committed there under version-less names and pinned by `agent/bundles.lock`: the agent build embeds exactly the bundles the lock names, or fails naming the jar. `prebuild.sh --update <name>` takes a bundle's latest published snapshot (checked against Maven Central's checksum, never from `~/.m2`); `prebuild.sh --lock <name>` records one you built and copied in yourself. |
| `clientlib` jar | `code/clientlib/target/` | The external [Java SDK](../clients/java.md) (shaded). |
| `pycrescolib` | `run/venv/` | The [Python SDK](../clients/python.md), installed editable into a venv. |

The bootstrap (`HostApplication`) and `StaticPluginLoader` reference the staged bundles by **version-less**
names (e.g. `controller.jar`), so a version bump never touches runtime Java — only the build version.

## Run hygiene

Each node keeps on-disk state (Felix cache, Derby DB, ActiveMQ journal) under a `cresco-data/`
directory in its working directory. **Two nodes on the same host must never share a working directory** —
the launch scripts give each node its own `run/nodes/<region>-<agent>/`.

!!! note "Wipe state between builds/runs"
    Stale Derby/broker state from a previous run can cause spurious plugin/state errors. Between runs (and
    especially after a rebuild) remove `run/cresco-data` and `run/nodes`:
    ```bash
    rm -rf run/cresco-data run/nodes
    ```

Next: **[Quickstart](quickstart.md)** to launch a mesh and connect a client.
