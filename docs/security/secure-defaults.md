# Secure defaults

Source: `controller/docs/caller-authorization.md` section 11 and `controller/docs/upgrade-1.3-secure-defaults.md`.

## What a node does with no security setting

| Setting | Before 1.3 hardening | Now |
|---|---|---|
| `broker_security_enabled` | false | **true** (false only when set explicitly: insecure debug mode, a WARNING every boot) |
| `broker_require_client_auth` | follows security | follows security, so true |
| `tenant_namespacing` | false | follows `broker_security_enabled`, so true |
| `security_profile` | (none) | `standard`; `production` available |
| messages | unsigned | signed by their origin node and checked before dispatch (`msg_signing_required`) |
| `discovery_auth=compat` | open-ended | only inside `discovery_compat_until` (at most 90 days ahead) |
| `tenant_id` with a `.` | accepted | refused unless `tenant_id_dot_replacement` is set |
| plugin installs over the network | unchecked | only jars that verify against `plugin_signing_truststore` |
| remote file reads (`getfiledata`, `getfileinfo`) | anywhere | only below `file_access_roots` (unset: refused) |

## What a node refuses

A node **refuses to start** (exit code 78, one `REFUSING TO START` line per problem) when security is on but
`broker_require_client_auth=false` or `tenant_namespacing=false` is set, when its tenant name cannot be an identity, or
when its `security_profile` forbids a setting. Remove the offending setting, or set `broker_security_enabled=false`
explicitly for an insecure debug node.

At run time:

- a message whose caller stamp a node may not declare is stripped and counted (`STRIPPED a forged caller ...`);
- an unsigned message, a bad signature, a signature older or newer than `msg_signature_max_age_ms` (5 min) or a replay
  is refused (`MESSAGE SIGNATURE: REFUSED ...`; an RPC request is answered `401 unauthenticated`);
- a plugin jar that does not verify is not installed (`REFUSED to install`);
- a dataplane subscription that names another identity's recipient-bound stream, or a process's stdin, is refused.

## Checking a node

- No `REFUSING TO START` lines; `WARNING:` lines at boot name every temporary or insecure setting in effect.
- `message signatures ON: this node signs what it originates` and `... checks what it dispatches` once each.
- `Broker mutual-TLS ENABLED`, and `Peer identity verified` for each broker connection and bridge.
- In the production profile, `crypto: FIPS 140-3 approved-only mode: ... MD5 refused` (see [FIPS](fips.md)).
