# Upgrading a fabric to the 1.3 secure defaults

For a fabric running an earlier 1.3 build (security off, or on without namespacing), upgraded without a flag day.
Source: `controller/docs/upgrade-1.3-secure-defaults.md`. Production deployments with operator-issued certificates
follow [Security profiles](profiles.md) and [Operator PKI](pki.md) instead.

## Before you start

1. **Tenant names.** A `tenant_id` with a `.` cannot be an identity; set `tenant_id_dot_replacement=-` (or a letter or
   digit) on every node of that tenant, and check no other tenant already has the resulting name.
2. **Plugins installed over the network** (`pluginadd` from a repository, `pluginupload`, `putjar`) must be signed: set
   `plugin_signing_truststore` on every node that installs them ([Signed plugins](signed-plugins.md)). Jars embedded in
   the agent jar are trusted as part of it.
3. **Remote file reads** (`getfiledata`, `getfileinfo`) need `file_access_roots`.
4. **Callers that are not controllers.** API users authenticate to wsapi ([API authentication](api-auth.md)); a plugin
   acts only as itself. Tools that sent MsgEvents with a forged `cresco_caller` are refused.
5. **Multi-tenant regions.** A region or global managing agents of several tenants must be in a superuser tenant
   (`broker_superuser_tenants`, default `cresco-system`).
6. **API clients.** wsapi's TLS certificate is verified by both clients by default (pycrescolib 1.4.0+, clientlib 1.3):
   give every client the fingerprint from `ws.fingerprint` or a copy of `ws.crt.pem`.

## The upgrade window

Agents not yet upgraded stamp and sign nothing. During an explicit, time-bounded window they keep their own membership
(join over the legacy exchange, register, unregister, keep alive, ping) and nothing more. Open it on every upgraded node
they talk to:

```
discovery_auth=compat
discovery_compat_until=2026-12-15      # an ISO date (end of that day, UTC) or instant, at most 90 days ahead
```

Every boot logs `WARNING: UPGRADE WINDOW OPEN until ...`; without `discovery_compat_until` (or more than 90 days ahead)
compat is ignored with an ERROR. When the date passes the window closes by itself. Not available in production.

## Order

1. **Global**, with the window settings: verify it starts, logs `message signatures ON` and `UPGRADE WINDOW OPEN`.
2. **Regions**, one at a time, with the window settings: `Broker bridge from verified controller ...` at the global,
   `registered with Global`, and its not-yet-upgraded agents re-register (`agent:` stamps are expected meanwhile).
3. **Agents**, in any order; an upgraded agent re-enrolls over the authenticated exchange and signs from then on.
4. **Close the window**: once every node is upgraded, remove `discovery_auth=compat` and `discovery_compat_until`.

A node not upgraded before the window ends is cut off when it ends.

## Checking and rolling back

- `REFUSING TO START: ...` lines name every configuration problem (exit 78); `WARNING:` lines name every temporary or
  insecure setting; `MESSAGE SIGNATURE: REFUSED ...` names refused messages.
- Rolling a node back is possible while the window is open around it. Rolling the whole fabric back to insecure operation
  needs `broker_security_enabled=false` on every node: never on a network you do not control.

## Keystores in FIPS mode

A node moved to the production profile runs FIPS approved-only, which reads no PKCS#12 keystore: convert its identity
keystore once (`java -jar agent.jar --crypto convert-keystore ...`, see [FIPS](fips.md)). wsapi's own managed keystore
from an earlier standard-profile run is refused there too: convert it, or delete it and its password file to make a new
one (clients then pin the new fingerprint).
