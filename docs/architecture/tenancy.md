# Multi-Tenancy

Cresco supports **multiple mutually-distrusting tenants sharing one physical fabric** — the same brokers,
the same inter-broker bridges — with end-to-end isolation. It builds directly on the
[security model](security.md) (the tenant is cryptographically bound into each node's certificate DN as
`O=<tenant>`).

Everything here is gated by `tenant_namespacing`, which follows `broker_security_enabled` and so is **on by default**
since the 1.3 hardening ([Secure defaults](../security/secure-defaults.md)). The full design
and threat model live in the tenant-isolation design doc; see [Design Docs](../reference/design-docs.md).

## The principle: one global invariant, enforced locally

> A message stamped tenant *T* only ever lives on destinations prefixed `T.<tenant>.`, and is reachable
> only by a connection authenticated as *T*.

Because the tenant is both **cert-bound** (identity) and **name-carried** (a destination prefix), every
broker enforces the invariant with a purely local string check — no coordination — so it holds across
bridges and across globals.

## Destination namespacing

Both controlled channels are qualified with the owning tenant:

| Channel | Raw | Namespaced |
|---------|-----|-----------|
| [MsgEvent](../api/msgevent.md) inbox queue | `region_agent` | `T.<tenant>.region_agent` |
| [Data plane](dataplane.md) topics | `agent.event` … | `T.<tenant>.agent.event` … |

- **Producers** stamp the origin tenant onto the message (`cresco_tenant`, preserved across relays) and
  qualify the forward destination with the *message's* tenant, so every relay produces the identical queue
  name.
- **Leaf agents** (network clients) consume only their own `T.<tenant>.` inbox (a leaf of a superuser tenant: its
  inbox in every tenant's namespace).
- **Infra controllers** (on `vm://`, multi-tenant relays) consume the `T.*.` **wildcard** inbox, so they
  relay every tenant.

A cross-tenant send therefore lands in a queue nobody consumes *and* is denied by the ACL — isolation is
enforced twice over.

## Several tenants under one global

The shape: a **global in a superuser tenant** (`broker_superuser_tenants`, default `cresco-system`), and under it each
tenant's own region controllers and agents. A region joins the global over its bridge exactly as in a single-tenant
fabric. What makes it work, and what holds it together:

| What | How |
|---|---|
| a tenant's region registers with the global | a region publishes its state in its own namespace (`T.<tenant>.agent.event`); a global of a superuser tenant listens for every tenant's (`T.*.agent.event`). A state export is taken only from the **controller that joined**, reporting **its own region**, and its rows carry the tenant that controller was verified in, never one it claims |
| the global reaches a tenant's node | the global's own message travels in the namespace of the **destination region's tenant**, which it knows from the controllers that completed discovery with it |
| a tenant's node answers the global, or asks it | a tenant's node always sends in **its own** namespace; the global's answer travels in the **verified tenant of the caller it answers**, whatever the request named |
| a superuser tenant's own nodes (an operator's agent) | consume their inbox in every tenant's namespace (`T.*.<their inbox>`), and judge each message by its verified caller like any other |
| a request that arrives in another tenant's namespace than its caller's | refused where it arrives, counted (`msg.dropped.cross_tenant`); a request from another tenant that names a sender other than the node that signed it: `msg.dropped.sender_unbound` |

Rules an operator must know:

- **A region name belongs to one tenant.** If controllers of two tenants are recorded for one region name, the global
  sends its own messages for that region into neither namespace: `TENANT UNRESOLVED` in its log, counted in
  `msg.tenant.unresolved`. Under an operator PKI the second claimant is refused when it joins.
- **Node names are unique across the fabric.** The registry lists an agent (and a plugin) by its name, so a second node
  of the same name in another tenant is not listed (`REGISTRY NAME COLLISION` at the global). It is still reachable by
  its address; give nodes of different tenants different names.
- **A superuser node that did not see a region's controller join** (a second global, an operator's agent) does not know
  that region's tenant: its client names it (`cresco_tenant`), which the gateway vets.
- A region that just joined is addressed correctly within a second.

