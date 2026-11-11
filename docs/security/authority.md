# Authority Follows the Tree

Roles and tenants say **who** may call an action. This page is about **from where**.

Every node's controller holds the `infra` role for its whole tenant, and any node's gateway (wsapi) may vouch for an
API user with `admin` of its tenant. With roles and tenants alone, one leaf agent that is taken over could add plugins
to its region controller, read files and logs of its siblings, stop the global, and run commands on all of them through
the executor plugin. Plugin signing does not help against a signed plugin that runs processes.

So a management action is honoured only when it comes **from the node itself or from a controller above it**. A leaf
manages nothing but itself.

!!! note "Logging only, for now"
    From agent release 1.3.377 on, a node runs the rule in `log` mode on every node and in every security profile: a call the rule would refuse
    is still allowed, and it is counted and logged. Nothing a fabric does today stops working. `enforce` is a setting
    (`authority_direction`), and becomes the default in a later release, after one release in which every deployment
    can read its counters. The live test `authority_live_test.sh` ([Testing](../operations/testing.md)) runs both modes
    on four nodes.

## The rule

Every message has a **speaker**: the node whose word the caller is.

| Caller | Speaker |
|---|---|
| a node's controller, or a plugin | that node (only that node's key signs for it, see [Message Authentication](message-auth.md)) |
| an API user (`api:<tenant>/<user>`) | the node that vouched for the user: the node whose gateway the user is connected to. A node that only carries the message on is never the speaker. |

The receiving node works out where the speaker stands relative to itself:

| Position | Seen by an agent | Seen by a region controller | Seen by the global |
|---|---|---|---|
| itself | itself | itself | itself |
| above | its region controller, and the global above its region | the global it joined | nobody |
| below | nobody | every other node of its region | every other node of its own region, and every node of a region whose controller joined it |
| elsewhere | a sibling, any other node | any other node | a node of a region that has not joined |
| unknown | no speaker can be named | | |

And every action has a **direction**:

| Direction | Honoured when the speaker is | Actions |
|---|---|---|
| down (management) | the node itself, or above | adding, removing, uploading and pulling plugins; updating, stopping and restarting a controller; `killjvm`; log level and log streaming; `getlog`, `getfileinfo`, `getfiledata`; `getagentinfo`, `getnetworkstate`, `getcoordinators`, `getbroadcastdiscovery`; the event-processing actions `cepadd`, `cepremove`, `cepinput`; pipelines (`gpipelinesubmit`, `gpipelineremove`, `rpipelinesubmit`); the global's repository and scheduling actions (`addplugin`, `removeplugin`, `plugindownload`, `savetorepo`, `setinodestatus`); and every privileged action of the executor, filerepo and repo plugins (`run_process` among them) |
| up (a child reporting) | the node itself, or below | `agent_enable`, `agent_disable`, `region_enable`, `region_disable`, `regionalimport`, every WATCHDOG and KPI message |
| any | anyone (role and tenant only, as before) | `ping`, status, lists, inventories, the global's registry reads, discovery and error messages |

Role and tenant are checked first and are unchanged. Direction only ever refuses more.

A message a region controller carries on for one of its agents keeps the agent as its speaker: carrying a message lends
it no authority.

## What this means for an administrator

A management call must enter the fabric **at the node it manages, or at a controller above that node**:

| Connect to the gateway of | To manage |
|---|---|
| the global | anything: every region controller and every agent; pipelines, the plugin repository, scheduling |
| a region controller | that controller and the agents of its region |
| a leaf agent | that agent only |

A client connected to a leaf can still read the fabric from there: lists, inventories and status are open as before.

!!! warning "A controller's gateway credentials must not exist on its leaves"
    The rule is as strong as the gateways' credentials are separate. If the global's service key (or one of its users'
    passwords, or a client certificate its gateway accepts) is also in the configuration of a leaf, whoever takes that
    leaf over logs in at the global's gateway with it and is vouched for from the top of the tree. Give every
    controller's gateway its own credentials, and keep them off the nodes below it. See [API Authentication](api-auth.md).

