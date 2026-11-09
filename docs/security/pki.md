# Operator PKI: pre-provisioned identities

In the production profile every certificate is computed ahead of time by the operator's PKI; a node loads the identity
it was given, trusts the fabric's anchors and judges every peer by that PKI. Source: `controller/docs/pre-provisioned-identity.md`.

## Identity names

- **node**: subject `CN=<agent>, OU=<region>, O=<tenant>` (names of 1-64 `A-Z a-z 0-9 . -`; a tenant without `.`), with
  the SAN URI `cresco:node:<tenant>/<region>/<agent>`;
- **controller** (region or global): also `cresco:role:controller`; **the global**: also `cresco:role:global` (issued by
  the root);
- **API user**: SAN URIs `cresco:api:<tenant>/<user>` and `cresco:role:<admin|user>`, subject `CN=<user>, O=<tenant>`;
- **CA scope (fail closed)**: every CA on a path must allow the leaf's tenant, region and roles, with exactly one
  `cresco:ca:<tenant>/<region>` and one `cresco:ca-roles:<roles>` SAN; only the self-signed root may be unscoped.

## The offline tool: `tools/cresco-pki.sh`

Requires OpenSSL 3; run it on an offline, owner-only machine (the PKI directory holds the CA keys).

```
tools/cresco-pki.sh init   <pki> [--name "Cresco Fabric Root"] [--days 3650] [--key rsa|ec]
tools/cresco-pki.sh region <pki> <tenant> <region> [--days 1825]
tools/cresco-pki.sh node   <pki> <tenant> <region> <agent> [--controller|--global] [--issuer root|region] [--days 365]
tools/cresco-pki.sh api    <pki> <tenant> <user> <admin|user> [--days 365]
tools/cresco-pki.sh revoke <pki> <certificate.pem>
tools/cresco-pki.sh crl    <pki> [--days 30]
```

`node` writes `<pki>/nodes/<region>_<agent>/{node.p12, node.p12.pass, trust-anchors.pem, node.pem}` (the key exists only
in the keystore); issuing the same node again is a renewal (a new key, the same password file). `crl` writes
`<pki>/revocations.pem`, the `revocation_file` for every node. Production runs FIPS approved-only, whose keystores are
BCFKS: convert each node's keystore once, with the same password:

```sh
java -jar agent.jar --crypto convert-keystore node.p12 node.p12.pass node.bcfks
```

Start a node with

```
-Dsecurity_profile=production -Didentity_keystore=<dir>/node.bcfks -Didentity_keystore_password_file=<dir>/node.p12.pass \
-Dtrust_anchors=<dir>/trust-anchors.pem -Drevocation_file=<shared>/revocations.pem -Dregionname=<region> -Dagentname=<agent>
```

## Files, reload, revocation, expiry

- Every identity file must be a regular file owned by the node's user (or root), not writable by others (the keystore
  and its password file not readable by them either), in directories only that user or root can write; otherwise the node
  refuses to start, and a later change raises a CRITICAL `identity` alarm.
- A replaced keystore is reloaded within `identity_keystore_reload_seconds` (30) without dropping a connection; one that
  does not validate is refused and the node keeps its identity.
- Revocation: the operator's `revocation_file` (CRLs and a deny list of `key:`, `cert:` or `name:` entries) and the signed
  revocation lists controllers exchange; a revoked key is refused at TLS, at the broker and in discovery, and live
  connections holding it are closed.
- Expiry is alarmed by the `identity` health check (WARN in the renewal window, CRITICAL near and at expiry).
