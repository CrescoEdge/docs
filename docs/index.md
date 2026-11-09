# Cresco Documentation

![Cresco](images/cresco_logo.png){ width="220" }

**Cresco is a hierarchical, secure, multi-tenant distributed agent mesh.** It lets you deploy, connect,
and orchestrate software across many machines — from a single laptop to a global fabric of regions — with
a uniform messaging model, a streaming data plane, cryptographic identity, and per-tenant isolation.

This site is the **canonical documentation** for Cresco. All information about the project — architecture,
every module and plugin, every callable action, configuration, and all three client libraries — lives here.

!!! info "New here? Start with the [Architecture Overview](architecture/overview.md), then the [Quickstart](getting-started/quickstart.md)."

## What Cresco is

A Cresco deployment is a **mesh of agents**. Each agent is a JVM process (JDK 21) running an OSGi
(Apache Felix) host that loads the Cresco **controller** and a set of **plugins**. Agents self-organize
into a three-tier hierarchy:

| Tier | Role |
|------|------|
| **Agent** | A leaf node. Runs plugins, connects to a regional controller. |
| **Region** | A regional controller. Runs an embedded ActiveMQ broker; the agents in its region connect to it. |
| **Global** | A global controller. Federates regions together; the top of the hierarchy. A node can be `REGION_GLOBAL` (both). |

Nodes communicate two ways:

- **[MsgEvent](api/msgevent.md) control plane** — addressed, routed messages (commands, config, RPC,
  telemetry) between agents, regions, and globals over ActiveMQ queues.
- **[Data plane](architecture/dataplane.md)** — high-throughput pub/sub topics for streaming application
  data, with backpressure: a slow subscriber slows the writers, nothing is lost and order is kept.

## Key capabilities

- **Uniform messaging & routing** — every node addresses every other by `region_agent[_plugin]`; the
  [controller](plugins/controller.md) routes deterministically across a federated network of brokers. See
  [Messaging & Routing](architecture/messaging.md).
- **Plugins with self-describing actions** — functionality is packaged as OSGi bundles that expose
  `@CrescoAction` operations, aggregated fabric-wide into a [capability inventory](api/plugin-actions.md)
  (≈119 actions across 9 namespaces today).
- **Dynamic cost-aware routing** — a pushed link-state view and per-hop cost let the controller pick the
  cheapest path across a federated mesh, with automatic broker-bridge scaling under load. See
  [Dynamic Cost-Aware Routing](architecture/dynamic-routing.md).
- **Decentralized coordinators** — regions boot and run without a global (`global_optional`), multiple
  globals coexist, and failure detection + epoch-fenced quorum elect coordinators. See
  [Coordinator Decentralization](architecture/coordinator-decentralization.md).
- **Secure by default** — every node carries an identity-bearing X.509 leaf (`CN=agent, OU=region, O=tenant`),
  mutual TLS on every broker connection, end-to-end recipient-bound [message authentication](security/message-auth.md),
  signed plugins, and a `production` profile with operator-issued identities and FIPS 140-3 approved-only crypto. See
  [Security](security/index.md) and [Security & Identity](architecture/security.md).
- **Multi-tenancy** — end-to-end [tenant isolation](architecture/tenancy.md) with destination namespacing
  and role-based authorization (`SUPERUSER` / `TENANT`), enforced at every broker.
- **Health & metrics** — Felix [health checks](architecture/health.md) and a unified Micrometer
  [metrics](architecture/metrics.md) model with mesh-wide aggregation.
- **End-to-end tunnel tracing** — [stunnel](plugins/stunnel.md) tunnels are traced hop-by-hop through the
  broker network and streamed live (throughput + path) on a subscribable channel. See
  [Tunnel Path Tracing](architecture/tunnel-tracing.md).
- **Client SDKs** — drive the fabric from [Java](clients/java.md), [Python](clients/python.md), or
  [C++ / Arduino (ESP32)](clients/cpp.md) over a secure WebSocket.

## How this site is organized

- **[Getting Started](getting-started/installation.md)** — build from source, run a mesh, configure it.
- **[Architecture](architecture/overview.md)** — how the mesh, messaging, data plane, discovery, security,
  tenancy, health, and metrics work.
- **[Security](security/index.md)** — the operator's view of the 1.3 security model: defaults, profiles, PKI, message
  authentication, upgrade, API authentication, signed plugins, FIPS.
- **[Modules & Plugins](plugins/overview.md)** — a page for every module (`agent`, `library`, `controller`,
  `core`, `logger`, `repo`, `filerepo`, `sysinfo`, `wsapi`, `stunnel`, `executor`).
- **[API Reference](api/plugin-actions.md)** — all plugin actions, the Library API, the MsgEvent structure,
  and the full [configuration parameter](reference/configuration.md) reference.
- **[Client Libraries](clients/overview.md)** — Java, Python, and C++ (Arduino/ESP32) SDK references.
- **[Operations](operations/deployment.md)** — deployment, testing and troubleshooting.
- **[Reference](reference/glossary.md)** — glossary and the underlying design documents.

## Project layout

```
cresco/
├── code/            # source (one OSGi module / git repo per directory)
│   ├── agent/       # ★ HEADLINE repo — OSGi host that composes every bundle into the runnable agent uber-jar
│   ├── library/     # the plugin & controller API (io.cresco.library)
│   ├── controller/  # the fabric brain (messaging, discovery, security, health, db)
│   ├── core/        # controller/JVM lifecycle service
│   ├── logger/      # logging bootstrap
│   ├── repo/        # plugin repository
│   ├── filerepo/    # file replication/repository plugin
│   ├── sysinfo/     # system info + benchmark plugin
│   ├── wsapi/       # WebSocket API (external client bridge, :8282)
│   ├── stunnel/     # secure TCP tunnel plugin
│   ├── executor/    # OS process / command executor plugin
│   ├── clientlib/   # external Java SDK
│   └── pycrescolib/ # external Python SDK
├── cppcrescolib/    # external C++ / Arduino (ESP32) SDK
└── run/             # staged agent jar + launch scripts + tests
```

Each module is its own repository under [github.com/CrescoEdge](https://github.com/CrescoEdge), released and
versioned independently. The headline repo is **[CrescoEdge/agent](https://github.com/CrescoEdge/agent)** —
the final bundle that composes every other module into the single runnable artifact; start there.
