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
  does not validate is refused and the node keeps its identity. Trust anchors and the revocation file are reloaded the
  same way. A reload is atomic: the files are read as one set (one that changes while it is read is taken at the next
  check), everything is validated and built aside, then put in force at once; a handshake always gets one chain and its
  key. The role a peer has is the one its presented certificate carries, never one remembered for its name.
- Revocation: the operator's `revocation_file` (CRLs and a deny list of `key:`, `cert:` or `name:` entries) and the signed
  revocation lists controllers exchange; a revoked key is refused at TLS, at the broker and in discovery, and live
  connections holding it are closed. Message authentication applies `revocation_file` itself, **on every message**: the
  signer's certificate and the certificates above it are checked against the operator's keys, certificates and CRL
  serials ([Message authentication](message-auth.md)). The lists controllers exchange run both ways with a different
  reach: a parent's list (a global's, to its regions; a region's, to its agents) is taken whole, while a region's list
  reaches its parent for that region's own names only -- an entry is accepted for a key the parent verified under that
  region (or, where the parent verifies the region's names by certificate, for an identity of that region the entry
  names), applies to that region's names alone, and a region holds at most 10,000 entries there. A region cannot have
  another region's nodes refused. (Before 2026-11-11 an entry of a region's list was applied to every name.) Under
  per-node trust a region's revocation of one of its own agents therefore stays in that region unless the global had
  bound that agent's key; revoke at the global what the whole fabric must refuse. Revoking a name reserves it; a rebind token
  (`<region>/<agent>=<new key fingerprint>`) lifts that once the PKI issued a new certificate, and only for a chain the
  PKI accepts in full (a refused chain never spends the token).
- **CRL freshness.** A CRL must carry a `nextUpdate` and a CRL number (the tool's do). Past its `nextUpdate` (plus
  `revocation_crl_grace_seconds`, default 0) the revocation status of its CA's certificates is unknown and they are
  refused until `revocation_file` is refreshed (`cresco-pki.sh crl`): the `identity` alarm is CRITICAL then, and WARN in
  the CRL's last week. CRL numbers only go up: the highest in force per CA is recorded, and a file whose CRL of a trusted
  CA is older, or missing, is refused at reload (the current revocations stay in force) and at start (`REFUSING TO
  START ... went BACKWARDS`).
- Expiry is alarmed by the `identity` health check (WARN in the renewal window, CRITICAL near and at expiry).
