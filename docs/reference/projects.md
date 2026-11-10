# Projects built on Cresco

Systems that run on a Cresco fabric as plugins and clients. Each has its own documentation; this page says what each
one is and which parts of Cresco it relies on.

| Project | What it is | Documentation |
|---|---|---|
| **GaiaKeep** | A global file system built for agents, not people: it keeps versioned datasets durable and streams any version wherever it is needed, to a GPU node, a partner site or an agent's working cache. | [gaiakeep.github.io/docs](https://gaiakeep.github.io/docs/) |
| **panAtlas** | A reference atlas for multi-modal evaluation across every domain of knowledge: a provenance-first knowledge graph that links concepts, datasets, models, their training curricula and the benchmarks that measure them. | [thepanatlas.github.io/docs](https://thepanatlas.github.io/docs/) |

## GaiaKeep

GaiaKeep's storage core is a Cresco plugin (`io.cresco.gfs`) that runs over the mesh across institutions and sites.
Cresco carries its transport, identity and tenant isolation: control requests ride [MsgEvent](../api/msgevent.md)
and data rides the [data plane](../architecture/dataplane.md). Its flows between site tenants are the
[named cross-tenant flows](../architecture/tenancy.md#named-cross-tenant-flows) of the tenant policy, and it follows
the fabric's one [crypto policy](../architecture/security.md#crypto-policy-one-set-of-allowed-algorithms-per-fabric).
[filerepo](../plugins/filerepo.md) is a client of its live tier.

## panAtlas

panAtlas is a distributed system on a Cresco fabric (region `panatlas-global`). Every registered dataset is managed by
its own instance of one Cresco plugin (`io.cresco.panatlas`), running beside the data at the site that holds it. The
managers report structure, never content, up the mesh over the [control plane](../architecture/messaging.md); their
actions carry `@CrescoAction` annotations, so they appear in the [capability inventory](../api/plugin-actions.md) and
any client can call them as tools. The hub reaches them through [wsapi](../plugins/wsapi.md) with
[pycrescolib](../clients/python.md). GaiaKeep, on the same mesh, is designed as its durable storage backend.
