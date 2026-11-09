# Troubleshooting

What a node, wsapi or a client reports when something is refused or held back, what it means, and what to do. Every
refusal names its reason in the log; the lines below are quoted as the code writes them (`...` marks the variable part).

## A node does not start

**`REFUSING TO START: ...`** (exit code 78), one line per problem, before anything listens:

| Line says | Cause | Fix |
|---|---|---|
| `broker_security_enabled=true with broker_require_client_auth=false: ...` | client certificates turned off with security on | remove `broker_require_client_auth=false` (or set `broker_security_enabled=false` explicitly for an insecure debug node) |
| `broker_security_enabled=true with tenant_namespacing=false: ...` | namespacing turned off with security on | remove `tenant_namespacing=false` |
| `tenant_id '...' has a '.', which a tenant name may no longer carry ...` | the tenant name cannot be an identity | set `tenant_id_dot_replacement=-` on every node of that tenant ([upgrade](../security/upgrade.md)) |
| `security_profile=production: ... is not allowed` / `... is a password, which production does not allow` | a setting the production profile refuses | remove it ([profiles](../security/profiles.md)) |
| `security_profile=production: discovery_secret_... is set: production discovery is authenticated by the operator's PKI only ...` | a discovery secret left over from the standard profile | remove the setting, its `_file` and its `CRESCO_` variable |
| `security_profile=production: ... is required` / `... is not a readable file` | a required identity file is missing | provide `identity_keystore`, `identity_keystore_password_file`, `trust_anchors`, `plugin_signing_truststore` |
| `the pre-provisioned identity cannot be used: ...` | the keystore, its password file or the anchors do not validate (a PKCS#12 keystore under FIPS is refused with how to convert it) | fix the files; convert keystores with `java -jar agent.jar --crypto convert-keystore` ([FIPS](../security/fips.md)) |
| `security_profile=production runs in FIPS 140-3 approved-only mode, and this JVM is not in it: ...` | the JVM has a non-FIPS provider, Bouncy Castle FIPS is not approved-only or failed its self-tests, ACCP-FIPS is not healthy, or MD5 is available | start the node with the agent jar, whose crypto boot sets the mode up; do not add JCA providers |
| `... went BACKWARDS ...` | the `revocation_file` holds an older CRL of a trusted CA than one already in force | install the current CRLs (`cresco-pki.sh crl`) |

## Message signature refused

**`MESSAGE SIGNATURE: REFUSED <why> (<n> refused so far; logged at most every 10 s)`**: a message failed its end-to-end
authentication and was not dispatched; an RPC request is answered `401 unauthenticated`. See
[Message authentication](../security/message-auth.md).

| `<why>` | Meaning | What to check |
|---|---|---|
| `an unsigned message (from ..., not delivered first hand)` | an unsigned message that did not come straight from the receiver's parent or child | a node of an earlier build outside the upgrade window (`from agent:...`), or a sender with `msg_signing_required=false` |
| `a message signed for <r>/<a>, not for this node (...)` / `a message signed for plugin ..., delivered to ...` | the message names another recipient | a copy re-sent or misrouted; nothing to fix on the receiver |
| `a message that names no recipient (cresco_sig_to)` | a sender of a build before recipient binding | upgrade the sender |
| `a message for the global above ..., which is not a region controller of this global` | an agent's global function arrived at a global that is not its region's | the region's federation |
| `a signature made <n> s from now (msg_signature_max_age_ms=...)` | the signing time is more than 5 min off this node's clock, either way | clock synchronization (NTP) on both nodes |
| `a signature seen before (a replay) from ...` | the same signature or MAC arrived twice | a replay; nothing to fix on the receiver |
| `a message signed <n> s ago while the replay memory is full` / `a message while the replay memory is full` | more traffic within one max-age than `msg_replay_memory_max` entries | raise `msg_replay_memory_max` |
| `the signer's credential does not verify here (...)` | the signer's certificate or attestation does not chain to anything this node trusts | the trust bundle (has the global listed that region?), anchors, pins |
| `the signer ... is refused: its key is revoked` / `... is REVOKED by the operator (...)` / `... is not valid now (...)` | a revoked or expired signer or issuer, checked on every message | re-issue the node; refresh an out-of-date CRL (`revocation_file`) |
| `a message under a session key this node does not hold (...)` | this node restarted or the session ended; the sender returns to signatures by itself | nothing, unless it persists |
| `the signature does not verify` / `the batch signature does not verify` / `the session MAC does not verify` | the message was altered on the way | the path the message took |
| `the broker's caller ... is not the signed caller ...` / `... was signed by another node, ...` | the signer may not speak for the stamped caller | a forged or misattributed caller |

On the sending side: `cannot sign a message this node sends (...): it goes unsigned and is refused` means the node's own
key or provider failed.

## Discovery refusals

| Line | Meaning |
|---|---|
| `Discovery: REFUSED ... from ...: under security_profile=production discovery is authenticated by the operator's PKI only ...; is that node production?` | a standard-profile (secret-keyed) or legacy peer met a production node; they never join each other |
| `Discovery: REFUSED a node of an earlier build at ...` | an earlier build tried the legacy exchange outside the upgrade window |
| `discovery v2: ... failed authentication: it does not hold the ...` | the peer's discovery secret differs from this node's |
| `discovery v2: ... claims <r>/<a> but its certificate does not name it` | the peer's certificate does not match the identity it claims |
| `discovery v2: the result from ... failed authentication (altered or replayed)` | the exchange was tampered with on the way |
| `WARNING: UPGRADE WINDOW OPEN until ...` (ERROR level, every boot and hourly) | the upgrade window is open: remove `discovery_auth=compat` and `discovery_compat_until` once every node is upgraded |
| `the upgrade window has CLOSED (...)` | the window's date passed; remove the two settings |
| `discovery_auth=compat is IGNORED: it needs discovery_compat_until ...` | compat without a date at most 90 days ahead; the window is decided at start, so restart after fixing it |

## Dataplane: slow or stopped streams

With [backpressure](../architecture/dataplane.md#backpressure-nothing-lost-order-kept) (the default) a slow subscriber
slows its writers; nothing is lost.

| Symptom | Meaning | What to do |
|---|---|---|
| a writer's sends slow down or block | a subscriber of the same topic on that broker is slower than the writer, and the topic's memory (`activemq_topic_memory_limit`) is full | find the slow reader (`wsapi.dataplane.egress.paused`, `.pending.bytes`); a stopped reader is closed after 30 s |
| a WebSocket client closed with **1013** `stalled: nothing read in 30000 ms` | the client read nothing for `wsapi_dataplane_stall_close_ms` while frames waited for it, holding its topic's writers | read the stream continuously, or raise the setting; reconnect and resume (`cresco_seq` says where) |
| a WebSocket client closed with **1013** `too many message bytes in flight (...); retry shortly` | the caller (or all callers) sent more than `wsapi_max_inflight_bytes_per_user` / `wsapi_max_inflight_bytes` that is not yet handled | send less at once, then retry |
| **1008** `Subscription refused: ...` | the caller may not subscribe to that stream (e.g. another identity's recipient-bound stream) | the stream name and the caller's tenant and role |
| **1008** `credentials revoked or changed: authenticate again` | the connection's credential was revoked or changed | authenticate again |
| **1009** `message larger than <n> bytes (...)` | an inbound message over its size cap | send smaller messages |
| `bridge ...: <topic> stayed full with nothing taken for 10000 ms; delivering without waiting until it has room (kept pending here, nothing discarded; overflows total=<n>)` | two brokers full of each other's traffic, or every subscriber of a topic stuck | the stuck subscriber; the bridge waits again once the topic has room |
| `broker DISCARDED <n> message(s) for slow subscriber(s), ...` | only when backpressure is off (`dataplane_backpressure=false` or `activemq_topic_producer_flow_control=false`): a subscriber fell more than prefetch x 2.5 behind | turn backpressure back on, or speed the subscriber up |
| a stream message with `cresco_gap` = n, `dataplane.stream.gap` rising | n messages of that stream never reached this subscriber: a node or link died with messages on the way, the subscriber was cut off, or backpressure is off | recover or report from the gap; check the path's health |
| `DataPlaneWsHandler: the dataplane did not take a client frame ...` | wsapi could not publish a client frame (no session, or the controller not active); counted in `wsapi.dataplane.ingress.failed` | the node's controller and dataplane health |

## API clients

| Response | Meaning |
|---|---|
| RPC reply `401` | the request failed message authentication on the way (see above) |
| RPC reply `403` | the caller's role may not run that action |
| RPC reply `429 busy: ...` | the caller has `wsapi_rpc_max_inflight_user` (8) / `_admin` (32) RPCs in flight |
| RPC reply `503 busy: wsapi has 640 ...` | wsapi's RPC pool is full |
| HTTP `503` at login | the node runs production and this wsapi did not start in production ([profiles](../security/profiles.md)) |

## Plugins

| Line | Meaning |
|---|---|
| `addBundle: REFUSED to install <jar>: ...` | the jar does not verify against `plugin_signing_truststore` ([Signed plugins](../security/signed-plugins.md)) |
| `wsapi NOT started: ...` | wsapi's own `security_profile` conflicts with the node's, or production requirements are missing (`wsapi_client_ca` ...) |
| `filerepo <name>: no scan and no sync: ...` | a filerepo over a directory another filerepo of the agent already serves, or whose catalog did not open: that plugin serves nothing (one ERROR at start) |
| `STRIPPED a forged caller '...' on a message to '...'` | a sender set a caller stamp it may not declare; the stamp was removed and counted |
