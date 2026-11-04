# DDMS-06 · Petrel Data Services

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). What Petrel Data
> Services does, what users can do with it, how data gets in and out, and when
> it's the right choice. It's an SLB service, not part of OSDU.

## TL;DR

- **Petrel** is SLB's desktop application for seismic interpretation,
  geological modelling and well planning, and one of the most widely used tools
  in the industry. Its work lives in **Petrel projects**, traditionally files on
  shared drives.
- **Petrel Data Services** lets Petrel use **ADME as its data store**: projects
  are saved in ADME, with versions, and opened from there.
- **Liberation** copies data out of saved projects, for example well logs, into
  the **OSDU DDMS**, so that applications other than Petrel can use it.
- It's an **SLB service, not OSDU**. Microsoft says it needs no additional
  licence to get started; Petrel itself is licensed by SLB.
- Where Petrel is the main application, Petrel Data Services can be **the main
  way subsurface work reaches ADME**.

## 1. What it is and what's inside

**Petrel and its projects.** Geoscientists use Petrel to interpret seismic,
build geological and reservoir models, and plan wells. Everything they load and
create, such as seismic, well data, horizons, faults, models and their
settings, is kept in a **Petrel project**.

Traditionally, projects are files on shared drives. That causes familiar
problems:
- nobody knows **which project** holds the latest interpretation;
- projects are **copied** for each study, and the copies drift apart;
- the data inside a project is **invisible** to anyone without Petrel;
- access and history depend on file-share permissions.

**Petrel Data Services** moves the projects into ADME. Microsoft describes it as
letting you "use Petrel from SLB with Azure Data Manager for Energy as its data
store": view your Petrel projects, liberate data from Petrel, and collaborate
with data permanently stored in ADME.

As in the other chapters, there are two parts:
- **Records:** a **project record** for each project, and a **version record**
  for each saved version.
- **Bulk data:** the **project's files**, stored in ADME's managed storage and
  transferred through short-lived SAS URLs.

**How it's organised:**

```text
Petrel Data Services
└── project: "North Sea 2026 study"          ← project record
    ├── version 1, version 2, …              ← version records + project files
    │
    └── liberation, on save ─────────────▶ OSDU DDMS, with lineage to the project
                                            ├── Wellbore DDMS: well logs
                                            └── Seismic DDMS: seismic
```

### Where the data lives, and how many copies

**Petrel Data Services runs in ADME,** in your partition, with ADME's sign-in and
entitlement groups. Its project files sit in ADME's managed storage.

| Copy | Where | Why |
|---|---|---|
| **The Petrel project**, with its versions | Petrel Data Services | Petrel's working copy: everything the geoscientist needs, in Petrel's own format |
| **Liberated data** | Wellbore and Seismic DDMS, as ordinary OSDU records and bulk data | So that other applications, analytics and search can use it |

Liberated data is a **second copy by design**: the project stays Petrel's
working copy, and the liberated copy is what the rest of the platform uses.

### How it relates to ADME's catalogue and the OSDU DDMS

| | OSDU DDMS (Wellbore, Seismic, Reservoir, RAFS) | Petrel Data Services |
|---|---|---|
| **Defined by** | The OSDU community | SLB |
| **Stores** | **Data**, for any application | **Projects**, for one application |
| **Format** | Open: OSDU schemas, RESQML, OpenVDS… | Petrel's project format |
| **Who can read the content** | Any application using the DDMS API | Petrel, until the data is liberated |
| **Reaches the OSDU catalogue** | Through records (each chapter explains how) | Through **liberation** into the OSDU DDMS |

So the two work together: Petrel Data Services is where Petrel **works**, and
liberation makes the results **available** to everyone else.

### What users actually read

| Who | What | Through |
|---|---|---|
| **Petrel users** | Whole projects: open, work, save a new version | Petrel and Project Explorer |
| **Users of other applications** | Liberated well logs and seismic | The [Wellbore](02-wellbore.md) and [Seismic](03-seismic.md) DDMS |
| **Data managers** | Every project, its versions and its owner | Project Explorer, the project records, audit logs |

