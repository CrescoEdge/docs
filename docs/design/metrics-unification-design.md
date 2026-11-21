# Cresco — Metrics & Measurements Unification (B-2)

**Status:** ✅ SHIPPED + PROVEN (2026-07-03). All phases (0–4) complete. Live multi-node mesh proof:
`run/tests/metrics_unification_test.sh` **11/11 PASS** — one `getmetricinventory` (scope=global) pulled
via the Python client returns, from every node in a global+region+agent mesh, the controller's own
Micrometer groups (`jvm/controller/netlink/regional/global`), **every** plugin's metrics via the unified
`getmetrics` contract (`sysinfo`→processor/memory/disk/network, `wsapi`, `stunnel`, `repo`), the sysinfo
`resource_summary`, and child-node aggregation (mesh fan-out). No regression: tenant isolation 10/10,
B-1 startup 4/4. See §9 for the proof detail.
**Author:** Claude (Opus 4.8) working session, on user direction ("complete full unification of all
metrics and measurements across all code libraries, plugins, and clients, everything").
**Related:** `broken-and-untouched-report.md` §B-2, `health-check-design.md` (the template this mirrors),
`link-metrics-design.md` (net-metrics that already live on the unified registry).

---

## 1. Problem

Monitoring was split into **health / measurements / benchmarks**. **Health** was migrated onto Felix
Health Checks (cross-bundle OSGi discovery, tag grouping, mesh ping/pong rollup — see
`health-check-design.md`). **Measurements** never followed. Today three measurement paths coexist:

1. **Micrometer `MeasurementEngine`** (`io.cresco.library.metrics.*`) — the intended standard. A
   per-bundle registry; `getAllMetrics()` returns `Map<group, List<Map<String,String>>>`. The
   controller populates `jvm`/`processor`/`controller`/`netlink` groups; the `getmetricinventory` EXEC
   already merges controller-own metrics + each **local** plugin's metrics (pulled via a standard
   `getmetrics` EXEC) + an optional resource summary. **Only `stunnel`** among plugins implements
   `getmetrics`.
2. **Legacy KPI / resource path** — `sysinfo`'s bespoke `getsysinfo` EXEC (OSHI cpu/mem/disk/net/fs as
   ad-hoc nested JSON), aggregated by `PerfControllerMonitor.getResourceInfo`, persisted to Derby
   `inodekpi` via `MsgEvent.Type.KPI`. Not on Micrometer. (Note: `AppScheduler` placement is
   location-based and does **not** actually consume resource data today.)
3. **Client side** — `clientlib` (Java) and `pycrescolib` (Python) can query agent `resourceinfo` but
   have **no** way to pull the unified metric inventory, and no first-class client-side perf counters.

Net: `wsapi`, `sysinfo`, `repo` don't participate in the unified registry; the regional/global controller
gauges are stubbed; `getmetricinventory` isn't mesh-aggregated; clients are blind to it.

## 2. Goal

**One measurement model, everywhere.** Every bundle (controller + every plugin) registers its metrics in
its Micrometer `MeasurementEngine` and exposes them through the *single* `getmetrics` EXEC contract. The
controller aggregates them — locally, region-wide, and global-wide — behind the *single*
`getmetricinventory` surface. Both clients (Java/Python) get a first-class API to pull that
inventory and to measure their own dataplane. The legacy `sysinfo`/KPI path is **kept for back-compat** but folded in: its data
also appears as MeasurementEngine gauges, and the unified `resource_summary` derives from them.

**Non-negotiables (match the rest of this codebase):**
- **Additive & back-compat.** No public library API is removed or changed incompatibly; `getsysinfo`,
  `resourceinfo`, and the `inodekpi` KPI path keep working unchanged.
- **Flag-gated where behavior changes.** New collection defaults on only where it is cheap and safe.
- **No new fabric-breaking coupling.** Aggregation reuses existing EXEC/RPC + (optionally) the health
  mesh ping/pong pattern; it never blocks the control plane.

## 3. Target architecture

