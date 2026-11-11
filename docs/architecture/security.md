# Security & Identity

Cresco's security model has three layers: **identity**, **trust**, and **authorization**. Multi-tenant isolation
builds on top of these and has its own page: [Multi-Tenancy](tenancy.md).

!!! note "1.3: secure by default"
    Since the 1.3 hardening every layer is **on by default** (insecure operation is only an explicit
    `broker_security_enabled=false` debug mode), and the production profile adds operator-issued identities and FIPS
    140-3 approved-only crypto. The operator's guide is the [Security](../security/index.md) section:
    [secure defaults](../security/secure-defaults.md), [profiles](../security/profiles.md),
    [operator PKI](../security/pki.md), [upgrade](../security/upgrade.md), [API authentication](../security/api-auth.md),
    [signed plugins](../security/signed-plugins.md) and [FIPS](../security/fips.md). This page describes the mechanisms.

## Identity — the certificate *is* the identity

Every node generates a three-tier X.509 chain (root → intermediate → leaf). The **leaf carries the node's
identity in its subject DN**:

```
CN=<agent>, OU=<region>, O=<tenant>, UID=<stable-node-uid>
```

`CrescoIdentity` parses this both ways (identity ⇄ DN). Because identity is bound *into* the certificate,
any holder of the issuing CA can cryptographically verify a node's claimed tenant/region/agent — there is
no separate identity database to trust. In the `production` [security profile](../security/profiles.md) nothing is
generated or enrolled: every identity is issued ahead of time by the operator's PKI ([Operator PKI](../security/pki.md)).

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
and no single point of compromise. The region CA's private key (`region-ca.p12` in the node's identity directory) is
encrypted at rest under `region_ca_password_file`, an owner-only file of at least 16 characters the operator keeps
elsewhere (a mounted secret, another volume); without it the key is protected only by the random password beside it and
that directory's permissions, and a WARNING says so. The regional CA is a `standard`-profile convenience: production
refuses `security_regional_ca=true`.

## Authorization — role-based tenant ACLs

With `broker_security_enabled=true` the broker installs `CrescoAuthorizationBroker`, which checks every
consumer/producer/send against `TenantPolicy` using the connection's verified identity. Authorization is
**role-based**:

| Role | Access |
|------|--------|
| `SUPERUSER` | every destination in every tenant (god view). Granted to the local controller (`vm://`), the bridge to a region's own global, and network clients and controllers whose tenant is in `broker_superuser_tenants` (default `cresco-system`). Any other bridge (a region below a global, a peer region) carries its peer's own tenant's and region's traffic only. |
| `TENANT` | confined to its own tenant's destinations. |

This is the enforcement point for [multi-tenancy](tenancy.md).

## End-to-end message authentication

Transport TLS and the broker's caller stamp vouch for one hop. Every MsgEvent a node originates is also authenticated by
its origin and checked by its recipient before dispatch, so the origin's word survives relays, source routes and the
bridges of other regions: unsigned between a node and its own parent or child (the receiving broker attests the sender
first hand), otherwise an RSA-3072/SHA-384 signature (batched under load) or an HMAC-SHA-384 under an end-to-end session
key agreed with ECDH P-384. The signature or MAC covers the recipient (`cresco_sig_to`), so a message signed for one node
is refused by any other, and the signer's revocations are checked on every message. See
[Message authentication](../security/message-auth.md).

The library also offers plugins `io.cresco.library.security.MessageSigner`, a plain sign/verify of a byte payload with a
private key or a certificate's public key.

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

## Defaults

Since the 1.3 hardening mutual TLS (`broker_require_client_auth`), the authorization broker (`broker_security_enabled`),
tenant namespacing (`tenant_namespacing`), message authentication (`msg_signing_required`) and plugin signing
(`plugin_signing_required`) are **on by default**. With security on, `broker_require_client_auth=false` or
`tenant_namespacing=false` refuses the start; `msg_signing_required=false` and `plugin_signing_required=false` are
development settings, logged as a WARNING (and refused in production). `security_regional_ca` stays opt-in (standard
profile only). See [Secure defaults](../security/secure-defaults.md).

The full rationale is in the distributed-identity-trust design doc — see [Design Docs](../reference/design-docs.md).
