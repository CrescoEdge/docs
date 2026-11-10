# filerepo — Distributed File/Artifact Repository

The `filerepo` plugin (bundle `io.cresco.filerepo`) stores, catalogs, and moves files and plugin JARs
across the fabric. Where [`repo`](repo.md) is specialized for plugin JARs, `filerepo` is a general
file repository: it watches a directory, tracks every file in an embedded catalog, and moves files of
**any size** across the mesh — inline for small files, byte-range dataplane streaming for large ones,
watched-directory sync between peers, and push-to-remote-repo.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/filerepo` |
| Bundle symbolic name | `io.cresco.filerepo` |
| Catalog | Embedded Derby (file path / content hash in the `md5` column / size / mtime) |
| Loaded | At runtime by the controller (default-on for a global). |
| Type | Functional plugin (`@Component` implementing `PluginService`). |

## What it does

A **repo** is a directory of files tracked in a Derby catalog. In *producer* mode `filerepo` scans a
`scan_dir` on an interval and keeps the catalog current; in *consumer* mode it receives files pushed
or synced from peers. Files move by one of three mechanisms:

- **Inline** (`getfile` / `getjar`) — the whole file rides back in one control-plane reply. Guarded by
  `max_inline_bytes` (default 512 KB) so an inline read can never exceed the wss frame limit; larger
  files must stream.
- **Streaming** (`streamfile`) — a byte-range is sent over the dataplane in chunks, each carrying a
  `transfer_id` + `seq_num` header the receiver reassembles by. Any size; the chunk size is clamped
  to a safe ceiling so a caller can't oversize a frame.
- **Directory sync** (`repolistin` / `repoconfirm`) — a producer's catalog diff is sent to a consumer,
  which pulls only the changed files.

Safeguards throughout: all catalog queries are parameterized (SQL-injection-safe); `putjar`/`getfile`
paths are canonicalized and confined to the repo dir (path-traversal-safe); `putjar` accepts only a jar signed by a
signer chaining to `plugin_signing_truststore` and computes its content hash itself (a supplied `md5` that differs is
refused); the transfer pool is bounded and cleaned up on stop.

## Executor actions

`ExecutorImpl` handles EXEC actions ([`MsgEvent`](../api/msgevent.md) type `EXEC`). `status`/
`status_code` `10` = success; `object` returns are gzip+base64 compressed, `bytes` returns are base64.
Full table: [Plugin Actions › filerepo](../api/plugin-actions.md#filerepo-plugin).

| Action | Behaviour |
|--------|-----------|
| `getrepofilelist` | List catalog rows (path/md5/size/mtime), paginated via `limit`/`offset`. |
| `repolist` | Plugin/jar inventory of the repo dir + server identity. |
| `getfile` / `getjar` | Return a cataloged file / plugin jar inline (size-guarded). |
| `putjar` / `putfiles` | Write a signed jar (signature-verified) / land pushed files into a repo. |
| `putfilesremote` | Push a set of local files to another agent's repo. |
| `streamfile` / `streamfilecancel` | Stream a byte-range over the dataplane (any size) / cancel it. |
| `removefile` / `clearrepo` | Remove one file / clear the repository directory (disk + catalog), armed explicitly: see [`repostate` and clearing](#repostate-and-clearing). |
| `repolistin` / `repoconfirm` | Directory-sync consumer side / handshake ack. |
| `getscandir` | Report the configured scan directory. |
| `repostate` | What would be lost with this repository: `LIVE`, `WORKSPACE_CLEAN`, `WORKSPACE_DIRTY`, `MIRROR` or `ONLY_COPY`. |
| `liverelay` / `liverealise` | Carry a signed request to the agent's live-tier provider unchanged / realise a live-backed repository's copy again (`infra`/`admin`). |
| `getmetrics` | Central metrics — see [Observability](#observability). |

## Live-backed repositories and live-tier roots

filerepo is a **client** of a live tier, never part of it: it talks only to a live-tier provider service registered by
another bundle on the same agent (`LiveTierClient`; GaiaKeep's live tier is one, see
[Projects built on Cresco](../reference/projects.md)). Without a provider it behaves as before.

- **Live-backed repository.** A repository whose configuration names `live_row` is a live-tier copy or workspace held
  at this node. It runs no scanner, broadcast or subscription, answers only `repostate`, `liverelay`, `liverealise`,
  `getmetrics` and `getcapabilities`, and refuses every other verb (`status 9`, `refusal=live-backed`), even when the
  provider cannot be reached. The backing is read once at start; no message can change it.
- **Live-tier roots.** Files under a provider's live root are the provider's. filerepo learns each registered
  provider's root (and takes `live_roots` from configuration) and remembers it in its catalog: `getfile`, `streamfile`,
  `putfilesremote` and `putfiles` refuse a file in a live root (`refusal=live-root`), a repository whose directory is or
  holds a live root is refused like a live-backed one, and while any registered provider's root is unknown every file
  path is refused.

## `repostate` and clearing

`repostate` answers from the provider's cached answers (live-backed) or from configuration and filerepo's own catalog,
never from a walk of the tree:

| State | Meaning |
|---|---|
| `LIVE` | backed by a live copy: read-only here, recreatable by realising it again |
| `WORKSPACE_CLEAN` / `WORKSPACE_DIRTY` | backed by a workspace without / with unpublished changes (unknown counts as dirty) |
| `MIRROR` | a subscriber mirror: its producer holds what it holds |
| `ONLY_COPY` | the origin of what it holds: a producer, an on-demand repository, or a mirror that files were landed in directly |

`clearrepo` is armed explicitly: `confirm` must name the repository directory exactly (`refusal=unconfirmed`); it
refuses an `ONLY_COPY` unless `force=true`, which needs the `admin` role (`refusal=only-copy`,
`refusal=force-needs-admin`); and it never clears a protected directory (the filesystem root, a system directory, the
home, working, JVM, agent-data or plugin-data directory, or one that is or holds a live-tier root;
`refusal=protected-dir`). A clear holds the repository gate exclusively, so nothing lands between its check and its
delete. A live-backed repository refuses `clearrepo`, forced or not.

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `scan_dir` | *(unset)* | Directory to watch/catalog (producer mode). |
| `filerepo_name` | *(unset)* | Logical repo name for mesh discovery/sync. |
| `enable_scan` | `false` | Enable producer-mode periodic scanning. Must be explicitly set to `true` to activate the producer scan path; it is **not** on by default. |
| `scan_delay` / `scan_period` | `5000` / `15000` ms | First-scan delay / scan interval. |
| `repo_dir` | `<agent data dir>/filerepo/repos/<filerepo_name or default>` | Repo directory for on-demand/consumer storage. Every verb uses one directory: `scan_dir`, else `repo_dir`, else this default. |
| `max_inline_bytes` | `524288` (512 KB) | Max file size returned inline by `getfile`/`getjar`. |
| `stream_buffer_size` | `262144` (256 KB) | Default streamfile chunk size (hard-capped at 768 KB). |
| `transfer_threads` | `8` | Max concurrent streamfile transfers (bounded pool). |
| `live_row` | *(unset)* | Back this repository by a live-tier row held at this node (read once at start). |
| `live_roots` | *(unset)* | Comma-separated live-tier roots to refuse from the start, before any provider registers. |
| `live_list_cache_ms` | `1000` | How long a live-backed repository reuses the provider's `list` answer for `repostate`. |

See [Configuration](../getting-started/configuration.md).

## Observability

- **Metrics** — `getmetrics` exposes `MeasurementEngine` gauges `filerepo.files.count` (catalog size)
  and `filerepo.active.transfers` (in-flight `streamfile` count); aggregated by the controller's
  `getmetricinventory`. See [Metrics & Measurements](../architecture/metrics.md#plugin-filerepo).
- **Health** — registers the `filerepo` Felix HealthCheck (tag `local`), discovered by the
  controller's health executor: reports the catalog size and verifies the repo dir is writable
  (`WARN` if not). See [Health & State](../architecture/health.md#plugin-checks).

## See also

- [repo](repo.md) — the plugin-JAR-specialized sibling repository.
- [Plugin Actions](../api/plugin-actions.md#filerepo-plugin) — the full action table.
- [Data Plane](../architecture/dataplane.md) — how `streamfile` chunks ride the mesh.
