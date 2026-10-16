# Security & Identity

Cresco's security model has three layers, each independently switchable and default-off so a fabric can be
hardened incrementally: **identity**, **trust**, and **authorization**. Multi-tenant isolation builds on
top of these and has its own page: [Multi-Tenancy](tenancy.md).

## Identity — the certificate *is* the identity

Every node generates a three-tier X.509 chain (root → intermediate → leaf). The **leaf carries the node's
identity in its subject DN**:

```
CN=<agent>, OU=<region>, O=<tenant>, UID=<stable-node-uid>
```

`CrescoIdentity` parses this both ways (identity ⇄ DN). Because identity is bound *into* the certificate,
any holder of the issuing CA can cryptographically verify a node's claimed tenant/region/agent — there is
no separate identity database to trust.

## Transport — mutual TLS

Broker connections use `nio+ssl`. With `broker_require_client_auth=true` the broker demands a client
certificate, validates its chain, and derives the **authenticated principal from the verified DN** —
overriding any self-asserted username. A client thus cannot spoof its tenant/region/agent; it would need
the private key of a certificate the broker trusts.

## Trust — regional CAs, distributed

Two trust models:

| Model | Config | Trust material |
|-------|--------|----------------|
| **Direct** (default) | — | each node imports every peer's leaf cert — O(nodes²) |
| **Regional-CA** | `security_regional_ca=true` | each node trusts a handful of **region CA** certs — O(regions) |

Under the regional-CA model, issuance is **regional and distributed**: a region/global controller holds a
persistent region CA; a joining node enrolls over the [discovery](discovery.md) handshake (gated by the
join secret), and the region CA **signs it a leaf with a stamped identity** and returns the chain. Globals
distribute (do not sign) trust — they aggregate region-CA bundles and gossip them across the mesh, so an
agent under one global can verify an agent under another. This scales to a large mesh with no central CA
and no single point of compromise.

## Authorization — role-based tenant ACLs

With `broker_security_enabled=true` the broker installs `CrescoAuthorizationBroker`, which checks every
consumer/producer/send against `TenantPolicy` using the connection's verified identity. Authorization is
**role-based**:

| Role | Access |
|------|--------|
| `SUPERUSER` | every destination in every tenant (god view). Granted to the local controller (`vm://`), broker bridges, and network clients whose tenant is in `broker_superuser_tenants` (default `cresco-system`). |
| `TENANT` | confined to its own tenant's destinations. |

This is the enforcement point for [multi-tenancy](tenancy.md).

## Message signing & payload encryption

Beyond transport TLS, the library provides `MessageSigner` for **selective, end-to-end message signing** —
a sender signs the payload digest with its leaf key; any receiver verifies with the sender's leaf
(chain-validated), proving origin and tamper-evidence independent of how many brokers relayed the message.
Optional payload encryption to a recipient's public key is available for confidentiality beyond the first
hop.

## Crypto policy — one set of allowed algorithms per fabric

Which algorithm types the fabric may choose for anything new (certificate signature suites, a storage domain's
content-address hash, key wraps, KDFs, ciphers) is one policy (`io.cresco.library.crypto.CryptoPolicy`), set on the
global controller (`crypto_policy` = `cnsa1` by default or `fips140-3`, plus per-family `crypto_allow_*` lists of FIPS
140-3 approved names) and enforced by Cresco and every plugin. The global stamps it on the `region_enable` /
`agent_enable` registration replies; each ping carries the id the child holds and the pong carries the policy when the
ids differ, so a joining node gets it at once and a change reaches every node within a ping interval per tier. Regions
and agents ignore local crypto keys. A node without the policy refuses every algorithm choice for a new object
(fail closed). Existing data, fixed wire formats and verification are never refused. See the
[configuration reference](../reference/configuration.md#fabric-crypto-policy-global-controller-only).

## Enable it, in order

1. `security_regional_ca=true` — every node gets a CA-signed, identity-bearing leaf.
2. `broker_require_client_auth=true` — mutual TLS binds identity non-spoofably.
3. `broker_security_enabled=true` — the ACL enforces tenant isolation.
4. `tenant_namespacing=true` — end-to-end [tenant](tenancy.md) destination isolation.

The full rationale is in the distributed-identity-trust design doc — see [Design Docs](../reference/design-docs.md).
