# Security in Cresco 1.3

Cresco 1.3 is **secure by default**: a node started with no security setting at all runs with broker authorization,
mutual TLS between every node, tenant-namespaced destinations, a verified caller on every message, per-action roles,
end-to-end message signatures and signed plugin installs. Insecure operation exists only as an explicit debug mode
(`broker_security_enabled=false`, a WARNING at every boot) and is never for a network you do not control.

This section is the operator's view of that model. The normative contracts live in the repositories:
`controller/docs/caller-authorization.md` (who may do what), `controller/docs/production-deployment.md` (the
production profile), `controller/docs/pre-provisioned-identity.md` (operator PKI), `controller/docs/upgrade-1.3-secure-defaults.md`
(the upgrade), the wsapi README (API authentication) and the agent README (crypto modules and FIPS).

| Page | What it covers |
|---|---|
| [Secure defaults](secure-defaults.md) | what a node does with no security setting, what it refuses, how to check it |
| [Security profiles](profiles.md) | `standard` (the default) and `production` (operator PKI, FIPS, no passwords) |
| [Operator PKI](pki.md) | pre-provisioned identities and `tools/cresco-pki.sh` |
| [Upgrading to 1.3](upgrade.md) | the order, the time-boxed upgrade window, checks and rollback |
| [API authentication](api-auth.md) | wsapi users, API tokens, the service key, client certificates (mTLS) |
| [Signed plugins](signed-plugins.md) | plugin jar signing and what is verified where |
| [FIPS 140-3](fips.md) | the validated crypto modules, approved-only mode, keystores, throughput |

## The layers

- **Identity.** Every node's identity is its X.509 certificate's subject (`CN=<agent>, OU=<region>, O=<tenant>`),
  verified at every TLS handshake. In the standard profile nodes make or enroll their certificates; in production the
  operator's PKI issues them ahead of time.
- **Transport.** Broker connections (`nio+ssl`) and the wsapi API are TLS; the broker requires client certificates
  (`broker_require_client_auth`) and binds each connection to the identity its certificate names.
- **Authorization.** The broker stamps the verified caller of every message (`cresco_caller`, kind `controller`,
  `agent`, `plugin` or `api`), receivers check it against each action's roles (`@CrescoAction(roles=...)`), and
  every message a node originates is signed with its identity key (RSA-3072/SHA-384 by default) and checked before it
  is dispatched (`msg_signing_required`).
- **Tenancy.** Destinations are namespaced per tenant (`T.<tenant>.*`); a tenant's nodes and users reach only their
  tenant, except admins of a superuser tenant (`broker_superuser_tenants`, default `cresco-system`).
- **Code.** Every plugin jar that does not come embedded in the agent jar is installed only if it verifies against
  `plugin_signing_truststore`.
- **Crypto.** Algorithms meet FIPS-approved and CNSA 1.0 strength (SHA-256/384, AES-256, RSA-3072 or EC P-384,
  PBKDF2-HMAC-SHA-384, a DRBG). The production profile runs the JVM in FIPS 140-3 approved-only mode.

## What reaches an API client

A reply reaches an API client without the fabric's own stamps: wsapi removes the origin's end-to-end signature
(`cresco_sig*`, including a node's certificate in `cresco_sig_cred`), the broker's caller stamp (`cresco_caller*`), the
namespace tenant, the routers' annotations (`routepath-*`, `linkcost-*`, `srcroute*`) and the RPC plumbing
(`callId-*`, `is_rpc`), and drops the same names from what a client sends. A dataplane envelope frame names its verified
writer in its own fields and keeps only the stream's sequence numbers (`cresco_seq`, `cresco_gap`) among the
`cresco_*` properties.