### What isn't in Petrel Data Services

| Data | Where it lives instead |
|---|---|
| Data for applications other than Petrel | The OSDU DDMS, after liberation, or loaded there directly |
| Searchable well logs and seismic | The Wellbore and Seismic DDMS and their catalogue records |
| Lab results, documents | [RAFS](05-rock-and-fluid-samples.md), the File service |

## 2. What users and applications can do

| Operation | Endpoint (under `/api/petreldms/v1`) or tool | Example |
|---|---|---|
| Open and save projects | **Project Explorer**, from Petrel | A geoscientist's daily work |
| Get a project record | `GET …/projects/{projectId}` | Owner, name, latest version |
| Get a version record | `GET …/projects/{projectId}/versions/{versionId}` | What changed and when |
| Update project properties | `PATCH …/projects/{projectId}` | Rename, change metadata |
| Upload a project | `POST …/projects/{projectId}/upload`, then `PUT` files to the SAS URL | Loading an existing project from a file share |
| Download a project | `GET …/projects/{projectId}/download` | A SAS URL to the project's files |
| Update a project's content | `PUT …/projects/{projectId}/update` | Update the project with data uploaded to its SAS URL |
| Delete a project | `POST …/projects/{projectId}` | Remove a project and its record |
| Search projects | Full text, ranges, geospatial | "Projects covering this area" |

- **Microsoft recommends Project Explorer** for uploading projects. The API is
  for automation and administration.
- **SAS URLs are short-lived secrets.** Keep them in memory, never in logs.
- **Check:** project search is described in Microsoft's tutorial but isn't in
  the M26 API definition. Confirm it on your instance.

## 3. How data gets in

```text
Petrel ──Project Explorer──▶ Petrel Data Services (new project version)
                                 └──liberation──▶ Wellbore DDMS, Seismic DDMS (+ lineage)
Existing project files ──API + SAS upload──▶ Petrel Data Services
```

| Way in | What it does | Use when |
|---|---|---|
| **Saving from Petrel** through Project Explorer | Stores the project as a new version in ADME | Day-to-day work: the recommended way |
| **API upload** with a SAS URL | Uploads existing project files, then registers them | Migrating projects from file shares in bulk |
| **Liberation** | Extracts data from saved projects into the OSDU DDMS, linked back to the project | Making Petrel's data usable outside Petrel |

**Liberation is triggered by saving.** Microsoft's example: a project with many
well logs is saved, the wellbore liberation runs, and the logs appear in the
Wellbore DDMS, with their link to the project tracked through lineage.
Microsoft describes round-trip liberation for **seismic, wellbore and project
data**.

## 4. When new or changed data arrives

| Situation | What happens in ADME | Why |
|---|---|---|
| **A geoscientist saves** | A new project version; earlier versions stay | Full history of the work |
| **New logs or seismic in a saved project** | Liberation copies them into the OSDU DDMS | Other applications can use them |
| **A project is migrated** from a file share | Upload through the API, then register | One governed copy instead of scattered files |
| **A project is deleted** | The project and its record are removed | **Check:** what happens to data already liberated |

**Unlike the OSDU DDMS, some of this is automatic:** saving creates a version,
and liberation runs on save. Migrations, deletions and access stay manual.

## 5. Typical needs it answers, and what to point users to

| The need | Point to |
|---|---|
| Petrel projects scattered over file shares, with copies drifting apart | Petrel Data Services, with projects saved from Petrel |
| "Which project holds the latest interpretation of this field?" | Project Explorer and the project records |
| Applications other than Petrel need Petrel's logs and seismic | Liberation into the Wellbore and Seismic DDMS |
| A history of who changed what in a project | Project versions, and ADME audit logs |
| Analytics on interpretation results | The liberated data, through the OSDU DDMS and their lakehouse options |

