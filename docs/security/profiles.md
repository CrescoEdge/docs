# Security profiles

`security_profile` selects how a fabric gets and trusts its identities. Sources: `controller/docs/production-deployment.md`,
`controller/docs/caller-authorization.md` section 11.

| | `standard` (default) | `production` |
|---|---|---|
| Node identities | made by the node, or issued by its region CA at a discovery join (`security_regional_ca`) | issued ahead of time by the operator's PKI (`identity_keystore`); nothing is generated or enrolled |
| Trust | pins on first join; regional CAs | `trust_anchors` only; no trust on first use |
| Discovery | discovery v2, keyed by the shared discovery secrets | the same exchange signed with each side's provisioned key, chains verified to `trust_anchors`; no discovery secret is used or accepted |
| Upgrade window (`discovery_auth=compat`) | available, time-boxed | refused |
| Passwords and shared secrets (`cresco_service_key`, `keystorepwd`, ...) | allowed | refused (secrets only as owner-only `<name>_file` or `CRESCO_<NAME>`) |
| API clients (wsapi) | user passwords, API tokens, the service key | client certificates only (tokens only with `wsapi_allow_tokens=true`) |
| Security settings | secure by default, adjustable | all mandatory: broker security, client auth, namespacing, plugin and message signing, `peer_identity_mode=enforce` |
| Crypto | the JDK's providers | FIPS 140-3 approved-only mode ([FIPS](fips.md)) |

Insecure debug mode (`broker_security_enabled=false`) is not a profile: it is never production.

## Production: what is required

A controller with `security_profile=production` **refuses to start** (exit 78) and names every problem when anything
below is missing or anything disallowed is set:

| Required | |
|---|---|
| `identity_keystore` | this node's key and certificate chain, a **BCFKS** keystore (convert the PKI's PKCS#12, see [FIPS](fips.md)) |
| `identity_keystore_password_file` | owner-only file holding that keystore's password |
| `trust_anchors` | PEM bundle of the fabric's root CA(s) |
| `plugin_signing_truststore` | the anchors plugin jars are signed to (a PEM bundle needs no password) |

Optional: `revocation_file` (the operator's CRLs and deny list), `controller_trust_anchors`. Refused: every setting
that weakens security (`broker_security_enabled=false`, `broker_require_client_auth=false`, `tenant_namespacing=false`,
`plugin_signing_required=false`, `msg_signing_required=false`, a `peer_identity_mode` other than `enforce`,
`broker_allow_unverified_bridges=true`), the upgrade window, `security_regional_ca=true`, `peer_identity_rebind`,
`public_key_trusted_fingerprints`, the passwords (`cresco_service_key`, `broker_security_secret`, `keystorepwd`,
`truststorepwd`, `plugin_signing_truststore_password`), the discovery secrets (`discovery_secret_agent`, `_region`,
`_global`, in any form: a leftover is named, not ignored) and `db_password` on the command line. Any secret a production
node is given comes only from an owner-only `<name>_file` file or a `CRESCO_<NAME>` environment variable.

The controller also checks that the JVM really runs in FIPS 140-3 approved-only mode before it starts: every JCA provider
is one FIPS mode leaves (`BCFIPS`, `BCJSSE`, `CrescoACCP-FIPS`, `CrescoFIPS`, `SunJSSE`), Bouncy Castle FIPS is in
approved-only mode and passed its self-tests, ACCP-FIPS (where installed) is the FIPS build and healthy, MD5 is refused and
SHA-384 gives its known answer. Otherwise: `REFUSING TO START: security_profile=production runs in FIPS 140-3
approved-only mode, and this JVM is not in it: ...`. Start production nodes with the agent jar, whose crypto boot sets the
mode up ([FIPS](fips.md)).

## Discovery in production

Discovery is authenticated by the operator's PKI alone. The exchange has the same four messages as the standard
profile's discovery v2 (HELLO, CHALLENGE, PROOF, RESULT, over fresh nonces from both sides), but each side signs the
transcript with its provisioned identity key instead of keying a MAC with a shared secret:

- the joiner's whole chain must verify to `trust_anchors` as what it claims (name, role, validity, revocation, the
  region's CA scope) before the responder signs anything for it;
- the responder's chain must verify the same way as the controller it answered as (a GLOBAL answer only from a
  certificate carrying `cresco:role:global`), and its signature (over both nonces, both identities, both chains and its
  broker port) must verify under its leaf before the joiner answers;
- the joiner's signature must verify under the leaf it presented, so a copied (public) chain without its key gets
  nothing, and a replayed or altered exchange fails;
- nothing is issued, enrolled or pinned on first use.

Locators (the UDP broadcast and the first TCP probe) carry no secret either: a production controller answers a PKI
locator with its region, name and broker port and nothing else, and does not answer a secret-keyed locator or a legacy
exchange (`Discovery: REFUSED ... authenticated by the operator's PKI only ...`, once per address). A production node and
a standard-profile node never complete a discovery exchange with each other.

## Plugins follow the node's profile

Plugins read the profile from their controller, `PluginBuilder.getSecurityProfile()` (`PRODUCTION` when the
controller cannot be asked), never from their own configuration. wsapi refuses to start when its own `security_profile`
names another profile than the node's, and refuses every login while its node answers production and it did not start
in production.

## A production node, configured

```
security_profile=production
tenant_id=t1
identity_keystore=/etc/cresco/node.bcfks
identity_keystore_password_file=/etc/cresco/node.p12.pass
trust_anchors=/etc/cresco/trust-anchors.pem
plugin_signing_truststore=/etc/cresco/codesign-anchors.pem
revocation_file=/etc/cresco/revocations.pem               # optional
wsapi_client_ca=/etc/cresco/api-client-ca.pem             # wsapi: the CA of API client certificates
```
