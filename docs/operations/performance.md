# Performance

Measured numbers for the 1.3 release, from the multi-host release gate. Every number here was measured; nothing is
estimated.

## The release gate (2026-11-09)

The gate runs the same harness against the previous build and the release, on 4 DGX nodes:

- node 1: the global (with wsapi);
- node 2: a region, bridged to the global;
- node 3: an agent behind the region;
- node 4: the agent the benchmark clients run on.

Three builds were measured:

| Build | What it is |
|---|---|
| old | the previous 1.3 agent (`5f28a0a`) |
| std | the 1.3 release, `security_profile=standard` |
| prod | the same release, `security_profile=production` (operator PKI, client certificates only on the API, FIPS 140-3 crypto) |

The harness is `controller/docs/tests/dgx/gate.sh`, and `report.py` beside it compares the builds. A line passes when
throughput is no more than 10% below the old build and p99 latency no more than 20% above it. **Every line passed.**

### Control plane

RPCs through wsapi to each target, one caller at a time (serial) and with several callers at once.

| | old | std | prod |
|---|---|---|---|
| global, serial, p99 | 2.15 ms | 2.33 ms | 2.36 ms |
| global, 32 callers | 3755 msg/s | 3831 msg/s | 3845 msg/s |
| region, 32 callers | 3500 msg/s | 3656 msg/s | 3622 msg/s |
| agent behind the region, 8 callers | 1194 msg/s | 1113 msg/s | 1204 msg/s |

### Data plane

Streams across hosts through wsapi with the Python client (`scripts/bench/dataplane_perf.py` in the agent repository):
from the bench agent through the global, across the bridge to the region, to the agent behind it. "Paced" is a run at a
fixed offered load of 100 MiB/s per flow, which measures latency instead of throughput.

| | old | std | prod |
|---|---|---|---|
| 256 KiB frames, 1 flow | 323 MiB/s | 341 MiB/s | 334 MiB/s |
| 256 KiB frames, 1 flow, paced, p99 | 6.12 ms | 6.71 ms | 6.78 ms |
| 256 KiB frames, 4 flows | 594 MiB/s | 731 MiB/s | 738 MiB/s |
| 4 MiB frames, 1 flow | 320 MiB/s | 340 MiB/s | 346 MiB/s |
| 4 MiB frames, 4 flows | 602 MiB/s | 766 MiB/s | 725 MiB/s |

In every run, 0 frames were lost, damaged or delivered out of order. The 4-flow lines are from a re-run with a fixed
harness.

### Scale

Light agents (no plugins) were started together in steps of 50, 100 and 200, on the global and on the region, while a
probe kept calling the global.

| | old | std | prod |
|---|---|---|---|
| all 200 agents listed by the global after | 49.9 s | 49.9 s | 17.4 s |

- Every agent registered at every step.
- On the release, the cost of one control-plane call stayed flat from 50 to 200 agents (the gate's test: the global's
  serial p50 at 200 agents within 1.25x of its value at 50, and the 8-caller rate to the agent behind the region within
  0.8x).
- The global kept answering during every startup storm: 0 errors, and the longest gap between answers was 0.09 s.

### Logs

The release logged 0 refusal or loss lines across the gate. The old build logged 18.

## Other measurements

- Message authentication cost, one host: [Message authentication](../security/message-auth.md#cost).
- Crypto throughput per mode, and dataplane throughput on one DGX node: [FIPS 140-3](../security/fips.md#throughput).
