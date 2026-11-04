# DDMS-04 · Reservoir DDMS

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). What the Reservoir
> DDMS holds, what users can do with it, how data gets in, and when it's the
> right choice.

## TL;DR

- The Reservoir DDMS stores **subsurface models**: the 3D geological and
  reservoir models built from seismic, wells and cores, used to estimate volumes
  and simulate production. Their industry standard is **RESQML**.
- A model is a **connected graph** of objects: grids, surfaces, properties and
  the links between them. The DDMS keeps the graph, and lets users read **one
  property of one grid** without the rest.
- Work is organised in **dataspaces**, for example `northsea/base-case`. Each is
  a unit of access, can be **cloned** to try a new scenario, and can be
  **locked** as read-only.
- **Like Seismic, it's a separate store inside ADME.** Its objects become
  searchable only when you **build an OSDU manifest** from them and ingest it.
- **Not in ACZ.** Analytics on model properties needs your own export.
- Generally available, and **on all ADME instances by default**.

## 1. What it is and what's inside

A reservoir model is a 3D grid of the subsurface, often millions of cells,
with rock and fluid properties in each cell. Geoscientists build it from
seismic interpretation, well logs and core analyses; engineers use it to
estimate how much oil or gas is there and to simulate how it will flow. For the
background, see
[02 §8](../fundamentals/02-energy-domain-primer.md#8-models-and-production-briefly).

**In plain words, how a model comes together:**
1. On the seismic, geologists pick the **top of the reservoir** and **three
   faults**.
2. Between those surfaces they build a **3D grid**: millions of small boxes that
   fill the reservoir.
3. They give every box values, such as **porosity** and **permeability**, from
   well logs and core analyses.
4. Engineers copy the model to test a **high case**, run simulations, and the
   team approves one version.

The Reservoir DDMS keeps all of this: the surfaces, the grid, the values, and
how they connect. Any modelling application can open the model, a user can
read just the porosity, and each scenario stays separate. The rest of this
section names these parts the way RESQML does.

Microsoft describes the Reservoir DDMS as storage for **seismic and well
interpretation, structural modelling, geological modelling and reservoir
modelling, including reservoir simulation input**.

As in every DDMS, there are two parts:
- **Objects** are the model's building blocks: a fault, a horizon, a grid, a
  property. Each is described in RESQML (XML).
- **Arrays** are the bulk data: grid geometry and the values of each property,
  for example porosity in every cell. They're often far larger than the
  objects that describe them.

### How RESQML organises a model

RESQML separates **what exists** from **someone's view of it**, from its
**geometry**, from the **values** on that geometry:

| Layer | Meaning | Example |
|---|---|---|
| **Feature** | Something real in the subsurface | Fault F1; the top of the reservoir |
| **Interpretation** | One person's or one study's view of a feature | "F1 as interpreted in the 2025 study" |
| **Representation** | The geometry of an interpretation | A triangulated surface; a 3D grid |
| **Property** | Values on a representation | Porosity, permeability, saturation per grid cell |

A property means nothing without its grid, and a grid nothing without the
interpretations it was built from. That's why the DDMS keeps **objects, arrays
and the links between them together**.

**Data types** (the RESQML objects the M26 code maps to OSDU catalogue kinds,
examples):

| Type | What it is | Values stored separately in the DDMS? |
|---|---|---|
| **HorizonInterpretation**, **FaultInterpretation** | A study's view of a layer boundary or a fault | No: description and links |
| **GeobodyInterpretation**, **EarthModelInterpretation** | A body of rock, or the whole model as one interpretation | No: description and links |
| **StratigraphicColumn**, **StratigraphicUnitInterpretation** | The sequence of rock layers used in the model | No: description and links |
| **IjkGridRepresentation**, **UnstructuredGridRepresentation** | The 3D grid: regular (i, j, k) or unstructured | **Yes:** cell geometry |
| **GenericRepresentation**, **SubRepresentation** | Surfaces, point sets, or part of a grid | **Yes:** coordinates, or indices into another representation |
| **GenericProperty** | Values on a representation, for example porosity per cell | **Yes:** one value per cell or node |
| **TimeSeries** | Time steps, for example for simulation input | No: the time steps themselves |

**How it's organised:**

```text
Reservoir DDMS (in one data partition)
└── dataspace: northsea/base-case        ← unit of access: owners, viewers, legal tag
    ├── objects (RESQML)                  features, interpretations, representations, properties
    │     FaultInterpretation "F1 (2025)" ──▶ GenericRepresentation "F1 surface"
    │     IjkGridRepresentation "Model grid" ◀── GenericProperty "PORO"
    │                                        ◀── GenericProperty "PERMX"
    └── arrays                            grid geometry, surface coordinates, property values
└── dataspace: northsea/high-case        ← a clone, changed for another scenario
```

Dataspace names have **two parts**, such as `project/scenario`. Each dataspace
is a separate scope: the same object can exist in two dataspaces without
conflict.

### Where the data lives, and how many copies

**The Reservoir DDMS is part of ADME, not a separate product.** It's integrated
with ADME's other services and runs in your data partition. Its storage is
managed by Microsoft and isn't accessible directly.

| Shared with the rest of ADME | Its own |
|---|---|
| The same instance, billing and Microsoft-managed storage | Its own **APIs**: ETP (WebSocket) and REST (`/api/reservoir-ddms/v2`) |
| The same partition | Its own **store** of objects and arrays, organised in dataspaces |
| The same sign-in and tokens | Its own **access unit**: the dataspace |
| The same entitlement groups and legal tags | **Not in Search**, until you ingest a manifest (below) |

**Copies come from your choices:**

| Copy | Why |
|---|---|
| **The original RESQML package** (EPC + HDF5 files) in the File service | Optional evidence of what was delivered |
| **The model in a dataspace** | The working copy that applications read and write |
| **Cloned dataspaces** | One per scenario; each clone duplicates the model's content, so plan its storage |

### How it relates to ADME's catalogue

**Like Seismic Store, the Reservoir DDMS keeps its own store, and Search
doesn't see it. Unlike Seismic Store, it can build the catalogue records for
you.**

```text
RESERVOIR DDMS (dataspaces: objects + arrays, not searchable)
    northsea/base-case: FaultInterpretation, IjkGridRepresentation, GenericProperty…
                │
                │ 1. POST /manifests/build  (choose dataspaces or objects)
                ▼
    OSDU manifest: one catalogue record per object
                │
                │ 2. manifest ingestion (Workflow API)
                ▼
CATALOGUE (Storage + Search: OSDU records, searchable)
    work-product-component--FaultInterpretation, --IjkGridRepresentation, --GenericProperty…
```

1. **Build** the manifest from chosen dataspaces or objects. You can add tags
   and technical assurances (quality flags), and ask it to create references
   that don't exist yet in the catalogue.
2. **Ingest** it like any manifest
   ([DI-04 §4](../data-integration/04-building-and-loading-records.md#4-route-b-manifest-ingestion)).
3. Users **find** the model's objects in Search, alongside wells and seismic,
   and **read** them from the dataspace.

**Nothing does this automatically** at M26. When the model changes, rebuild
and re-ingest the manifest, or the catalogue describes an older state.

**Compared with the other DDMS:**

| | Wellbore DDMS | Seismic DDMS | Reservoir DDMS |
|---|---|---|---|
| **Catalogue records** | Created by the DDMS | Created by you; conversion workflows add those of converted copies | **Built by the DDMS on request**, ingested by you |
| **Access unit** | The record's ACL | Subproject or dataset | **Dataspace** |
| **Stays in step with the catalogue** | Automatically | Only through your pipeline | When you rebuild and re-ingest the manifest |

### What users actually read

| What | What it is | Who uses it |
|---|---|---|
| **Grid geometry** | The cells of the model: their corners and how they connect | Modelling and simulation applications |
| **Property arrays** | Porosity, permeability, saturation, pressure: one value per cell | Engineers running volume estimates and simulations |
| **Horizon and fault surfaces** | The structural framework of the model | Geologists, and checks against seismic |
| **Stratigraphic column** | The order and names of the rock layers | Everyone working on the model |
| **Simulation input** | The grid, properties and time steps a simulator needs | Reservoir engineers |
| **The graph** | What a property belongs to, what was built from a fault | Data managers, tracing impact and lineage |

### What isn't in the Reservoir DDMS

| Data | Where it lives instead |
|---|---|
| Seismic volumes | [Seismic DDMS](03-seismic.md) |
| Well logs and trajectories | [Wellbore DDMS](02-wellbore.md) |
| Core and fluid analyses | [RAFS DDMS](05-rock-and-fluid-samples.md) |
| Simulation results stored as RESQML properties over time steps | In the Reservoir DDMS: Microsoft lists simulation outputs in its scope |
| Simulator-native result files, reports | File service, described by Document or other records |
| Petrel projects | [Petrel Data Services](06-petrel-data-services.md) |

## 2. What users and applications can do

| Operation | REST endpoint (under `/api/reservoir-ddms/v2`) | Example |
|---|---|---|
| List and create dataspaces | `GET` and `POST …/dataspaces` | A dataspace per project and scenario |
| **Clone** a dataspace | `POST …/dataspaces/{id}/clone` | Start a high-case scenario from the base case |
| **Lock** a dataspace | `POST` and `DELETE …/dataspaces/{id}/lock` | Make an approved model read-only |
| List objects by type | `GET …/dataspaces/{id}/resources/{type}` | Every grid in a dataspace |
| Navigate the graph | `…/resources/{type}/{guid}/targets` and `…/sources` | "Which properties belong to this grid?" |
| Read an object | `GET …/dataspaces/{id}/resources/{type}/{guid}` | The description of one fault |
| **Read an array** | `GET …/resources/{type}/{guid}/arrays/{path}`, with `starts` and `counts` for a slice | Porosity values of one grid, or of part of it |
| Write objects and arrays | `PUT …/dataspaces/{id}/resources` and `…/resources/arrays` | Add a new property |
| Write in a **transaction** | `POST`, `PUT` (commit), `DELETE` (roll back) `…/transactions` | Several changes that must succeed together |
| Build a manifest | `POST …/manifests/build` | Make the model searchable (§1) |
| Delete a dataspace | `DELETE …/dataspaces/{id}` | Remove an abandoned scenario |

**Transactions:** changes to objects should be made inside a transaction. Only
one transaction can be open per dataspace at a time, and an idle transaction is
rolled back automatically, after 300 seconds by default.

**ETP or REST?**
- **ETP**, the Energistics Transfer Protocol over WebSocket, is the fast,
  binary protocol that modelling applications use to stream whole models in
  and out.
- **REST** is simpler for scripts, notebooks and web applications that read or
  write a few objects and arrays.

## 3. How data gets into the DDMS

```text
RESQML package (EPC + HDF5) ──ETP client──▶ dataspace
Modelling application ─────────ETP──────────▶ dataspace
Script or notebook ────────────REST─────────▶ dataspace (objects + arrays)
                                                 └──▶ manifest ──▶ catalogue
```

| Way in | What it does | Use when |
|---|---|---|
| **ETP client** | Loads a RESQML package (EPC + HDF5) into a dataspace. Microsoft's tutorial uses OSDU's prebuilt client container | Loading delivered model files |
| **Modelling applications** | Read and write models over ETP | Day-to-day modelling work. **Check:** which of your applications support the Reservoir DDMS |
| **REST** | Writes individual objects and arrays, in transactions | Scripts, and small changes from notebooks or services |

**Whichever way in, keep the order:**
1. Create the dataspace, with its owners, viewers and legal tag.
2. Load the model.
3. Build the manifest and ingest it, so the model is findable.

**The DDMS checks structure, not content.** It checks that objects and their
links are well formed, but not that a model is geologically sensible. Quality
checks stay with the modellers and data managers.

## 4. When new or changed data arrives

| Situation | What to do in ADME | Why |
|---|---|---|
| **A new model** | A new dataspace | Its own scope and access |
| **A new scenario** of an existing model | **Clone** the dataspace, then change the clone | The base case stays intact |
| **An approved model** | **Lock** its dataspace | Read-only, so the approved version can't change |
| **An updated property** | Write it in a transaction, in a working dataspace | All related changes succeed or fail together |
| **A model that should be findable** | Build and ingest the manifest, again after each significant change | The catalogue only knows what was ingested |
| **An abandoned scenario** | Delete its dataspace | Each clone duplicates the model, so it costs storage |

**Nothing happens automatically.** The DDMS doesn't clone, lock or catalogue by
itself. Modellers, data managers or your pipeline do each step.

## 5. Typical needs it answers, and what to point users to

| The need | Point to |
|---|---|
| Every modelling tool keeps its own copy of the model | One shared model in a dataspace, read over ETP |
| Scenarios are hard to track | One dataspace per scenario: clone to branch, lock to approve |
| Nobody outside the modelling team can find the models | Build and ingest manifests, so models appear in Search |
| A user needs one property, not the whole model | Read that property's array through REST |
| "What depends on this fault?" | Navigate the graph: targets and sources |
| Reports on model properties across fields | A lakehouse copy of selected properties, through your own export (§6) |

## 6. Alternatives, and when each wins

| Option | Wins when | Falls short when |
|---|---|---|
| **File service: RESQML packages (EPC + HDF5)** | Models kept as evidence or exchanged, rarely opened | Anything shared or reused: the package is read whole, and the links inside it aren't visible to anyone else |
| **Reservoir DDMS** | Models shared between tools and people, with scenarios and approvals | Analytics across many models |
| **Lakehouse copy** | Selected properties or summaries as tables, for example one row per cell, or average porosity per zone | As the model's home: a table loses the grid, the surfaces and the links between them |

**Without the Reservoir DDMS, models can still be stored in ADME** as RESQML
packages through the File service. They're then only an **archive**: opened
whole, invisible to Search beyond the file's record, and without dataspaces,
scenarios or graph navigation.

**Why not keep models only in the lake?** Grid properties can be flattened
into tables, which works better than for seismic. What you give up when the
lake is the **main** home:

| You lose | Why it matters |
|---|---|
| The model's structure | Grids, surfaces and their links become disconnected tables |
| ADME's governance | Dataspace access and legal tags don't apply in the lake |
| Modelling applications | They read and write over ETP, not from your lake |
| A managed export | ACZ doesn't export reservoir data, so you build the copy yourself |

So: **lake copies of selected properties for reporting and ML, yes; the lake as
the model's home, no.**

## 7. Governance and access

- **Access is set per dataspace.** When you create one, you give it owners,
  viewers, legal tags and the data's countries, using ADME's entitlement groups
  and legal tags.
- **Owners can write; viewers can only read.** Locking a dataspace makes it
  read-only for everyone.
- **The catalogue records have their own ACLs.** When you build and ingest a
  manifest, give the records the **same data groups** as the dataspace
  ([DG-02 §4](../data-governance/02-access-design.md#4-acls-on-every-record-the-propagation-rule)).
- **Design dataspaces around access and lifecycle**, for example one per
  project and scenario, with approved versions locked.

## 8. Limits in ADME

- **Generally available** since February 2026, on all instances by default,
  in all ADME regions except Indonesia Central
  ([06 §3](../fundamentals/06-ddms-and-extensions.md#3-the-four-ddms-in-adme)).
- **Bound to a data partition:** every call names the partition.
- **Not in ADME at M26:** uploading EPC packages through REST, and registering
  objects in the catalogue automatically. Newer OSDU releases add both; ADME's
  M26 API definition has neither, so manifests are built on request (§1).

## Common misconceptions

- *"A reservoir model is one big file."* It's a graph of objects and arrays.
  The DDMS keeps the links, so users can read one property or follow one fault.
- *"Loading a model makes it searchable."* Only an ingested manifest makes its
  objects findable.
- *"Cloning a dataspace is free."* A clone duplicates the dataspace's content.
- *"Grid properties belong in Delta tables, like everything else."* Selected
  properties can be exported as tables for reporting. The model itself needs
  its grid and links.
- *"The DDMS checks that a model is right."* It checks structure and links, not
  geology.

## Go deeper

- [Reservoir DDMS REST tutorial](https://learn.microsoft.com/azure/energy-data-services/tutorial-reservoir-ddms-apis), [WebSocket (ETP) tutorial](https://learn.microsoft.com/azure/energy-data-services/tutorial-reservoir-ddms-websocket) and [Reservoir DDMS availability](https://learn.microsoft.com/azure/energy-data-services/how-to-enable-reservoir-ddms) (Microsoft Learn)
- [Reservoir DDMS](https://community.opengroup.org/osdu/platform/domain-data-mgmt-services/reservoir/home) and [OpenETPServer](https://community.opengroup.org/osdu/platform/domain-data-mgmt-services/reservoir/open-etp-server) (OSDU community)
- In this guide: manifest ingestion in [DI-04 §4](../data-integration/04-building-and-loading-records.md#4-route-b-manifest-ingestion), and the three copies in [DDMS-01](01-choosing-file-service-ddms-or-lakehouse.md)
