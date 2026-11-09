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

Agents not yet upgraded (an earlier 1.3 build) stamp nothing, sign nothing and run without tenant namespacing (they write
their controller's raw inbox `<region>_<agent>` and listen on their own). A secured broker stamps such a node
`agent:<tenant>/<region>/<agent>` with role `user` only. During an explicit, time-bounded window they keep their own
membership (join over the legacy exchange, register, unregister, keep alive, ping) and nothing more: everything else
they send meets the action tables as a `user`, and their dataplane use is refused by the broker (`DENY ... global.event`
lines for them are expected). Open the window on every upgraded node they talk to (global, regions, upgraded agents):

```
discovery_auth=compat
discovery_compat_until=2026-12-15      # an ISO date (end of that day, UTC) or instant, at most 90 days ahead
```

- Every boot logs `WARNING: UPGRADE WINDOW OPEN until ...` at ERROR level, repeated every hour while the window is open.
  Without `discovery_compat_until` (or with one more than 90 days ahead) compat is ignored, and an ERROR says so.
- **The window is decided when the node starts.** A date more than 90 days ahead stays ignored for good: it does not open
  by itself as the date comes nearer; a restart within 90 days of it opens it.
- When the date passes the window closes by itself, without a restart: legacy exchanges, unsigned messages and `agent:`
  registrations are refused from then on, and every boot logs that the window has CLOSED until the two settings are
  removed.
- Not available under `security_profile=production`.

While the window is open an upgraded controller also consumes its raw inbox (`AgentConsumer also consumes the raw inbox
[<region>_<agent>] during the upgrade window` at boot), answers earlier-build nodes there (the reply to a registration,
health pings), and answers a discovery locator in the format it was asked in. An earlier build keeps no identity key
across restarts, so its upgraded build presents a new key: the binding the legacy exchange made for that agent name is
replaced, once, by the key the upgraded agent proves over the authenticated exchange (`Discovery: <region>/<agent>
UPGRADED: ...`; the old key stays refused). A binding made by the authenticated exchange is never replaced this way.

## Order

Controllers first, from the top: a parent must understand everything its children send.

1. **Global**, with the window settings. Verify: no `REFUSING TO START`, `UPGRADE WINDOW OPEN`, `AgentConsumer also
   consumes the raw inbox`, and `Discovery: accepting the LEGACY unauthenticated certificate exchange` once its
   earlier-build agents rejoin; they register again.
2. **Regions**, one at a time, right after the global, with the same window settings. Each region rejoins its global over
   the authenticated discovery exchange (controllers never use the legacy exchange, so a region of the earlier build is
   not federated with an upgraded global until it is upgraded; its agents keep working with it). Verify per region:
   `Broker bridge from verified controller ...` at the global, and that its not-yet-upgraded agents re-register and appear
   in the global's `listagents`.
3. **Agents**, in any order, one at a time or in batches. The window settings are not needed on an agent. An upgraded
   agent joins over the authenticated exchange (`discovery v2: authenticated AGENT exchange` in its log, `<region>/<agent>
   UPGRADED` at its controller), registers, and its messages are authenticated from then on.
4. **Close the window**: once every node is upgraded, remove `discovery_auth=compat` and `discovery_compat_until` from
   every node at its next restart, or let the date pass. An earlier-build node still running when it closes is refused
   from that moment (`MESSAGE SIGNATURE: REFUSED an unsigned message (from agent:...)` at its controller), and one that
   starts later is refused at discovery (`Discovery: REFUSED a node of an earlier build`).

A node not upgraded before the window ends is cut off when it ends.

## Checking and rolling back

- `REFUSING TO START: ...` lines name every configuration problem (exit 78); `WARNING:` lines name every temporary or
  insecure setting; `MESSAGE SIGNATURE: REFUSED ...` names refused messages (see
  [Troubleshooting](../operations/troubleshooting.md)).
- `controller/docs/tests/upgrade_window_live_test.sh <upgraded agent.jar> <earlier agent.jar> <dir> <python>` proves this
  guide on one host: earlier-build agents join upgraded controllers in the window, are upgraded one by one (with and
  without the window settings), the window closes by its date while an earlier-build agent still runs, and the
  controllers restart without it.
- Rolling a node back is possible while the window is open around it. Rolling the whole fabric back to insecure operation
  needs `broker_security_enabled=false` on every node: never on a network you do not control.

## Keystores in FIPS mode

A node moved to the production profile runs FIPS approved-only, which reads no PKCS#12 keystore: convert its identity
keystore once (`java -jar agent.jar --crypto convert-keystore ...`, see [FIPS](fips.md)). wsapi's own managed keystore
from an earlier standard-profile run is refused there too: convert it, or delete it and its password file to make a new
one (clients then pin the new fingerprint).