## The setting

`authority_direction`, set like every other setting ([Configuration](../getting-started/configuration.md): a
`-D` flag or the node's configuration):

| Value | What a node does with a call the rule would refuse |
|---|---|
| `log` (default) | allows it, counts it, and logs one line per action and position |
| `enforce` | answers `403` |
| `off` | does not look at direction at all; a WARNING on every start |

Any other value stops the node at start. `security_profile=production` does not force `enforce` yet.

`enforce` needs to know the speaker, so a node refuses to start with `authority_direction=enforce` when
`msg_signing_required=false` (no verified signatures: no speaker), `peer_identity_mode=off` (the global cannot place a
region's controller) or `broker_security_enabled=false` (no caller is checked at all). In `log` mode the first two start
with a WARNING that the counts on that node are not meaningful. None of the three is a default.

## What you see

In `log` mode, once per action and position (every occurrence is counted):

```
AUTHORITY: WOULD REFUSE CONFIG killjvm from controller:t1/rgA/a1[infra], the word of t1/rgA/a1 (ELSEWHERE): it is
honoured only from the node itself or a controller above it; allowed (authority_direction=log); logged once per action
and position, every one counted in authz.direction.would_refuse
```

In `enforce` mode the caller gets:

```
status_code=403
status_desc=unauthorized: killjvm is honoured only from the node itself or a controller above it
            (caller controller:t1/rgA/a1[infra] speaks from a node elsewhere)
```

(`or a node below it` for an action that goes up.)

The counters are in the [metric inventory](../architecture/metrics.md), group `authz`:

| Metric | Counts |
|---|---|
| `authz.direction.would_refuse` | calls `enforce` would have refused (`log` mode) |
| `authz.direction.would_refuse.<action>` | the same, per action (`.watchdog` and `.kpi` for those message types; `.other` for an action a plugin gave a direction of its own) |
| `authz.direction.refused`, `authz.direction.refused.<action>` | calls refused for their direction (`enforce` mode), named the same way |

## Before you turn on `enforce`

1. Run in `log` mode through everything the fabric normally does: a restart of a controller, a deployment, your
   administration scripts, your dashboards.
2. Read `authz.direction.would_refuse` on every node (`getmetricinventory`, or the log lines). It must be zero for
   everything you mean to keep doing.
3. Move what is counted to the right gateway. Usual cases: an administration script or dashboard connected to a leaf
   or to a region controller that manages nodes outside its reach; setting a log level or reading a log through a
   gateway that is neither the target nor above it.
4. Set `authority_direction=enforce` on the leaves first, then the region controllers, then the global.

Known to be counted on a healthy fabric today, and being changed before `enforce` becomes the default: a file pulled
directly between two agents (`getfileinfo` / `getfiledata` sent to the node that holds the file) gets a transfer ticket
issued from above.

## For plugin authors

A plugin's privileged actions (the ones it restricts to `infra` or `admin`) are `down` without any change to the
plugin. An action can state its own direction:

```java
@CrescoAction(name = "report_usage", direction = "up", ...)
```

`down`, `up` or `any`; an unknown value fails the plugin's start. A privileged action stays `down`: a plugin that
declares one `any` or `up` fails its start with a message that names the action (like roles, directions can only be
narrowed). A plugin's call to its own node is always the node itself.

A plugin built against this library on a controller of an earlier release behaves as before (the older controller is
never asked). Nothing changes on the wire.

## Limits

- An agent knows the global above it from the trust data its present region controller signed. When a region moves to
  another global, its agents stop honouring the old one as soon as they next hear from their region controller (within
  seconds).
- At the global, a region that left stays "below" until its certificate stops verifying. That only affects what a child
  may report upward (`up` actions), not management.
- A speaker the node cannot place is refused in `enforce` (and counted in `log`).
