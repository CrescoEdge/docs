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
- **Leaf agents** (network clients) consume only their own `T.<tenant>.` inbox.
- **Infra controllers** (on `vm://`, multi-tenant relays) consume the `T.*.` **wildcard** inbox, so they
  relay every tenant.

A cross-tenant send therefore lands in a queue nobody consumes *and* is denied by the ACL — isolation is
enforced twice over.

## Role-based authorization

Enforced at every broker by `TenantPolicy` via `CrescoAuthorizationBroker`:

| Role | Can touch |
|------|-----------|
| `TENANT` | only its own `T.<tenant>.` subtree — cross-tenant subscribe/publish is denied |
| `SUPERUSER` | every tenant (god view) — for the infrastructure and for admin/dashboard clients whose tenant is in `broker_superuser_tenants` |

This closes a real hole in the flat-name scheme: previously a tenant could **write** into a same-region
peer's `region_agent` inbox; with tenant-scoped inboxes that is denied.

## What is and isn't guaranteed

**Guaranteed** (with mutual TLS + `broker_security_enabled` + `tenant_namespacing`): a tenant client cannot
subscribe to or publish into another tenant's subtree at **any** broker in the mesh; identity is
non-spoofable.

**Not guaranteed by this layer** (documented, with tiered mitigations): a *compromised broker* is trusted
infrastructure and can see all tenants (mitigation: dedicated per-tenant bridges + payload encryption);
payloads at rest inside a transit broker are readable by that broker (mitigation: payload encryption);
advisory topics can leak destination *names* (not contents). See the design doc's threat-model section.

## Enabling it

Turn on the [security](security.md) stack first, then namespacing, staged region-by-region:

```
-Dsecurity_regional_ca=true
-Dbroker_require_client_auth=true
-Dbroker_security_enabled=true
-Dtenant_namespacing=true
-Dtenant_id=<tenant>
-Dbroker_superuser_tenants=cresco-system   # for admin/god-view clients
```

Proven by `tenant_namespacing_test` (federation forms + zero self-denial under namespacing; TENANT
confined; SUPERUSER god view) with no regression across the security/isolation suites.