```
                         ┌─────────────────────────────────────────────┐
                         │  client (Java clientlib / Python pycrescolib)│
                         │  get_metric_inventory(region, agent[, opts]) │
                         │  get_metrics(region, agent, pluginId)        │
                         │  dataplane.get_metrics()  (client-side perf) │
                         └───────────────────────┬─────────────────────┘
                                                 │ EXEC getmetricinventory / getmetrics
                                                 ▼
        ┌───────────────────────────────────────────────────────────────────────┐
        │ controller  PerfControllerMonitor.getMetricInventory(scope, opts)      │
        │  • controller-own MeasurementEngine.getAllMetrics()  (jvm/processor/   │
        │    controller/netlink + NEW regional/global gauges)                    │
        │  • each LOCAL plugin  ── EXEC getmetrics ──▶ plugin.getAllMetrics()     │
        │  • resource_summary  ── derived from sysinfo metrics (back-compat: also │
        │    still available via getsysinfo / resourceinfo)                       │
        │  • scope=region|global ── fan out getmetricinventory to children/nodes  │
        └───────────────────────────────────────────────────────────────────────┘
             ▲                    ▲                    ▲                    ▲
   getmetrics│          getmetrics│          getmetrics│          getmetrics│
   ┌─────────┴───┐    ┌───────────┴──┐    ┌────────────┴─┐    ┌─────────────┴┐
   │  stunnel    │    │   sysinfo    │    │    wsapi     │    │     repo     │
   │ (reference) │    │ OSHI→gauges  │    │ WS conns,    │    │ jar count,   │
   │ tunnels/... │    │ cpu/mem/disk │    │ dp throughput│    │ repo size    │
   └─────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
   every plugin: MeasurementEngine registry + `getmetrics` returns gson(getAllMetrics())
```

**The single plugin contract** (already what stunnel does — becomes universal):

```java
// in <plugin>/PluginExecutor.executeEXEC:
case "getmetrics":
    incoming.setParam("metrics", gson.toJson(measurementEngine.getAllMetrics()));
    incoming.setParam("status", "10");
    return incoming;
```

`getAllMetrics()` returns `Map<group, List<Map<String,String>>>` — group → list of
`{name, class, type, value|count|mean|max|...}`. This is the wire schema for the whole fabric.

## 4. Unified JSON shape (getmetricinventory)

```json
{
  "node": "<region>_<agent>",
  "scope": "node|region|global",
  "metrics_by_source": {
    "<region>_<agent>:io.cresco.agent.controller": { "jvm": [...], "processor": [...],
                                                      "controller": [...], "netlink": [...],
                                                      "regional": [...], "global": [...] },
    "<region>_<agent>:io.cresco.sysinfo": { "processor": [...], "memory": [...], "disk": [...],
                                            "network": [...] },
    "<region>_<agent>:io.cresco.wsapi":   { "wsapi": [...] },
    "<region>_<agent>:io.cresco.stunnel": { "stunnel": [...] },
    "<region>_<agent>:io.cresco.repo":    { "repo": [...] }
  },
  "resource_summary": { "cpu_core_count": N, "mem_available": B, "mem_total": B,
                        "disk_available": B, "disk_total": B },
  "children": { "<child_region>_<child_agent>": { ...same shape (scope fan-out)... } }
}
```

Every metric row keeps its `group` under its source, so a caller can flatten by group across sources
(all `processor` from everywhere) or read a single bundle.

## 5. Phasing

| Phase | Scope | Flag(s) | Breaking? |
|-------|-------|---------|-----------|
| **0 Design** | this doc | — | no |
| **1 Plugins** | `sysinfo`, `wsapi`, `repo` implement `getmetrics` via MeasurementEngine (OSHI→gauges; WS conns/throughput; repo size). `stunnel` already done — align group names. | `plugin_metrics_enabled` (default true) | no (additive EXEC + gauges) |
| **2 Controller** | Un-stub `initRegionalMetrics` (`brokered.agent.count`) + `initGlobalMetrics` (resource/app queue depth). `getMetricInventory` gains `scope=node|region|global` fan-out; `resource_summary` derives from sysinfo metrics. Default `include_plugins=true`. | `enable_controllermon` (exists), `metric_inventory_scope` | no |
| **3 Clients** | `clientlib` (Java) + `pycrescolib` (Python): `get_metric_inventory(...)` / `get_metrics(...)` helpers wrapping the EXEC; client-side dataplane throughput/latency counters (`dataplane.get_metrics()`). | — | no (new API only) |
| **4 Test/docs** | Live-mesh test pulling unified metrics from all plugins + both clients; exhaustive docs; push all repos. | — | no |