## 6. Alternatives, and when each wins

| Option | Wins when | Falls short when |
|---|---|---|
| **Projects on file shares** | A small team, few projects, no need to share data outside Petrel | Finding, versioning, governing or sharing projects |
| **Petrel Data Services** | Petrel is the main application, and projects must be governed and shared | Organisations with little or no Petrel use |
| **Petrel exchanging specific data with the platform** | Moving specific data in or out, for example ZGY seismic, as in SLB's guides linked from Microsoft's tutorial | Storing the whole project: its interpretations, models and settings |
| **Lakehouse** | Analytics on the **liberated** data, through the other DDMS | Whole Petrel projects: they're in Petrel's format, not tables |

## 7. Governance and access

- **API callers need `users.datalake.admins`,** according to Microsoft's
  tutorial. It's a powerful group, so give it only to the identities that
  administer projects, not to every user.
- **Audit logs** cover create, read, update and delete events of the Petrel Data
  Services APIs, as for the Seismic and Wellbore DDMS.
- **Liberated data needs the right ACLs and legal tags.** It becomes ordinary
  OSDU records and bulk data, so the rules in
  [DG-02](../data-governance/02-access-design.md) and
  [DG-03](../data-governance/03-legal-tags.md) apply to it.
- **Check:** how access to individual projects is controlled on your instance,
  and how it maps to ADME's entitlement groups.

## 8. Limits in ADME

- **An SLB service, not OSDU.** SLB defines its features and roadmap; check them
  with SLB and Microsoft.
- **Licensing:** Microsoft says Petrel Data Services doesn't require additional
  licensing to get started. Petrel itself is licensed by SLB; check with SLB
  what else your users need, for example access to Project Explorer.
- **API v1** at M26 (`/api/petreldms/v1`).
- **Check:** that Petrel Data Services is enabled on your instance, and which
  liberation workflows run for your data.

## Common misconceptions

- *"Petrel Data Services makes Petrel's data available to every application."*
  Only **liberated** data is. The project itself is in Petrel's format.
- *"Petrel Data Services is an OSDU DDMS."* It's an SLB service that runs on
  ADME, next to the OSDU DDMS.
- *"Liberation replaces the project."* The project stays Petrel's working copy;
  the liberated data is a second copy for everyone else.
- *"Without Petrel Data Services, Petrel can't use ADME."* Petrel can also
  exchange specific data with the platform, for example ZGY seismic.
  Petrel Data Services adds storage of whole projects.
- *"Every user needs admin rights to use it."* The admins group is for API
  callers. Keep it small.

## Go deeper

- [Petrel Data Services tutorial](https://learn.microsoft.com/azure/energy-data-services/tutorial-petrel-ddms), [DDMS overview](https://learn.microsoft.com/azure/energy-data-services/overview-ddms) and [release notes](https://learn.microsoft.com/azure/energy-data-services/release-notes) (Microsoft Learn)
- [Petrel Data Services API definition, M26](https://github.com/microsoft/adme-samples/blob/main/rest-apis/M26/petrel_ddms_openapi.yaml) (Microsoft)
- SLB guides, linked from Microsoft's tutorial (sign-in may be required): [get started with Project Explorer](https://guru.delfi.slb.com/content/1015/help/1/en-US/299EC605-5CC6-4CD9-9B07-482B16426769), [save a Petrel project in Project Explorer](https://guru.delfi.slb.com/content/1015/help/1/en-US/C86C74BE-6FF9-4962-AEBF-887897B95D4B), [upload ZGY data from Petrel](https://guru.delfi.slb.com/content/1015/help/1/en-US/9F76FDB2-7817-491A-995F-A18D558A191C)
- In this guide: [DDMS-01](01-choosing-file-service-ddms-or-lakehouse.md) for the three copies, and the [Wellbore](02-wellbore.md) and [Seismic](03-seismic.md) DDMS for liberated data
