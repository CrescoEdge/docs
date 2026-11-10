# Deployment

This page covers running Cresco beyond a single dev node: multi-host meshes, secured/multi-tenant fabrics,
and operational hygiene. For first bring-up see the [Quickstart](../getting-started/quickstart.md).

## Topology building blocks

| You want | Run | Notes |
|----------|-----|-------|
| A self-contained fabric | one **global** | acts as its own region + agent; exposes wsapi `:8282` |
| Scale within a site | globals + **regions** | each region runs its own broker; agents attach locally, region federates up |
| Leaf workloads | **agents** | no broker; attach directly to a region's broker |
| Redundancy / scale-out | multiple **globals** | globals federate to each other (multi-global mesh) |

Each node needs a unique `regionname`/`agentname`. Regions and globals need reachable
[discovery](../architecture/discovery.md) + broker ports.

## Ports

| Port (default) | Service | Who needs it |
|----------------|---------|--------------|
| `32005` | discovery listener | parents (region/global) |
| `32010` | ActiveMQ broker | parents; children connect here |
| `8282` | wsapi (`wss://`) | globals that serve clients |
| `8080` | Felix web console (`enable_console=true` only; loopback by default) | optional |

On a **single host**, give each node distinct `broker_port` / `netdiscoveryport` and its own working
directory (the launch scripts do this — `run/nodes/<region>-<agent>/`).

## One node, one directory

Every node keeps on-disk state (Felix cache, Derby DB, ActiveMQ journal) under `cresco-data/` in its
working directory. **Two nodes must never share a directory** or they corrupt each other's state. Run each
from its own directory.

## Securing a fabric

Security is **on by default**: mutual TLS, the authorization broker, tenant namespacing, end-to-end
[message authentication](../security/message-auth.md) and signed plugin installs need no flag
([Secure defaults](../security/secure-defaults.md)). What a deployment supplies depends on its
[profile](../security/profiles.md):

```bash
# standard profile: shared join secrets (owner-only files or CRESCO_<NAME>, not -D), the tenant
CRESCO_DISCOVERY_SECRET_GLOBAL_FILE=/etc/cresco/dg CRESCO_DISCOVERY_SECRET_REGION_FILE=/etc/cresco/dr \
CRESCO_DISCOVERY_SECRET_AGENT_FILE=/etc/cresco/da java -Dtenant_id=<tenant> ... -jar agent.jar

# production profile: the operator's PKI, no shared secrets, FIPS approved-only
java -Dsecurity_profile=production -Didentity_keystore=/etc/cresco/node.bcfks \
     -Didentity_keystore_password_file=/etc/cresco/node.p12.pass -Dtrust_anchors=/etc/cresco/trust-anchors.pem \
     -Dplugin_signing_truststore=/etc/cresco/codesign-anchors.pem -Dtenant_id=<tenant> ... -jar agent.jar
```

Production is described in [Security profiles](../security/profiles.md) and [Operator PKI](../security/pki.md); a fabric
of an earlier 1.3 build is upgraded without a flag day with [Upgrading to 1.3](../security/upgrade.md). Validate with
the [test harnesses](testing.md); refusals and their fixes are in [Troubleshooting](troubleshooting.md).

## Data-plane backpressure

A slow dataplane subscriber slows the writers of its topic instead of losing messages
([Data Plane](../architecture/dataplane.md#backpressure-nothing-lost-order-kept)). Two things to plan for:

- Each topic holds up to `activemq_topic_memory_limit` (256 MB) per broker for its slowest subscriber before its
  writers wait; size the broker's heap (`broker_memory_limit` defaults to max(512 MB, heap / 2)) for the topics that
  carry streams at once.
- A subscriber that stops reading holds every writer of its topic on that broker. wsapi closes such a WebSocket client
  after `wsapi_dataplane_stall_close_ms` (30 s, status 1013); a plugin's own `PacedListener` must give up the same way.

## Multi-global meshes

Multiple globals can coexist and federate for redundancy and scale. On the same host, distinguish them with
distinct ports (`global_controller_port` / `discovery_port_remote`) and stagger their launches so discovery
settles. This has been exercised at mesh scale (see the [design docs](../reference/design-docs.md)).

## Operational hygiene

- **Reset state between test runs / after rebuilds:** `rm -rf run/cresco-data run/nodes`.
- **Shut down cleanly:** `run/stop.sh` (or `pkill -9 -f agent-1.3-SNAPSHOT.jar`).
- **Watch health:** the [health](../architecture/health.md) subsystem logs a periodic rollup; a stale
  parent link self-heals via re-discovery.
- **Watch metrics:** pull `getmetricinventory` at `region`/`global` scope for a mesh-wide view.