**Legacy path disposition:** `getsysinfo` / `resourceinfo` / `inodekpi` KPI persistence are **retained
unchanged** (public clients depend on `resourceinfo`; the KPI table is written by the perf tick). The
unification adds a *parallel* Micrometer view of the same data and makes it the primary read surface.
A future phase may deprecate the ad-hoc `perf` blob once all readers move to `getmetricinventory`.

## 6. Reused health-check patterns (see health-check-design.md)

- **Standard per-source DTO** — metrics already serialize to a uniform `Map` shape; keep it.
- **Tag/group-based grouping** — `getAllMetrics()` groups are the metric analogue of health tags.
- **Single aggregating surface** — `getmetricinventory` is to metrics what the health executor's
  `execute(tags)` is to health.
- **Mesh fan-out** — `scope=region|global` mirrors the health mesh rollup, but pull-based (on demand)
  rather than pushed every ping, to keep the control plane light.
- **Additive, flag-gated, in-JVM compute** — same discipline that made the health migration safe.

## 7. Metric catalog added by this work

| Bundle | Group | Metric | Type |
|--------|-------|--------|------|
| controller | regional | `brokered.agent.count` | gauge |
| controller | global | `incoming.resource.queue`, `incoming.application.queue` | gauge |
| sysinfo | processor | `system.cpu.count`, `system.cpu.load`, `system.load.average.1m`, `cpu.context.switches`, `cpu.interrupts` | gauge |
| sysinfo | memory | `memory.total`, `memory.available`, `memory.swap.total`, `memory.swap.used` | gauge (bytes) |
| sysinfo | disk | `disk.total`, `disk.available`, `disk.read.bytes`, `disk.write.bytes` | gauge (bytes) |
| sysinfo | network | `net.bytes.sent`, `net.bytes.recv`, `net.packets.sent`, `net.packets.recv` | gauge |
| sysinfo | sensors | `sensor.cpu.temperature`, `sensor.cpu.voltage`, `sensor.fan.max.rpm` | gauge |
| sysinfo | power | `power.remaining.percent`, `power.remaining.seconds` | gauge |
| sysinfo | system | `system.uptime.seconds`, `system.process.count`, `system.thread.count` | gauge |
| sysinfo | gpu | `gpu.count`, `gpu.vram.total` | gauge (VRAM = capacity, not live GPU load — OSHI can't) |
| sysinfo | devices | `usb.device.count`, `display.count`, `soundcard.count` | gauge |

**OSHI upgraded 4.2.1 → 7.3.2 (latest; JNA 5.19.1).** The old 4.2.1 bundled a pre-arm64 JNA (5.5.0) with
no arm64 native lib, so hardware calls threw `UnsatisfiedLinkError` on arm64 hosts and sysinfo reported
zeros there. 7.3.2 ships arm64 macOS/Linux native libs (verified: `darwin-aarch64/libjnidispatch.jnilib`
embedded in the bundle), so real values now flow on arm64 (e.g. cpu temp 45°C, 14 cores, 36 GB, uptime).
The upgrade also unlocked the `sensors`, `power`, and `system` groups and the load-average / swap / disk-IO
/ packet counters above — measurements OSHI 4.2.1 didn't expose. API migration (arrays→Lists,
`getProcessorIdentifier()`, `getVersionInfo()`, `ProcessSorting`) was confined to `SysInfoBuilder`
(the only OSHI user). Each read stays guarded so a metric OSHI can't supply on a given platform degrades
to 0 without dropping sysinfo from the inventory.
| wsapi | wsapi | `wsapi.dataplane.connections`, `wsapi.dataplane.bytes` | gauge |
| repo | repo | `repo.plugin.count`, `repo.bytes` | gauge |
| stunnel | stunnel | `stunnel.active.tunnels/clients/targets` (existing) | gauge |

