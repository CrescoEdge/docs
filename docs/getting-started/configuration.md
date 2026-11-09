# Configuration

Cresco nodes are configured with **`-D<name>=<value>` JVM system properties** (and/or a config map),
read through a typed `Config` helper. This page explains how configuration works and the flags you reach
for most; the exhaustive list is the **[Configuration Parameters reference](../reference/configuration.md)**
(180+ parameters).

## How it works

Every setting is read through `plugin.getConfig().get<Type>Param("name", default)`:

```java
int brokerPort   = plugin.getConfig().getIntegerParam("broker_port", 32010);
boolean secure   = plugin.getConfig().getBooleanParam("broker_security_enabled", true);
String tenant    = plugin.getConfig().getStringParam("tenant_id", "default");
```

Resolution order is **system property → config map → baked-in default**. Two principles hold throughout the
codebase:

- **Security is on by default (1.3).** A node with no security setting runs secured: broker authorization, mutual TLS,
  tenant namespacing, signed messages and signed plugin installs ([Secure defaults](../security/secure-defaults.md));
  insecure operation is only an explicit `broker_security_enabled=false` debug mode.
- **Other new behavior is flag-gated.** Auto-tuning, sharding and the like are opt-in, so you can roll them out
  incrementally.

Set a property on the launch command line:

```bash
java -Dbroker_port=32020 -Dtenant_id=tenantA -jar agent-1.3-SNAPSHOT.jar
```

## Node role

The role flags select which tier a node runs as (see [Architecture](../architecture/overview.md)):

| Flag | Effect |
|------|--------|
| `is_global=true` | Run as a global controller (also its own region + agent). |
| `is_region=true` + `global_controller_host=<host>` | Run as a regional controller that federates up to a global. |
| `is_agent=true` + `regional_controller_host=<host>` | Run as a leaf agent that connects to a region's broker. |
| *(none)* | Standalone. |
| `regionname` / `agentname` | This node's region and agent names (its identity). |

## Networking

| Flag | Default | Purpose |
|------|---------|---------|
| `broker_port` | `32010` | this node's embedded ActiveMQ broker |
| `discovery_port` | `32010` | broker port used to **connect** to a parent controller |
| `netdiscoveryport` | `32005` | this node's discovery listener |
| `port` | `8080` | Felix console / HTTP |
| `enable_wsapi` | varies | run the [wsapi](../plugins/wsapi.md) WebSocket endpoint (`:8282`) for clients |

## Security & tenancy

Security is **on by default**: a node with no setting runs with mutual TLS, the authorization broker, tenant
namespacing, message authentication and signed plugin installs ([Secure defaults](../security/secure-defaults.md)).
The `production` profile adds operator-issued identities and FIPS 140-3 crypto ([Security profiles](../security/profiles.md)).

| Flag | Default | Purpose |
|------|---------|---------|
| `security_profile` | `standard` | `production`: operator PKI, every security setting mandatory, no passwords or discovery secrets, FIPS approved-only |
| `discovery_secret_agent` / `_region` / `_global` | *(generated)* | standard profile: pre-shared join secret per tier, required to join the mesh (refused in production) |
| `cresco_service_key` | — | wsapi's legacy service key (an `admin` of the agent's tenant); refused in production |
| `tenant_id` | `default` | this node's tenant (bound into its cert DN as `O=`) |
| `security_regional_ca` | `false` | standard profile: regional-CA enrollment → O(regions) distributed trust |
| `broker_security_enabled` | `true` | the tenant-authorization broker plugin (the ACL); `false` is insecure debug mode |
| `broker_require_client_auth` | follows security (`true`) | broker demands a client cert (mutual TLS); binds identity from the DN |
| `tenant_namespacing` | follows security (`true`) | qualify MsgEvent inboxes + dataplane topics `T.<tenant>.*` for end-to-end isolation |
| `msg_signing_required` | `true` | authenticate every message end to end ([Message authentication](../security/message-auth.md)) |
| `broker_superuser_tenants` | `cresco-system` | tenants whose network clients get the cross-tenant `SUPERUSER` role |
| `crypto_policy`, `crypto_allow_<hash\|sig\|kdf\|cipher\|wrap>` | `cnsa1` | **global controller only**: the fabric-wide crypto policy (allowed algorithm types for anything new), carried to every node; see the [reference](../reference/configuration.md#fabric-crypto-policy-global-controller-only) |

!!! danger "Discovery secrets default to random UUIDs"
    In the standard profile, if you do not set `discovery_secret_*`, a node generates random secrets — other nodes
    cannot join. For a real fabric set these to fixed shared values, from owner-only `<name>_file` files or
    `CRESCO_<NAME>` environment variables rather than `-D` (see
    [Secrets](../reference/configuration.md#secrets-and-fatal-errors)).

## Everything else

For the complete, categorized list — broker/transport tuning, failover, certificates, discovery timeouts,
health checks, link-quality metrics, auto-tuning, data-plane sharding, producer/QoS, database, and the
`repo`/`wsapi`/`stunnel` plugin settings — see the **[Configuration Parameters reference](../reference/configuration.md)**.
