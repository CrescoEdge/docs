# Testing

Cresco validates behavior primarily through **integration harnesses** under `run/tests/` that stand up a
real fabric (global + regions + agents), exercise it, and assert outcomes — plus a set of Python
performance harnesses. This page maps the suites so you know what proves what.

## Running a suite

Most suites are self-contained shell scripts that build/launch nodes, run assertions, and tear down:

```bash
cd run/tests
./tenant_namespacing_test.sh      # e.g. tenant isolation under namespacing
```

They source `run/cresco.env` for the shared secrets/keys, launch nodes into `run/nodes/`, and print
`PASS`/`FAIL` lines with a final tally. Rebuild the agent jar first (`./build.sh agent`) if you changed code.

## Security & tenancy

| Suite | Proves |
|-------|--------|
| `security_foundation_test.sh` | identity ⇄ DN, message signing (accept/reject), chain-trust to a region CA |
| `mtls_binding_test.sh` | mutual-TLS binds the principal from the client-cert DN (anti-spoof) |
| `regional_ca_mtls_test.sh` | regional-CA enrollment + fabric-wide mutual-TLS federation |
| `tenant_isolation_mesh_test.sh` | cross-tenant deny at every broker in a federated mesh (namespacing off) |
| `tenant_namespacing_test.sh` | end-to-end tenant namespacing: fabric forms + zero self-denial; TENANT confined; SUPERUSER god view |

## Health, state & recovery

| Suite | Proves |
|-------|--------|
| `exhaustive_health_test.sh` | health checks, grace/sticky windows, parent-link loss → recovery |
| `b1_startup_race_test.sh` | TLS trust-ready connect gate (no PKIX race on same-host bring-up) |
| `mesh_massive_test.sh` | large-mesh bring-up and fault isolation |

## Capability & metrics

| Suite | Proves |
|-------|--------|
| `capability_inventory_test.sh` | fabric-wide `getcapabilityinventory` (all actions self-described) |
| `metrics_unification_test.sh` | `getmetricinventory` returns every plugin's metrics at node/region/global scope |

## Performance

| Harness | Measures |
|---------|----------|
| `dataplane_perf.py` / `dataplane_internode.py` / `dataplane_aggregate.py` | data-plane throughput/latency |
| `stunnel_perf.py` / `stunnel_direct.py` | tunnel throughput |
| `msgevent_perf.py` | control-plane message rate |
| `adaptive_net_suite.sh` | network auto-tuning / link-metric behavior |
| `run_speedtests.sh` | aggregate speed suite (see `SPEEDTEST_RESULTS.md`) |

## Live harnesses in the controller repository

The security and dataplane contracts are proven by live harnesses that ship with the controller, under
`controller/docs/tests/`. Each starts and stops only its own JVMs on one host, prints `PASS`/`FAIL` lines (or one JSON
result line for the benchmarks) and needs an agent jar, a working directory and, where it drives clients, a Python with
pycrescolib:

| Harness | Proves / measures |
|---------|-------------------|
| `peer_identity_live_test.sh` | peer identity binding (agent to broker, broker bridges) and dataplane recipient binding with mutual TLS and tenant authorization; `DEFAULTS=1` runs a fabric with no security setting at all |
| `fabric_matrix.py` | one cell of the matrix **topology x security profile x lifecycle events** on a live fabric (one JVM per node): `single`, `tree`, `mesh`, `tenants`, `alias` (and `regions<N>` for scale) under `standard`, `production` or `upgrade`, through start, restarts of an agent, a region and the global, a global outage, a region re-key, a moved agent, a cut bridge and a revocation. After each event it checks that the global lists exactly the nodes that should be up, that every node answers, and that **nothing a node logged is outside a list of known items with measured bounds**: any other refusal, drop, strip or ERROR fails the cell. About 11 minutes for `tree` |
| `authority_live_test.sh` | [authority follows the tree](../security/authority.md) on four nodes: the same admin stands at each node's gateway and sends a management action to every node. Logging (the default): all are allowed and each target logs what `enforce` would refuse, naming the gateway's node and its position. Enforcing: only the calls from the node itself or a controller above it are honoured, the rest are answered 403 with the reason, open actions work everywhere and the fabric stays whole |
| `provisioned_live_test.sh` | a production mesh on operator-issued identities (`tools/cresco-pki.sh`), PKI discovery, no shared secret |
| `upgrade_window_live_test.sh` | the [upgrade guide](../security/upgrade.md): earlier-build agents keep their membership in the window, are upgraded one by one, and the window closes by its date |
| `dataplane_backpressure_test.sh` | [backpressure](../architecture/dataplane.md#backpressure-nothing-lost-order-kept): fast and slow readers on one broker and across a bridge each way, every frame delivered whole, once and in order, the writer slowed to a slow reader, nothing discarded |
| `control_plane_bench.sh` | MsgEvent round-trip latency (p50/p90/p99) and concurrent rate, direct and across a bridge, per build and mode (`MODE=standard`, `nosign`, `nodirect`, `insecure`, `production`) |
| `many_agents_bench.sh` | membership at scale: N light agents started together, time until the global lists them all, controller CPU, refusals |
| `dgx/gate.sh` + `dgx/report.py` | the multi-host release gate: control plane, dataplane across hosts and scale, an earlier build against a candidate in both profiles ([results](performance.md)) |

The dataplane client they drive is the agent repository's `scripts/bench/dataplane_perf.py`: it streams frames through
wsapi (optionally across a broker bridge to another agent's wsapi), checks every frame (whole, once, in order), can read
slowly to exercise backpressure, and `--rate-mib-s` offers a fixed load to measure the pipeline's latency rather than the
depth of a queue a faster writer builds.

## Writing a new integration test

Follow the pattern of the existing scripts: source `cresco.env`, define a `launch()` helper that runs the
agent jar with role/port/secret flags into a per-node dir, wait for readiness (grep the node log for a
marker such as `Starting Bridge`), assert with a small `ck()` PASS/FAIL helper, then `pkill` and exit with
the fail count. Keep every new flag **default-off** so the suite stays green with the feature both off and
on.
