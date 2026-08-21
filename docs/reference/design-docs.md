# Design Documents

The architecture pages in this site are the *narrative* reference. Behind them are the original
**engineering design documents** — deeper, decision-level records of why each subsystem is built the way it
is, with measurements, trade-offs, and threat models. They are included here in full so that all project
knowledge lives in one place.

!!! note
    These are engineering documents. Some contain source-path references (`code/...`) that are links within
    the repository rather than this site.

## Security, identity & tenancy

| Document | Covers |
|----------|--------|
| [Distributed Identity, Trust & Data-Rights](../design/distributed-identity-trust-design.md) | identity-bearing certs, regional-CA issuance, federated trust bundles, mutual TLS binding |
| [Tenant Isolation (Glocal)](../design/tenant-isolation-design.md) | destination namespacing, role-based authorization, message-flow walkthrough, threat model, config/ops reference |

## Routing, messaging & performance

| Document | Covers |
|----------|--------|
| [Dynamic Cost-Aware Routing](../architecture/dynamic-routing.md) | **shipped** — link-state push, Dijkstra path selection, source-route enforcement, self-organizing link inference |
| [Optimal Global Routing Plan](../design/optimal-global-routing-plan.md) | the design/roadmap; phases A/C/D are now implemented (see the architecture page above), B/E/F remain |
| [Coordinator Decentralization Plan](../design/coordinator-decentralization-plan.md) | region-first autonomy + elected/sharded coordinators; decomposes the single static global by duty, confines consensus to liveness verdict + global optimization |
| [Broker & Data-Plane Performance](../design/broker-performance.md) | every broker/data-path performance change with measured effect and the config lever |
| [Region Federation](../design/region-federation-design.md) | same-host region federation, identity-vs-IP self-detection |
| [Network Link Metrics + Auto-Tuning](../design/link-metrics-design.md) | per-edge RTT/jitter/throughput/backlog, the auto-tuner control loop, the link cost model |
| [Control / Data Plane Isolation](../design/control-dataplane-isolation.md) | why priority alone is not enough, dedicated control sockets, the queue/topic bridge split, prefetch bounds, measured effect |
| [stunnel Data Path](../design/stunnel-dataplane-design.md) | tunnel vs session lifetimes, the per-tunnel demux, ordering guarantees, bounded waits and failure modes |

## Health, metrics & capability

| Document | Covers |
|----------|--------|
| [Health, State & Inventory](../design/health-check-design.md) | the Felix-health-check migration, grace/sticky windows, mesh rollup |
| [Metrics & Measurements Unification](../design/metrics-unification-design.md) | one Micrometer model across controller + all plugins + clients, mesh aggregation |
| [Capability Inventory](../design/capability-inventory-design.md) | `@CrescoAction` self-description → fabric-wide capability catalog |

## Exhaustive action reference

| Document | Covers |
|----------|--------|
| [Cresco Tools Reference](../design/cresco-tools-reference.md) | the full auto-generated per-tool detail for all actions (the [Plugin Actions](../api/plugin-actions.md) page is the distilled version) |
