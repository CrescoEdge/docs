# Discovery

Before a node can join the mesh it must **find** a parent (region/global) and prove it is **allowed** to
join. Cresco does this with an authenticated discovery exchange, keyed by a pre-shared secret in the standard
profile and by the operator's PKI in production, implemented under the controller's `netdiscovery` package.

## How a node finds a parent

| Mechanism | When |
|-----------|------|
| **UDP broadcast/multicast** (`UDPDiscoveryEngine`) | dynamic discovery on a local network — a joining node broadcasts a `DISCOVER`, listening controllers respond |
| **Static/TCP** (`TCPDiscoveryEngine`, `TCPDiscoveryStatic`) | directed discovery to a known host:port (e.g. `global_controller_host`) — used across networks where broadcast doesn't reach |

Discovery exchanges `DiscoveryNode` records carrying the peer's region/agent identity, IP/port, agent
count, a validator, and a measured latency. Discovery runs per tier: `AGENT`, `REGION`, `GLOBAL`,
`NETWORK`.

## The join secret (standard profile)

A node proves it may join by demonstrating knowledge of the **tier join secret**. There is one secret per
tier:

| Config | Gate |
|--------|------|
| `discovery_secret_agent` | joining as an agent |
| `discovery_secret_region` | joining as a region |
| `discovery_secret_global` | joining as a global |

Locator messages (`DISCOVER` / `DISCOVERED`) carry a validator keyed from the secret (AES-256-GCM, a fresh IV each time,
the key derived with PBKDF2-HMAC-SHA-384): they only find a controller. Trust is established by **discovery v2**, a
mutually authenticated, replay-proof exchange of four messages over fresh 32-byte nonces from both sides:

| Message | From | Carries |
|---|---|---|
| `HELLO` | joiner | type, its nonce, its region/agent and certificate chain |
| `CHALLENGE` | responder | its nonce, region/agent, chain, broker port, and MAC "S1" over the transcript |
| `PROOF` | joiner | MAC "C" over the transcript and its own whole chain |
| `RESULT` | responder | ok or the reason, the result chain, MAC "S2" |

The MACs are HMAC-SHA-384 under a key derived from that tier's secret (PBKDF2-HMAC-SHA-384, 210,000 iterations). The
transcript covers both nonces, both identities, both leaf keys, the responder's broker port and both whole chains, so a
replayed or altered exchange fails, and neither side records a name, pin, bridge peer or enrollment before the other has
proven itself. A node that doesn't hold the right secret cannot join.

!!! danger "Set your secrets"
    If `discovery_secret_*` are unset a node generates random values and nobody can join it. For a real
    fabric, set fixed shared values. See [Configuration](../getting-started/configuration.md).

## Discovery in production

Under `security_profile=production` no discovery secret is used or accepted (one set refuses the start). The same four
messages run on the operator's PKI: each side signs the transcript with its provisioned identity key, and its chain must
verify to `trust_anchors` as what it claims (a GLOBAL answer only from a certificate carrying `cresco:role:global`).
Nothing is issued, enrolled or pinned on first use. See [Security profiles](../security/profiles.md#discovery-in-production).

## Certification (establishing trust at join)

Discovery is also where nodes exchange the cryptographic material that later secures the broker
connections. In the standard profile (see [Security & Identity](security.md)):

- **Direct trust (default):** peers exchange leaf certificates during discovery; a joiner pins the controller it
  joined (its key, and under the regional CA its region CA), and a different key for that name later is refused.
- **Regional-CA (`security_regional_ca=true`):** the responder (a region/global acting as an issuing
  authority) signs the joiner a leaf certificate under its CA and returns the chain; the joiner installs
  it and trusts the issuing CA. This collapses trust material from O(nodes²) to O(regions).

The authenticated exchange is the enrollment gate for both: nothing is issued or trusted before it completes. The
legacy single-message exchange of earlier builds is accepted only inside the [upgrade window](../security/upgrade.md),
and never between controllers.

## One controller per region

The discovery secret proves membership, not a name: every region controller holds the global's secret, every agent its
region's. So from agent release 1.3.380 the node that records a peer also asks for proof of its key and holds three
rules (standard profile; under `security_profile=production` the operator's PKI already names every controller):

- **Proof of possession.** Each side signs the exchange with the key of the certificate it presents, bound to both
  nonces, both identities, the tenant and the whole chain, so a signature verifies for no other exchange. Anything new
  -- a name, another key for a name, a replacement -- is recorded only on that proof. A name and key that proved itself
  once is refused without proof afterwards (after `discovery_proof_grace_days`, default 7, the allowance for a node put
  back on an earlier release).
- **One controller per region.** The first identity recorded as a region's controller is that region's one controller
  on this node; another one is refused, whatever tenant it states. The same key under another agent name of the region
  is the same controller and replaces the former name. A key speaks for one region.
- **One global per node.** A node takes the first global it joins; any other responder of a global exchange is refused.

Only the operator replaces a recorded controller or moves a region to another global: the line
`<region>/<agent>=<key fingerprint>` in `<data>/security/peer-trust.json.rebind` on each node that recorded it (the
refusal prints it), a signed rebind statement (`controller_rebind_signers`), or a scoped
`controller_trust_anchors` entry `<region>/<agent>=<fingerprint>`. The decision and the record are one step and one
durable write, so of two joins at the same instant one is recorded. Cost: two signatures and two verifications per
discovery exchange, nothing per message.

An agent with no pin takes the first controller that answers its first discovery, holds the secret and proves its key.
Fleets that cannot accept that ship `controller_trust_anchors` or `controller_rebind_signers` to their agents, or use
the production profile.

## After discovery

Once a parent is found and trust is established, the node opens its [messaging](messaging.md) channels
(inbox queue + producers), and — if it is a controller — starts its broker and the **federation bridge**
to its parent. Liveness [health](health.md) checks then keep the parent link monitored.
