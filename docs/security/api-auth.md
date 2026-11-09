# API authentication

API clients (pycrescolib, clientlib, tools such as GaiaKeep) reach the fabric through wsapi's three WebSocket endpoints:
the API socket (`/api/apisocket`), the dataplane (`/api/dataplane`) and the log stream (`/api/logstreamer`). Every
upgrade request authenticates, on every endpoint. Source: the wsapi README.

## TLS first

wsapi serves `wss://` with its own certificate (EC P-384 by default) and writes `ws.crt.pem` and `ws.fingerprint`
(SHA-256 and SHA-384) in its plugin data directory. Both clients verify the certificate on every connection by default
and send nothing to a server they do not trust: give each client the fingerprint (pycrescolib `fingerprint=`, clientlib
`pinnedFingerprint(...)`, or `CRESCO_TLS_FINGERPRINT`) or the PEM file (`ca_file=` / `caFile(...)`).

## Credentials (standard profile)

| Credential | How it is sent | Use it for |
|---|---|---|
| **user and password** | `Authorization: Basic` | people; each user has one tenant and one role, `admin` or `user` |
| **API token** | `Authorization: Bearer cwt1_<id>.<secret>` | tools and automation (never throttled per user) |
| **service key** (legacy) | `cresco_service_key` header | an `admin` of the agent's tenant; turn it off with `wsapi_service_key_enabled=false` |

Users and tokens live in the owner-only credential file (`ws.users`, or `wsapi_users_file`), managed with the tool in the
wsapi jar (passwords are read without echo, never from the command line):

```bash
java -cp wsapi.jar io.cresco.wsapi.auth.CredentialTool add    <ws.users> <user> <tenant> <admin|user>
java -cp wsapi.jar io.cresco.wsapi.auth.CredentialTool grant  <ws.users> <user> <stream|prefix*>
java -cp wsapi.jar io.cresco.wsapi.auth.CredentialTool token add <ws.users> <user> [--expires-days N] [--label L]
java -cp wsapi.jar io.cresco.wsapi.auth.CredentialTool token revoke <ws.users> <token id>
java -cp wsapi.jar io.cresco.wsapi.auth.CredentialTool list   <ws.users>
```

- Passwords are stored as PBKDF2-HMAC-SHA-384 hashes (600,000 iterations, a random salt per user); a token is stored as
  its SHA-384 and shown once.
- A caller of another tenant than the agent's is refused (403), except an `admin` of a superuser tenant.
- A `user` sees only its own dataplane streams (`u:<user>:...`) and those granted to it; only an admin may set log
  levels or log routing.
- Failed attempts are limited per source so that strangers can slow a user's password logins at most, never lock them
  out (except from the user's own network); a valid API token is always accepted.
- Changes to the credential file reach open connections: a revoked token, a removed user, a changed password or role
  closes the connections it opened (WebSocket 1008), within `wsapi_revocation_check_ms` (1 s).
- Every message an API client causes is sent on its behalf: the controller stamps the verified caller
  `api:<tenant>/<user>` with its role, and every receiver authorizes the action against it.

## Client certificates (production profile)

With `security_profile=production` wsapi accepts **client certificates only** (mutual TLS on every endpoint):

- a client presents a certificate chaining to a CA of `wsapi_client_ca`, or TLS refuses the connection;
- the certificate is the caller, from its SANs: exactly one `cresco:api:<tenant>/<user>`, exactly one
  `cresco:role:admin` or `cresco:role:user`, any number of `cresco:grant:<stream or prefix*>`; `tools/cresco-pki.sh api`
  issues them;
- passwords and the service key are refused (401), API tokens too unless `wsapi_allow_tokens=true`;
- revocation: `wsapi_client_deny_file` (SHA-256 fingerprints) and `wsapi_client_crl` are re-read when they change, and
  close the open connections of a revoked certificate within a second or two.

```python
from pycrescolib.clientlib import clientlib
c = clientlib("node.example", 8282, fingerprint=FP, client_cert="admin.pem", client_key="admin.key")
```

```java
CrescoClient c = CrescoClient.builder("node.example", 8282).pinnedFingerprint(fp)
        .clientCertificatePkcs12(Path.of("admin.p12"), password).build();
```

In FIPS approved-only mode (production) wsapi's TLS is TLS 1.3 with AES-GCM and P-256/P-384 key exchange; both clients
offer that by default, so they need nothing new.
