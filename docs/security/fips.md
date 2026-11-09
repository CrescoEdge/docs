# FIPS 140-3

FIPS 140-validated cryptography is a requirement for Cresco's secure deployments. The agent carries the validated
modules and, in the production profile, runs the whole JVM in **FIPS approved-only mode**. Source: the agent README
("Crypto modules and FIPS 140-3 mode") and `controller/docs/production-deployment.md` section 6.

## The modules

| Module | Version | Validation | Where |
|---|---|---|---|
| Bouncy Castle FIPS Java API (BC-FJA): `bc-fips`, with `bcpkix-fips`, `bcutil-fips`, `bctls-fips` | 2.1.3 (a patched 2.1.2 under the certificate), 2.1.13, 2.1.8, 2.1.25 | CMVP #4943 (FIPS 140-3, interim) | every platform; native AES/SHA/DRBG code on Linux x86_64 |
| Amazon Corretto Crypto Provider, FIPS build (ACCP-FIPS), over AWS-LC FIPS 3.0.0 | 2.5.0 | CMVP #4816 (AWS-LC) | Linux x86_64 and aarch64 |

The agent jar carries them **unmodified** under `crypto/`: a FIPS module checks its own jar, so none is repackaged into
the agent or a bundle. At start the agent writes each to a new owner-only temporary directory (`cresco-crypto-<pid>-...`,
removed when the agent exits; a start also removes this user's directories whose agent is no longer running), checks it
against the SHA-384 the build's known-vulnerability gate recorded (covered by the release signature) and appends it to
the system class path (the jar's `Launcher-Agent-Class`). The OSGi framework exports the Bouncy Castle packages from there, and the controller
and wsapi import them instead of embedding Bouncy Castle. The SBOM lists each module as `crypto/<jar>` with its Maven
coordinates.

## Modes

- **Standard profile**: the JDK's providers serve crypto and TLS as before; Bouncy Castle FIPS is only the X.509 library
  certificates are made with.
- **`security_profile=production`** (or `crypto_fips_mode=true`): FIPS approved-only mode.
    - Every JCA provider that is not a FIPS module is removed (SUN, SunJCE, SunEC, SunRsaSign, ...).
    - ACCP-FIPS comes first where it runs, offering only its approved algorithms (AES-GCM, AES-KWP, SHA-2,
      HMAC-SHA-2, HKDF, ECDH, EC/RSA keys, ECDSA/RSA/RSA-PSS signatures with SHA-2); BC-FJA, in approved-only mode, serves
      everything else (X.509, PKIX, keystores, PBKDF2, the DRBG).
    - TLS (the broker's, wsapi's) is SunJSSE over those modules, TLS 1.3 only; `crypto_fips_tls=bcjsse` uses BCJSSE
      instead (TLS 1.3 and 1.2, the TLS key derivation inside BC-FJA; slower).
    - The operating system's entropy only seeds the DRBG.
    - A non-approved algorithm is refused: at start the agent checks that MD5, ChaCha20, RC4, Blowfish, X25519 and
      MD5withRSA are not available and that the modules work, and does not start otherwise. The log says
      `crypto: FIPS 140-3 approved-only mode: <modules>; ... MD5 refused`.
    - Refused: `crypto_fips_mode=false` with the production profile, `cresco_ssl_provider=CONSCRYPT` and wsapi's
      `wsapi_ssl_provider=OPENSSL` (BoringSSL is not a FIPS module).
    - Plugin jar signatures are verified through the installed providers only (CMS with Bouncy Castle FIPS, digests by
      the modules; SHA-256/384/512 only), never the JDK's internal jar verifier ([Signed plugins](signed-plugins.md)).

The controller does not take the mode on trust: before a `production` node starts it checks that every JCA provider is
one FIPS mode leaves (`BCFIPS`, `BCJSSE`, `CrescoACCP-FIPS`, `CrescoFIPS`, `SunJSSE`), that Bouncy Castle FIPS is in
approved-only mode and passed its self-tests, that ACCP-FIPS (where installed) is the FIPS build and healthy, that MD5 is
refused and that SHA-384 gives its known answer, and logs `FIPS 140-3 approved-only mode verified (self-tests passed,
MD5 refused): <providers>`. Otherwise it refuses to start: `REFUSING TO START: security_profile=production runs in FIPS
140-3 approved-only mode, and this JVM is not in it: <why>`.

| Setting | Default | Meaning |
|---|---|---|
| `crypto_fips_mode` | production: true, else false | FIPS approved-only mode |
| `crypto_fips_provider` | `auto` | `auto`: ACCP-FIPS first where it runs; `accp`: refuse to start without it; `bcfips`: BC-FJA only |
| `crypto_fips_tls` | `sunjsse` | `bcjsse`: BCJSSE over BC-FJA |

## Keystores

In FIPS mode keystores are **BCFKS**, the module's own format: a PKCS#12 file's integrity MAC derives its key with the
PKCS#12 KDF, which FIPS does not approve, so approved-only mode cannot check one. The `PKCS12` keystore name is served by
BCFKS (code keeps asking for PKCS12); a PKCS#12 file is refused with how to convert it. Convert each keystore once,
offline, with the same password:

```sh
java -jar agent.jar --crypto convert-keystore node.p12 node.p12.pass node.bcfks
```

PEM trust anchors, CRLs and PEM plugin-signing bundles need nothing. `java -jar agent.jar --crypto status --fips` shows what
a node would run with.

## Throughput

`scripts/bench-crypto.sh <agent.jar>` (agent repository) measures a host in every mode; `scripts/bench/dataplane_perf.py`
measures the dataplane through running agents, on one node or across the broker bridge. One thread, 2026-11-08:

| | AES-256-GCM MB/s | TLS 1.3 MB/s | RSA-3072 sign/s | ECDSA P-384 sign/s |
|---|---|---|---|---|
| Xeon 8480CL, standard (JDK) | 8717 | 2793 | 425 | 1064 |
| Xeon 8480CL, FIPS: ACCP-FIPS + BC-FJA, SunJSSE (default) | 11857 | 3623 | 1403 | 12662 |
| Xeon 8480CL, FIPS: BC-FJA only, SunJSSE | 4547 | 916 | 333 | 2433 |
| Xeon 8480CL, FIPS: BC-FJA + BCJSSE | 4547 | 818 | 349 | 2515 |
| Apple M3 Max, standard (JDK) | 4942 | 1890 | 519 | 1577 |
| Apple M3 Max, FIPS: BC-FJA only | 116 | 56 | 422 | 3570 |

End to end, on one DGX node (Xeon 8480CL, 8 cores; `dataplane_perf.py`, the Python client, 2000 x 256 KiB frames;
2026-11-08, the build before dataplane backpressure): a production (FIPS) mesh moved 299 MiB/s through one agent and
280 MiB/s from one agent to another across the broker bridge; a standard mesh 246 and 219 MiB/s. On Linux the default FIPS mode is faster than the JDK. Where ACCP-FIPS has no build (macOS, Windows) BC-FJA serves
everything in pure Java and TLS is far slower (the agent logs a WARNING): run production on Linux.

## Clients

Clients connect to a FIPS node with TLS 1.3, AES-GCM and P-256/P-384, which OpenSSL (pycrescolib) and the JDK (clientlib)
offer by default. A client that must itself be FIPS-validated runs on a FIPS-validated platform: Python against an OpenSSL
3 FIPS provider, Java with a FIPS provider installed in its JVM (clientlib names no provider; use a PEM client
certificate and key there).
