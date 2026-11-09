# Security profiles

`security_profile` selects how a fabric gets and trusts its identities. Sources: `controller/docs/production-deployment.md`,
`controller/docs/caller-authorization.md` section 11.

| | `standard` (default) | `production` |
|---|---|---|
| Node identities | made by the node, or issued by its region CA at a discovery join (`security_regional_ca`) | issued ahead of time by the operator's PKI (`identity_keystore`); nothing is generated or enrolled |
| Trust | pins on first join; regional CAs | `trust_anchors` only; no trust on first use |
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
that weakens security, the upgrade window, `security_regional_ca=true`, `peer_identity_rebind`, inline passwords and
secrets on the command line. Any secret a production node is given comes only from an owner-only `<name>_file` file or
a `CRESCO_<NAME>` environment variable; whether your build still admits nodes to discovery with the discovery secrets
beside the PKI is in its `controller/docs/production-deployment.md`.

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
