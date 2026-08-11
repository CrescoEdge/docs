# Cresco Documentation

# 📖 &nbsp; [**Read the docs → crescoedge.github.io/docs**](https://crescoedge.github.io/docs/)

The **canonical documentation** for [Cresco](https://github.com/CrescoEdge) — a hierarchical, secure,
multi-tenant distributed agent mesh — is published as a full site at
**<https://crescoedge.github.io/docs/>**. This repository holds its source.

**All project information lives there:** architecture, every module/plugin, every callable action, both
client libraries, configuration, operations, and the engineering design documents.

### Jump in

| | |
|---|---|
| 🚀 **[Getting Started](https://crescoedge.github.io/docs/getting-started/installation/)** | build, run a mesh, configure |
| 🏗️ **[Architecture](https://crescoedge.github.io/docs/architecture/overview/)** | how the mesh works |
| 🔌 **[Modules & Plugins](https://crescoedge.github.io/docs/plugins/overview/)** | every component |
| ⚙️ **[Plugin Actions](https://crescoedge.github.io/docs/api/plugin-actions/)** | all 89 callable actions |
| 💻 **[Client Libraries](https://crescoedge.github.io/docs/clients/overview/)** | Java + Python SDKs |
| 🔐 **[Security & Multi-Tenancy](https://crescoedge.github.io/docs/architecture/tenancy/)** | identity, isolation, roles |

Built with [MkDocs](https://www.mkdocs.org/) + [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

## Build & serve

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install mkdocs-material

mkdocs serve      # live-reload dev server at http://127.0.0.1:8000
mkdocs build      # render static HTML into ./site
```

## Structure

```
docs/
├── index.md                 # landing
├── getting-started/         # installation, quickstart, configuration
├── architecture/            # overview, messaging, dataplane, discovery,
│                            #   security, tenancy, health, metrics
├── plugins/                 # one page per module (agent, library, controller,
│                            #   core, logger, repo, sysinfo, wsapi, stunnel)
├── api/                     # plugin actions (89), library API, MsgEvent
├── clients/                 # Java (clientlib) + Python (pycrescolib)
├── operations/              # deployment, testing
├── reference/               # configuration params (180+), glossary, design-doc index
└── design/                  # the full engineering design documents
```

Navigation and theme are defined in [`mkdocs.yml`](mkdocs.yml).

## Contributing

Keep pages accurate to the source. When a subsystem changes, update its architecture page and, where
relevant, the design document under `docs/design/`. New features should be documented as flag-gated and
default-off, matching the codebase's convention.
