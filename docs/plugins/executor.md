# executor — Process / Shell Execution

The `executor` plugin (bundle `io.cresco.executor`) runs OS processes and shell commands on an agent,
**streaming stdout/stderr over the mesh dataplane** and accepting **stdin through `write_stdin`**, with
optional per-process metrics. It is the fabric's remote-execution surface.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/executor` |
| Bundle symbolic name | `io.cresco.executor` |
| Loaded | At runtime by the controller (gated by `enable_executor`, default true). |
| Type | Functional plugin (`@Component` implementing `PluginService`). |

## The model

A **runner** is a configured command identified by a `stream_name` — which is also the dataplane
stream its stdio rides on. Lifecycle: `config_process` (register) → `run_process` / `start_process`
(execute) → `status_process` (poll) → `end_process` (kill) → reusable. `reset_runners` stops
everything.

**Stdio over the dataplane.** stdout and stderr are forwarded as dataplane `TextMessage`s on the
**GLOBAL** topic, sharded by `stream_name`, with a `type` of `output` or `error`; an external client reads them via
`get_dataplane(stream_name)`. On exit, an `execution_log` event (with the exit code) and a `delete_exchange` event are
emitted on the same stream.

**Stdin is opt-in, and never on the dataplane.** A runner keeps stdin open only when `config_process` is sent with
`stdin=true` (an `-interactive-` command always does); otherwise the process sees end-of-file at once. Stdin is written
with the `write_stdin` action (a MsgEvent), accepted only from the verified caller that started the runner, or from an
`admin` or `infra` caller (`status 9`, `not-owner` / `no-stdin` / `not-running` otherwise). A dataplane handshake that
would write a process's stdin is refused by wsapi for every caller. A command containing `-interactive-` starts an
interactive shell driven entirely through `write_stdin`. Before the 1.3 hardening stdin came from the dataplane.

## Executor actions

Full table: [Plugin Actions › executor](../api/plugin-actions.md#executor-plugin).

| Action | Behaviour |
|--------|-----------|
| `config_process` | Register a runner (`stream_name` + `command`, optional `metrics` and `stdin`); does not start it (`infra`/`admin`). |
| `write_stdin` | Write a line of text (`data`) or bytes (`data_b64`) to a running runner's stdin, or close it (`close_stdin`). |
| `run_process` / `start_process` | Start a configured runner. |
| `status_process` | Report whether the runner is currently running. |
| `end_process` | Kill the process **and its whole descendant tree** and free the runner. |
| `reset_runners` | Stop every runner on this executor. |
| `getmetrics` | Central metrics — see [Observability](#observability). |

## Safeguards

- **Injection-safe, precise kill** — `end_process` kills the launched `Process` and its descendants via
  the JVM `ProcessHandle`, never by interpolating the command into a `ps | grep | kill` shell string.
- **No thread/handle leaks** — stdout/stderr readers exit on EOF; stdin is closed when the process
  ends; every worker is a named daemon; `isStopped()` kills all runners so
  nothing is orphaned.
- **Reusable stream names** — `end_process` removes the runner from the registry, so a `stream_name`
  can be reconfigured after stop.

> **Security note:** `executor` runs arbitrary shell commands by design. Authorization is enforced at
> the Cresco broker/tenant layer — only principals allowed to route messages to this plugin can invoke
> it. See [Security & Identity](../architecture/security.md) and [Multi-Tenancy](../architecture/tenancy.md).

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `stream_name` + `command` | *(unset)* | If both set at load, a runner is auto-created and started. |
| `metrics` | `false` | Collect per-process OSHI metrics (streamed live on the dataplane). |
| `enable_executor` | `true` | Agent-level auto-load gate. |

See [Configuration](../getting-started/configuration.md).

## Observability

- **Metrics** — `getmetrics` exposes `MeasurementEngine` gauges `executor.runners.configured` and
  `executor.runners.active`; aggregated by the controller's `getmetricinventory`. Per-runner process
  telemetry (cpu/mem/io of the tracked PID + descendants) streams separately when `metrics=true`. See
  [Metrics & Measurements](../architecture/metrics.md#plugin-executor).
- **Health** — registers the `executor` Felix HealthCheck (tag `local`): reports the running /
  configured runner counts. See [Health & State](../architecture/health.md#plugin-checks).

## See also

- [Plugin Actions](../api/plugin-actions.md#executor-plugin) — the full action table.
- [Data Plane](../architecture/dataplane.md) — how stdio rides the GLOBAL topic.
- [Client Libraries](../clients/overview.md) — reading/writing a runner's stream from a client.
