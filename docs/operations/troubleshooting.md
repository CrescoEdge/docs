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

**`MESSAGE SIGNATURE: REFUSED <why> [<type> <action>] (<n> refused so far; <k> of this kind, <reason>; each kind is
logged at most every 10 s)`**: a message failed its end-to-end authentication and was not dispatched; an RPC request is
answered `401 unauthenticated`. The bracket names the refused message. `<n>` is the node's total, `<k>` the count of
this reason. Each reason is logged at its first occurrence and then at most every 10 s by itself, so a reason that
happens all the time never hides one that happened once; every reason is also a counter in the node's metric inventory
(`msg.refused.<reason>`, see [Metrics](../architecture/metrics.md#controller-what-became-of-each-message)). (Builds
before 2026-11-11 kept one count and one line per 10 s for all reasons together.) See
[Message authentication](../security/message-auth.md).

The reason names: `unsigned`, `not_for_this_node`, `no_recipient`, `too_old`, `ahead_of_clock`, `replay`,
`replay_memory_full`, `credential`, `revoked`, `session_key`, `bad_signature`, `caller_mismatch`, `other`.

| `<why>` | Meaning | What to check |
|---|---|---|
| `an unsigned message (from ..., not delivered first hand)` | an unsigned message that did not come straight from the receiver's parent or child | a node of an earlier build outside the upgrade window (`from agent:...`), or a sender with `msg_signing_required=false`. From a parent of a build before 2026-11-11, for up to 60 s after this node moved to another controller, or `[EXEC ping]` once at a restarted global: a Cresco defect (a direct link was kept a minute after the link was gone); upgrade the sender |
| `a message signed for <r>/<a>, not for this node (...)` / `a message signed for plugin ..., delivered to ...` | the message reached a node other than the one it is for, and was not run there | on a fabric of two or more regions, `[EXEC ping]` about every 45 s at the global and at each regional controller came from a Cresco defect (fixed 2026-11-10: controller 21b432c, agent daeb294): each regional controller's path probe "via the global" was sent with source routing off, the global did not relay it, and its 401 reply was signed for the wrong node. Upgrade the regional controllers and the global. Any other message: the sender or a relay routed it to the wrong node; report it with the type and action from the bracket |
| `a message that names no recipient (cresco_sig_to)` | a sender of a build before recipient binding | upgrade the sender |
| `a message for the global above ..., which is not a region controller of this global` | an agent's global function arrived at a global that is not its region's | the region's federation |
| `a signature made <n> s ago (msg_signature_max_age_ms=...)` | the message is older than 5 min by this node's clock | clock synchronization (NTP) on both nodes; a message delayed that long is refused by design |
| `a signature made <n> s ahead of this node's clock (msg_signature_max_future_ms=...)` | the sender's clock is more than 60 s ahead of this node's | clock synchronization (NTP) on both nodes; raise `msg_signature_max_future_ms` only where clocks cannot be kept closer |
| `a signature seen before (a replay) from ...` | the same signature or MAC arrived twice | a replay; nothing to fix on the receiver |
| `a message signed <n> s ago while the replay memory for <signer> is full` / `a message while the replay memory for <signer> (or of this node) is full` | that one signer sent more within one max-age than its memory holds (an eighth of `msg_replay_memory_max`), or all signers together more than `msg_replay_memory_max` (about 10,000 signed or MAC'd messages a second at the default). The node then gives up its oldest signing times: messages signed longer ago than it still keeps are refused, fresh ones are accepted; the log line `the replay memory is full ... messages signed before ... are now refused as too old` says from when. Only that signer is refused in the first case | if the traffic is expected, raise `msg_replay_memory_max` (size it at 8 x the busiest signer's messages per max-age, and at the node's whole signed traffic per max-age); otherwise look at that signer |
| `the signer's credential does not verify here (...)` | the signer's certificate or attestation does not chain to anything this node trusts. A message whose signer is only not known here YET (its controller, or that controller's present key, is not in the trust bundle held) is first held up to `msg_signing_hold_ms` (5 s) and verified again when the trust data changes; this line then means it still did not verify | the trust bundle (has the global listed that region?), anchors, pins |
| `the signer ... is refused: its key is revoked` / `... is REVOKED by the operator (...)` / `... is not valid now (...)` | a revoked or expired signer or issuer, checked on every message | re-issue the node; refresh an out-of-date CRL (`revocation_file`) |
| `a message under a session key this node does not hold (...)` | this node restarted or the session ended; the sender returns to signatures by itself | nothing, unless it persists |
| `the signature does not verify` / `the batch signature does not verify` / `the session MAC does not verify` | the message was altered on the way | the path the message took |
| `the broker's caller ... is not the signed caller ...` / `... was signed by another node, ...` | the signer may not speak for the stamped caller | a forged or misattributed caller |

On the sending side: `cannot sign a message this node sends (...): it goes unsigned and is refused` means the node's own
key or provider failed.

A node's first moments: its trust bundle, an agent's attestation and its parent's revocations arrive on its parent's
first pong, which it asks for the moment it joins. Until it holds them -- the whole revocation list included, when that
is longer than one pong carries -- it holds every signed message but its parent's own (15 s at most), and verifies them
when the data is in; nothing is refused for arriving early, and nothing is dispatched unverified. A node that later
meets a signer it cannot judge (a region that joined a moment ago) asks its parent for the trust bundle at once instead
of waiting for the next ping.

`route: dropped a message routed more than 10 times or with a ttl outside 0..10 (ttl=...)`: a routing loop, or a sender
that wrote a `ttl` no router writes. The message ends at that router; an API client's `ttl` and `is_rpc` are removed by
wsapi.

## Message dropped

**`MESSAGE DROPPED: <cause in words> [<type> <action>] <src> -> <dst> (<n> dropped so far; <k> of this kind, <cause>;
each kind is logged at most every 10 s)`**: a control message ended on this node without reaching its recipient. Every
cause is logged at its first occurrence and then at most every 10 s, and is a counter (`msg.dropped.<cause>`). Before
2026-11-11 most of these were not counted at all, and one of them logged an ERROR line per message.

| `<cause>` | Meaning | What to check |
|---|---|---|
| `ttl` | routed more than 10 times, or with a hop count no router writes | a routing loop: report it with the type and action |
| `no_global` | for the global, at a region that has joined none | the region's link to its global |
| `parent_unknown`, `parent_link_down`, `no_sender` | the node had no parent, no link to its broker, or no producer yet when it had to send | expected for a moment around a restart of the node or its parent (`no_sender` in a node's first milliseconds; `parent_link_down` after a message was held up to `msgin_fault_wait_ms`, 20 s, for the link, or, from agent release 1.3.377, until its sender stopped waiting); persistent: the link |
| `controller_not_ready`, `controller_stopping`, `node_stopping` | for this node's controller while it was being rebuilt or shut down | expected at shutdown and during a re-join |
| `plugin_unknown`, `plugin_inactive`, `plugin_admin_unavailable` | for a plugin this node does not run, or that is not active yet | a sender with a stale address; `plugin_inactive` while a plugin loads (a storage core loading its log) is expected |
| `source_route_not_relayed` | a source-routed message at a node with `net_source_routing` off | the sender's routing settings |
| `tier_refused`, `misrouted`, `cross_tenant` | a controller function at the wrong tier, a message at a controller that is addressed elsewhere, a caller's message in another tenant's namespace | the sender; report it |
| `send_failed`, `no_destination`, `route_error`, `inbox_error` | the broker did not take it after every attempt, it named no destination, or this node failed on it | the broker's health and the ERROR line logged with it |
| `file_group_refused`, `file_group_expired` | a message with files whose group was not registered, or whose parts never all arrived | the sender of the transfer; see the file group lines below |
| `hold_closed` | held by an inbox that closed | expected when a node stops or re-joins in its first seconds |

A node also counts what it originated, relayed and delivered. On a quiet fabric the messages originated, summed over
the nodes, equal those delivered, refused and dropped, summed over the nodes: a sum that falls short means messages
were lost in a broker (expired, or their queue had no consumer).

## A global or a region controller is down

**The global is down.** Each region goes to `REGION_GLOBAL_FAILED` and keeps serving its own agents: its broker, its
discovery listener, the aging of its agents (STALE and LOST on schedule), its bridges to configured peer regions and the
dataplane all stay up. Pings and watchdogs are answered, state exports are applied, and a new or restarted agent can
discover the region and register. A request for the global is answered `no global controller joined`. The region retries
the join every few seconds; when the global is back it is `REGION_GLOBAL` again and sends its full state at once. Agents
do nothing: no reconnect, no new registration. (Builds before 2026-11-11 stopped the region's discovery and its agent
watcher for the whole outage: no agent could join or rejoin, and none was aged.)

With `global_optional=true` a region gives up after `global_connect_attempts`, runs as `REGION` (active), and keeps trying
in the background (5 s apart at first, backing off to 60 s) while a global is configured (`global_controller_host`) or one
found by discovery was joined before.

**A region controller is down.** Its agents keep their state and reconnect when it is back; agents of other regions and
the global are not affected.

| Line | Meaning |
|---|---|
| `timer task <name> failed (it keeps its schedule): ...` | one run of a periodic task (a watchdog, the agent aging, a ping) threw; the task goes on at its next interval. Before 2026-11-11 the first such failure ended that timer silently, and with it every watchdog on it |
| `REFUSED the file group ...` | an incoming file transfer was not registered: the receiver already holds 256 groups (64 for one caller), or the group id is held by another caller |
| `EXPIRED the file group ...` | a file transfer received no part for 30 min: its parts were deleted and its group forgotten |

## A node's parent refuses it ("PARENT REFUSES THIS NODE")

```
PARENT REFUSES THIS NODE: rgr_rc answered this node's ping with a refusal (...): 4 in a row, for 20 s. The link is up
and is NOT torn down; nothing that came with the refusal is taken ...
```

The node reaches its parent's broker, but the parent does not accept what it signs: a refusal is not a pong. The node
stays connected (it does not rejoin in a loop), its `link:parent` health reads WARN, and the line repeats at most once a
minute. The parent's own log names the reason (`MESSAGE SIGNATURE: REFUSED ...`, see above). The usual ones:

| Reason at the parent | Cure |
|---|---|
| the node's credential does not verify there (it is not attested, or its certificate is not the one the parent holds for its name) | restart the node so that it joins again, or authorise its key at the parent (a rebind in the standard profile; a certificate of the operator's PKI in production) |
| the time of its signatures | set the node's clock |
| its key is revoked | intended: the node is out |

After its region controller came back with a **new key**, an agent of release 1.3.378 or later recovers by itself (it
drops the attestation the former key made and is attested anew within a second). Agents of earlier releases under a
re-keyed controller must be restarted.

## A management call is logged as "AUTHORITY: WOULD REFUSE"

```
AUTHORITY: WOULD REFUSE CONFIG pluginadd from api:acme/alice[admin], the word of acme/rgA/a1 (ELSEWHERE): it is honoured
only from the node itself or a controller above it; allowed (authority_direction=log); ...
```

The call was allowed. It came through a node that is neither the target nor a controller above it, which
`authority_direction=enforce` will refuse. "The word of" names the node that spoke for the caller: for an API user, the
gateway the user is connected to. Move the client to the target's own gateway, its region controller's, or the global's.
Each line is written once per action and position; every occurrence is counted in `authz.direction.would_refuse` (and
`.would_refuse.<action>`). See [Authority follows the tree](../security/authority.md).

## Discovery refusals

| Line | Meaning |
|---|---|
| `Discovery: REFUSED ... from ...: under security_profile=production discovery is authenticated by the operator's PKI only ...; is that node production?` | a standard-profile (secret-keyed) or legacy peer met a production node; they never join each other |
| `Discovery: REFUSED a node of an earlier build at ...` | an earlier build tried the legacy exchange outside the upgrade window |
| `Discovery REFUSED controller <r>/<a>: it claims the superuser tenant ..., and a controller of a superuser tenant is never bound first come: list its key ... in controller_trust_anchors on this node` | a region or global controller whose certificate names a superuser tenant (`broker_superuser_tenants`) met a node that does not list it. Its tenant is its own certificate's word, and a superuser tenant's controller is SUPERUSER at every broker that binds it: add the key the line names to `controller_trust_anchors` on this node (and see [upgrade](../security/upgrade.md), item 5) |
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
| a WebSocket client closed with **1013** `stalled: <n> B read in 30000 ms, less than <m>` | while frames waited for it the client read less than half its egress budget in `wsapi_dataplane_stall_close_ms` (nothing, or a trickle), holding its topic's writers | read the stream continuously, or raise the setting; reconnect and resume (`cresco_seq` says where) |
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
| RPC reply `403 ... is honoured only from the node itself or a controller above it` | the action is a management action and the node you are connected to is neither its target nor above it (`authority_direction=enforce`): connect to the target's own gateway, its region controller's or the global's ([authority follows the tree](../security/authority.md)) |
| a reply with `gateway_status` | the gateway you are connected to gave this answer itself (400 unreadable or malformed request, 403 refused here, 429 or 503 busy, 504 no reply in time, 500 the send failed); without it, the status is the answer of the node you asked |
| RPC reply `400 unreadable request` | the frame was not a JSON object of `message_info` and `message_payload`; the gateway closes a connection at its third |
| RPC reply `429 busy: ...` | the caller has `wsapi_rpc_max_inflight_user` (8) / `_admin` (32) RPCs in flight |
| RPC reply `503 busy: wsapi has 640 ...` | wsapi's RPC pool is full |
| HTTP `503` at login | the node runs production and this wsapi did not start in production ([profiles](../security/profiles.md)) |

## Plugins

| Line | Meaning |
|---|---|
| `addBundle: REFUSED to install <jar>: ...` | the jar does not verify against `plugin_signing_truststore` ([Signed plugins](../security/signed-plugins.md)) |
| `addConfig REFUSED: plugin id <id> is already configured on this agent` | a second `pluginadd` with an `inode_id` the agent already holds (a retry after a timeout, a pipeline resubmitted); the add fails at once | remove the plugin first, or add under another id |
| `wsapi NOT started: ...` | wsapi's own `security_profile` conflicts with the node's, or production requirements are missing (`wsapi_client_ca` ...) |
| `filerepo <name>: no scan and no sync: ...` | a filerepo over a directory another filerepo of the agent already serves, or whose catalog did not open: that plugin serves nothing (one ERROR at start) |
| `STRIPPED a forged caller '...' on a message to '...'` | a sender set a caller stamp it may not declare; the stamp was removed and counted |
| `WITHHELD from a bridge a message on '...': the bridge of ... carries its own tenant's and region's traffic only (...)` / `DISCARDED a message a bridge brought in for '...': ...` | a bridge to another controller demanded, or brought in, a message that is not its peer's own: another tenant's destination. The message stays on (or never enters) this broker; the bridge itself stays up. A region below a global and a peer region carry their own tenant's subtree, what is addressed to an inbox of their own region, and answers to a superuser tenant's nodes; a region's global and a superuser tenant's controller carry everything. Expected when tenants share a broker mesh; from a single-tenant fabric, report it |
