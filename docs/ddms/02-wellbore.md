# DDMS-02 · Wellbore DDMS

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). What the Wellbore
> DDMS holds, what users can do with it, how data gets in, and when it's the
> right choice. The loading steps themselves are in
> [DI-05 §4](../data-integration/05-files-and-bulk-data.md#4-well-logs-and-trajectories-the-wellbore-ddms).

## TL;DR

- The Wellbore DDMS stores what is **measured or predicted along a wellbore**:
  well log curves, trajectories, pressure predictions and pressure tests.
- Its value is **partial, governed reads**: "GR and RHOB from 2,000 to 2,500 m"
  returns just that, as JSON or Parquet, and only to users who may read the
  log's record.
- **ADME doesn't fill it by itself.** Uploading a LAS or DLIS file stores the
  file, nothing more. A pipeline, a tool such as `wbdutil`, or Petrel Data
  Services must parse the file and write the curves.
- **New content means a new WellLog:** a new logging run or a reprocessed log
  gets its own dataset and WellLog record. Re-writing the curves of an existing
  WellLog is for fixing a load, and it creates a new version.
- It's the **only DDMS whose bulk data ACZ can export** (in preview), so it's the
  easiest to combine with a lakehouse: the DDMS for applications, the lake for
  analytics across many wells.

## 1. What it is and what's inside

A wellbore is surveyed and measured from top to bottom, often several times
during its life. The Wellbore DDMS keeps those measurements as **columns of
values against depth** (or time), next to the records that describe them. For
the domain background, see
[02 §6](../fundamentals/02-energy-domain-primer.md#6-measuring-along-the-wellbore).

The DDMS keeps two parts for each object: the **record**, a small, searchable
description, and, for some types, the **bulk data**: the measured values
themselves, often thousands to millions of numbers, held in the DDMS's own store
and read through its data endpoints.

**The Wellbore DDMS manages both parts.** Its typed endpoints create and update
the records in Storage, and writing curves updates the record with a link to
them. So records and curves stay consistent, and one ACL, the record's,
protects both. Not every DDMS works this way: the Seismic DDMS stores only the
bulk data, and you manage its records yourself
([DDMS-03 §1](03-seismic.md#how-it-relates-to-admes-catalogue)).

**Data types** (from the OSDU Wellbore DDMS code at M26):

| Type | What it is | Values stored separately in the DDMS? |
|---|---|---|
| **WellLog** | Curves measured or computed along the wellbore, for example gamma ray or resistivity | **Yes** |
| **WellboreTrajectory** | The wellbore's path: survey stations with depth, inclination and azimuth | **Yes** |
| **PPFGDataset** | Pore pressure and fracture gradient: predicted before drilling and observed after, with uncertainty. A key input for well design and drilling safety | **Yes** |
| **WellPressureTestRawMeasurement** | Raw readings from a well pressure test, before any processing | **Yes** |
| **WellboreMarkerSet** | Formation tops ("picks"): depths where a rock layer starts | No: record only |
| **WellboreIntervalSet** | Geological or fluid intervals along the wellbore, often based on markers | No: record only |
| **WellLogAcquisition** | The logging job: who logged, when, with what | No: record only |
| **Well, Wellbore** | The parent objects | No: record only |

"Record only" means the DDMS offers typed create, read and version endpoints
for these records, but they have no bulk data. Their content fits in the
record itself.

### What users actually read: curves

| Curve (typical mnemonic) | Unit | What it tells a geoscientist |
|---|---|---|
| Gamma ray (`GR`) | gAPI | Rock type: shale reads high, clean sand low |
| Resistivity, deep and medium (`RDEP`, `RMED`, `ILD`, `LLD`…) | ohm·m | Fluids: hydrocarbons resist current, salt water conducts |
| Bulk density (`RHOB`) | g/cm³ | Porosity and rock type |
| Neutron porosity (`NPHI`) | v/v | Porosity; with density, a gas indicator |
| Sonic (`DT`) | µs/ft | Porosity, rock strength, and the tie between wells and seismic |
| Caliper (`CALI`) | in | Borehole size; enlarged holes make other readings less reliable |
| Computed curves (`PHIE`, `SW`, `VSH`…) | v/v | Interpretation results: effective porosity, water saturation, shale volume |

**v/v** (volume per volume) is a fraction from 0 to 1: `0.25` means 25%. Files
may write it as `frac`, `dec`, `%` or `p.u.` (1 p.u. = 1%), so map units
explicitly when loading.

**All curves in this table are stored in WellLog records.** One WellLog groups
the curves recorded or computed together, usually one logging run or one LAS
file. Its bulk data is a table: the depth column plus one column per curve. A
wellbore typically has many WellLogs: several runs, tools and vendors, and
interpretations.

```text
Wellbore
├── WellLog: "Run 1, wireline" (typically one LAS file)
│     ├── record: Curves[] = DEPT, GR, RHOB, NPHI, RDEP… with units and families
│     └── bulk data: a table, depth + one column per curve
│           DEPT     GR     RHOB   NPHI   RDEP
│           1200.0   45.2   2.41   0.21   12.5
│           …
├── WellLog: "Run 2, logging while drilling"   ← another run, its own curves
├── WellLog: "Petrophysical interpretation"     ← computed curves: PHIE, SW, VSH
├── WellboreTrajectory                ← MD, inclination, azimuth: not a WellLog
├── PPFGDataset                       ← pressures and gradients vs depth: not a WellLog
└── WellPressureTestRawMeasurement    ← pressure over time: not a WellLog
```

- **Mnemonics vary by vendor** (`GR`, `SGR`, `GRC`…). The record's curve
  families make "every gamma-ray curve" findable across vendors.
- **Computed curves are a separate WellLog** from the measured curves they were
  derived from. Set its lineage (`ancestry.parents`) to point back to them.
- **A DLIS file can hold several logs,** which become several WellLogs.
- **Trajectories, pressure predictions and pressure tests aren't WellLogs.**
  They have their own types and their own data tables:
  - a trajectory is rows of measured depth, inclination and azimuth;
  - a PPFG dataset is pressures and gradients against depth;
  - a pressure test is pressure readings at test depths, often over time.

### What isn't in the Wellbore DDMS

| Data | Where it lives instead |
|---|---|
| Log reports, core photos, scanned paper logs | File service, described by Document records |
| Core and fluid lab analyses | [RAFS DDMS](05-rock-and-fluid-samples.md) |
| Seismic | [Seismic DDMS](03-seismic.md) |
| Real-time drilling streams (WITSML) | Not in ADME; the WITSML parser isn't available ([06 §5](../fundamentals/06-ddms-and-extensions.md#5-not-in-adme-as-of-m26)) |
| Reported production volumes (daily or monthly, per well) | `ProductionValues` records, with the values in an attached file. These are core-platform records, not a DDMS ([04 §4](../fundamentals/04-entities-and-files.md#4-time-series-and-production-data)) |
| High-frequency sensor streams (pressure, temperature) | Not in ADME, because the Production DDMS isn't available. Usually a time-series store such as Fabric Real-Time Intelligence or Azure Data Explorer, linked to ADME by record IDs |
| Marker depths, intervals | In their records; there's no bulk data |

## 2. What users and applications can do

| Operation | Endpoint (under `/api/os-wellbore-ddms/ddms/v3/welllogs/{id}`) | Example |
|---|---|---|
| Read chosen curves | `GET …/data?curves=GR,RHOB` | Only the curves a display needs |
| Filter by depth | `GET …/data?filter=MD:gte:2000&filter=MD:lte:2500` | One interval; the filter uses the index column's name, for example `MD` or `DEPT` |
| Page through rows | `…?offset=…&limit=…` | Large logs in steps |
| Describe without reading | `…?describe=true` | Column names and row count |
| Choose the format | `Accept: application/x-parquet` (the default) or `application/json` | Parquet for analytics code, JSON for simple scripts |
| Per-curve statistics | `GET …/data/statistics` | Summary values per curve, for quality checks |
| Read an earlier version | `GET …/versions/{version}/data` | Compare before and after a correction |

- **Every write creates a new version** of the record, and earlier bulk
  versions stay readable.
- **Curve recognition:** `POST /api/os-wellbore-ddms/log-recognition/family`
  suggests the curve family and unit for a mnemonic, for example `GR` → Gamma
  Ray. Useful when standardising logs from many vendors.
- **The record points to its bulk data.** After a write, the WellLog record
  carries a reference to the stored curves. That's why an update must start
  from the stored record: a record rebuilt from scratch loses the link.
- **Large data** (more than about 10 million values or 3,000 columns) is written
  in chunks through a session; see
  [DI-05 §4](../data-integration/05-files-and-bulk-data.md#4-well-logs-and-trajectories-the-wellbore-ddms).
- Trajectories, PPFG datasets and pressure tests have the same data, session
  and version endpoints under their own paths, for example
  `…/wellboretrajectories/{id}/data`. Statistics are for WellLogs only.

## 3. How data gets into the DDMS

```text
LAS / DLIS file ──▶ File service: the original + a dataset record
              └──▶ parse ──▶ WellLog record ──▶ curves written to the Wellbore DDMS
                    (your pipeline · wbdutil · Petrel Data Services)
```

ADME stores files, but doesn't read them. Something has to parse each file and
write its curves:

| Way in | What it does | Use when |
|---|---|---|
| **Your own pipeline** | Reads LAS with `lasio` or DLIS with `dlisio`, maps curves and units, creates the WellLog, writes the curves as JSON or Parquet | Production loads, DLIS, your own validation and mapping rules |
| **`wbdutil`** | OSDU community command-line tool: loads LAS files into Wellbore and WellLog records with their curves; also downloads logs back to LAS. Runs in Fabric notebooks | Quick loads and trials with LAS. It's a community tool, not supported by Microsoft, and it doesn't store the original file in the File service for you |
| **Petrel Data Services** | Extracts ("liberates") logs from Petrel projects stored in ADME into the Wellbore DDMS, with lineage to the project | Logs that live in Petrel projects ([DDMS-06](06-petrel-data-services.md)) |

**ADME has no parser workflow for LAS or DLIS.** Microsoft's list of services on
ADME at M26 has CSV parsing and SEG-Y conversion, but nothing for logs, so
parsing happens outside ADME.

**Whichever way in, keep the order:** original file and dataset record first,
then the WellLog that lists the dataset, then the curves. Access and legal tags
follow the same rule for all three
([DG-02 §4](../data-governance/02-access-design.md#4-acls-on-every-record-the-propagation-rule)).

## 4. When a new or changed file arrives

| Situation | What to do in ADME | Why |
|---|---|---|
| **A new logging run** in an existing wellbore | **New dataset + new WellLog**, then write its curves. Existing logs stay untouched | It's new content, so it's a new object ([DG-05 §4](../data-governance/05-identity-naming-and-versioning.md#4-re-runs-and-versions)) |
| **A reprocessed or corrected log** from the source | **New dataset + new WellLog**, with `ancestry.parents` pointing to the log it replaces | The old content stays available and traceable |
| **A load went wrong** (wrong unit mapping, missing curve) for the same content | Re-write the curves of the **same WellLog** | The fix becomes a new version; the faulty one stays readable |
| **The same file again** | Nothing. Your checksum → dataset ID mapping detects it | Avoids duplicates on re-runs |

**Nothing happens automatically.** ADME doesn't watch the File service. Start
the parse-and-load pipeline from:
- the **source delivery**, for example a new file in a landing folder or a
  message from the source system; or
- an **ADME notification** that a new dataset record was created, if files are
  uploaded by others ([DI-07 §5](../data-integration/07-reading-data-apis-and-change-feeds.md#5-change-feeds-notifications)).

## 5. Typical needs it answers, and what to point users to

| The need | Point to |
|---|---|
| Logs are LAS or DLIS files on shared drives, and every application re-imports them | Wellbore DDMS, with the originals in the File service |
| Petrophysicists need a few curves from many wells | Curve and depth selection through the DDMS API |
| Well planners need offset wells' logs and trajectories | Spatial search for nearby wells, then trajectories and logs from the DDMS |
| A standard set of curve names across vendors | Curve families on the records; log recognition to suggest them |
| Data-quality checks or ML across all wells | A lakehouse copy, through ACZ or your own export (§6) |
| Logs that sit inside Petrel projects | Petrel Data Services liberation into the DDMS |

## 6. Alternatives, and when each wins

| Option | Wins when | Falls short when |
|---|---|---|
| **File service only** | Logs are kept as evidence, rarely read, or always read whole | Applications need parts of logs, or many applications read them |
| **Wellbore DDMS** | Applications and users read selected curves and intervals, often | Scanning thousands of logs for statistics or training |
| **Lakehouse copy** | Statistics, quality KPIs, ML and joins across many wells | Interactive use by domain applications; enforcing ADME's access rules |
| **All three** | The usual production setup ([DDMS-01 §4](01-choosing-file-service-ddms-or-lakehouse.md#4-the-usual-pattern-original-ddms-and-analytics-copy)) | — |

**The lakehouse copy is easiest here.** ACZ's Wellbore DDMS export, in preview,
copies each record's bulk files, Parquet and JSON, into your own storage, next to
the catalogue table. They're files per record, not Delta tables
([DI-08 §3](../data-integration/08-export-to-analytics-and-ai.md#3-working-with-acz-tables)).
Without ACZ, your own export reads the curves as Parquet through the API.

## 7. Governance and access

- **Calling the API** needs `users.datalake.viewers` (read) or
  `users.datalake.editors` (write).
- **Reading a log's curves** also needs membership of a data group in the
  **WellLog record's ACL**. The curves have no separate ACL.
- **Keep the scopes aligned:** the LAS file's dataset record, the WellLog and
  its wellbore should carry the same data groups and legal tags. Otherwise
  someone may download the file who can't see the log, or the reverse.

## 8. Limits in ADME

- **v3 API:** the examples here use the `/ddms/v3` endpoints that ADME
  documents.
- **No file parsing** in the DDMS (§3).
- **Data types and endpoints:** the PPFG, pressure-test and curve-recognition
  endpoints used here are in ADME's own M26 API definition for the Wellbore
  DDMS.

## Common misconceptions

- *"Uploading a LAS file makes its curves available."* It stores the file. The
  curves reach the DDMS only when something parses the file and writes them.
- *"The DDMS replaces the LAS file."* Keep the original; the DDMS holds a
  restructured copy.
- *"A new run of the same log type updates the existing WellLog."* A new run is
  new content: a new dataset and a new WellLog.
- *"Curves have their own permissions."* They follow the WellLog record's ACL.
- *"Reading every curve is always fine."* Very wide or long logs hit the read
  limits. Use `describe`, then read in batches of curves.
- *"Markers live in the DDMS like curves."* Markers and intervals are records,
  with no bulk data.

## Go deeper

- [Wellbore DDMS tutorial](https://learn.microsoft.com/azure/energy-data-services/tutorial-wellbore-ddms) and [DDMS overview](https://learn.microsoft.com/azure/energy-data-services/overview-ddms) (Microsoft Learn)
- [ADME REST API reference](https://microsoft.github.io/adme-samples/rest-apis/index.html) (Microsoft)
- [Wellbore DDMS project](https://community.opengroup.org/osdu/platform/domain-data-mgmt-services/wellbore/wellbore-domain-services) and [`wbdutil` data loader](https://community.opengroup.org/osdu/ui/data-loading/wellbore-ddms-data-loader) (OSDU community)
- In this guide: loading steps in [DI-05 §4](../data-integration/05-files-and-bulk-data.md#4-well-logs-and-trajectories-the-wellbore-ddms), versions and new objects in [DG-05](../data-governance/05-identity-naming-and-versioning.md), and the lakehouse copy in [DI-08](../data-integration/08-export-to-analytics-and-ai.md)