Before agent release 1.3.378 this shape did not form: a tenant's region never registered with a global of another
tenant, and answers between them were dropped. (The test of the time checked that the bridges came up, not that the
regions were listed; the fabric matrix's `tenants` cell checks it now, see [Testing](../operations/testing.md).)

## Role-based authorization

Enforced at every broker by `TenantPolicy` via `CrescoAuthorizationBroker`:

| Role | Can touch |
|------|-----------|
| `TENANT` | only its own `T.<tenant>.` subtree — cross-tenant subscribe/publish is denied |
| `SUPERUSER` | every tenant (god view) — for the infrastructure and for admin/dashboard clients whose tenant is in `broker_superuser_tenants` |

This closes a real hole in the flat-name scheme: previously a tenant could **write** into a same-region
peer's `region_agent` inbox; with tenant-scoped inboxes that is denied.

### Named cross-tenant flows

Short of `SUPERUSER`, the only way across the tenant boundary is an explicit allow-list,
`broker_cross_tenant_sinks` (empty by default: the strict own-subtree rule). Each entry is
`<source tenant>-><destination>[:write|read|any]` (globs: `*` any run of characters, `?` one character; `write` when
no access is named), separated by `;` or `,`:

```
-Dbroker_cross_tenant_sinks='*->T.gfs-federation.*:write ; gfs-federation->T.*:write'
```

A principal whose tenant matches the source may reach the matching destinations with that access. A rule never grants
READ unless it says so, so a tenant still cannot consume another tenant's inbox.

## What is and isn't guaranteed

**Guaranteed** (with mutual TLS + `broker_security_enabled` + `tenant_namespacing`): a tenant client cannot
subscribe to or publish into another tenant's subtree at **any** broker in the mesh; identity is
non-spoofable. The same holds for a tenant's region controller and its bridges: a bridge carries its peer's own
tenant's subtree, what is addressed to an inbox of the peer's own region, and answers to a superuser tenant's nodes --
nothing else leaves over it and nothing else is taken from it (counted and logged as `WITHHELD from a bridge` /
`DISCARDED a message a bridge brought in`). Only the bridge to a region's own global, and a controller of a superuser
tenant, carry every tenant's traffic. (Before 2026-11-11 every verified bridge was exempt from tenant policy.) Under
`security_profile=production` a bridge peer is a controller of the PKI that completed discovery with this node, and a
region's names belong to one tenant: a certificate of another tenant that names that region is not that region's node.

**Within a tenant, a node reads its own inbox only** (from agent release 1.3.379). A node's broker gives it no other
queue of the tenant's namespace: not its region controller's inbox, not a sibling's, no pattern over them, no queue
browser, and no broker notice that carries a copy of another node's message; it cannot remove another node's queue.
Before, any node of a tenant could read, and take away, the control messages of every other node of its tenant at its
broker. A node moved to another region (`region_alias`) also reads the inbox of its former address, which is bound to
it by the agent name in its certificate: **agent names must be unique within a tenant**, because a second node given
the agent name of another, in a different region, can read that node's inbox through this form.

**Not guaranteed by this layer** (documented, with tiered mitigations): within a tenant every node can still read the
tenant's dataplane topics (streams bound to a recipient excepted) and can write to any inbox of its tenant (what it
writes is judged by its signature and by [where it stands](../security/authority.md)); a *compromised broker* is trusted
infrastructure and can see all tenants (mitigation: dedicated per-tenant bridges + payload encryption);
payloads at rest inside a transit broker are readable by that broker (mitigation: payload encryption);
advisory topics can leak destination *names* (not contents). See the design doc's threat-model section.

## Enabling it

Namespacing is on by default with the rest of the [security](security.md) stack; a node needs only its tenant (and,
for admin/god-view clients, the superuser tenants):

```
-Dtenant_id=<tenant>
-Dbroker_superuser_tenants=cresco-system   # the default
```

With security on, `tenant_namespacing=false` refuses the start. A fabric of an earlier build is moved over with the
[upgrade guide](../security/upgrade.md).

Proven by `tenant_namespacing_test` (federation forms + zero self-denial under namespacing; TENANT
confined; SUPERUSER god view) with no regression across the security/isolation suites.
