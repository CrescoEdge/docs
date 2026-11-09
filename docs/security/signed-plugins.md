# Signed plugins

Code that arrives over the network is code execution on every node that installs it, so with security on (the default)
every plugin jar that did not come embedded in the agent jar is installed only from an owner-only private copy that
verifies against `plugin_signing_truststore`. Sources: `controller/docs/caller-authorization.md` sections 5 and 8, the
repo README.

## What is verified

- `pluginupload`, `pluginadd` from a repository or an absolute path, `getjar` fetches, `plugindownload`,
  `controllerupdate`, the repository cache and `savetorepo` at the global: the jar must be signed, every entry, by a
  signer chaining to `plugin_signing_truststore` (`JarSignatures.verify`). The cache names a jar by its own hash; a
  hash a caller supplies is advisory.
- The repo plugin's `putjar` stores exactly the bytes it verified, and `repolist`/`getjar` list and serve only jars that
  pass the same check; unsigned, partially signed, tampered and foreign-signed jars are refused (`status 9`,
  `refusal=signature`).
- A signature carrying a trusted RFC 3161 timestamp (the TSA's certificate has the time-stamping extended key usage and
  chains to the trust store) is judged at the stamped time, so a stored jar stays installable after its signing
  certificate expires. Without one, re-sign jars before the signing certificate expires.
- Jars embedded in the agent jar are trusted as part of it (the release itself is signed: `scripts/sign-release.sh`,
  `scripts/verify-release.sh` in the agent repository).

## Settings

| Setting | Default | Meaning |
|---|---|---|
| `plugin_signing_required` | `broker_security_enabled` (true) | `false`: unsigned jars are installed (development only; a WARNING every boot) |
| `plugin_signing_truststore` | (none) | the anchors jar signatures must chain to: a PEM bundle (no password; required in production), or PKCS12 with `plugin_signing_truststore_password` (standard profile, outside FIPS mode, only) |

With signing required and no trust store, every jar arriving over the network is refused.

## Signing a jar

```sh
jarsigner -keystore codesign.p12 -storetype PKCS12 -digestalg SHA-384 -sigalg SHA384withRSA \
  -tsa https://<your RFC 3161 TSA> plugin.jar codesign
```

Keep the code-signing CA separate from the node PKI; give nodes only its certificate (the PEM trust bundle).