## 8. Verification

- `run/tests/metrics_unification_test.sh` — bring up global+region+agent with all plugins; assert
  `getmetricinventory` returns metrics from **every** plugin group + controller groups + resource
  summary, at node/region/global scope.
- Both clients (Java/Python): a Java example + a Python example call `get_metric_inventory` and
  print the unified view; assert non-empty per-plugin groups.
- No regression: existing health/security/startup suites stay green.

## 9. Proof (2026-07-03) — what shipped and how it was verified

**Live mesh test — `run/tests/metrics_unification_test.sh` 11/11 PASS.** A global (wsapi on) + region +
agent are brought up; the **Python client's new `globalcontroller.get_metric_inventory(scope='global')`**
issues one EXEC and gets back the whole fabric's metrics. Groups observed across all sources:
`controller, disk, global, jvm, memory, netlink, network, processor, regional, repo, stunnel, wsapi`
(11 sources, 2 mesh children). Asserted: client connect+query; controller JVM + `controller` timer +
un-stubbed `regional`/`global` role gauges; `resource_summary`; **sysinfo** cpu/mem/disk/net; **wsapi**;
**stunnel**; **repo**; mesh fan-out reached child nodes.

**What was built per phase:**
- **Plugins** — `sysinfo` (`SysInfoBuilder.getMetricsJson`: OSHI cpu/mem/disk/net as gauges, TTL-cached
  + pre-warmed, every OSHI read guarded with `Throwable` so a broken native lib degrades to zeros instead
  of dropping the plugin), `wsapi` (`DataPlaneWsHandler` static connection/byte/message counters →
  `PluginExecutor` gauges), `repo` (`ExecutorImpl` repo jar count/bytes). Each adds `case "getmetrics"`.
- **Controller** — un-stubbed `PerfControllerMonitor.initRegionalMetrics` (`brokered.agent.count`) and
  `initGlobalMetrics` (`reachable.agent.count`, `incoming.candidate.brokers`); `getMetricInventory` now
  takes `scope=node|region|global` and fans out to children **concurrently** (bounded per-child +
  overall budget); plugin `getmetrics` default-on; `getSysInfo` RPC bounded. `getmetricinventory` EXEC
  added to `AgentExecutor` and `RegionalExecutor` (was global-only) so every node answers.
- **Clients** — `clientlib` (Java) `GlobalController.get_metric_inventory(...)`; `pycrescolib` (Python)
  `globalcontroller.get_metric_inventory(...)` + `dataplane.get_metrics()` client-side counters.

**Key gotchas found + fixed during bring-up:**
- `UnsatisfiedLinkError`/`NoClassDefFoundError` from OSHI's JNA (arm64 dev box, x86 native lib) is an
  **`Error`, not an `Exception`** — the original `catch (Exception)` let it kill the whole `getmetrics`.
  Fixed by registering gauges first and guarding every OSHI access with `Throwable`. On production Linux
  the native lib loads and real values flow; on the arch-mismatched dev box the plumbing still reports
  (zeros), so unification is proven regardless.
- sysinfo's `getmetrics` briefly missed the per-plugin RPC timeout because OSHI re-enumerates NICs/FS
  every call (~2.6s) — fixed with the TTL cache + pre-warm.
- Mesh fan-out children were dropped because a child's nested inventory can wait on non-metric plugins;
  fixed by querying children concurrently with a generous bounded timeout, and by adding the handler to
  `RegionalExecutor` (the region node had no handler).

**Config flags (all additive):** `metrics_rpc_timeout_ms` (per-plugin, 2500), `metrics_fanout_timeout_ms`
(per-child, 12000), `resource_rpc_timeout_ms` (sysinfo resource RPC, 3000), `sysinfo_metrics_ttl_ms`
(5000). Legacy `getsysinfo`/`resourceinfo`/`inodekpi` paths unchanged.
</content>
