# DDMS-05 · Rock and Fluid Samples (RAFS) DDMS

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). What the Rock and
> Fluid Samples DDMS holds, what users can do with it, how data gets in, and
> when it's the right choice.

## TL;DR

- RAFS stores **physical samples** taken from wells (rock cores, fluid samples)
  and the **laboratory results** measured on them, as structured data instead
  of numbers locked in PDF and Excel reports.
- More than **40 analysis types** are supported (44 in the M26 code). Results
  are **validated** against a content schema per type, and can be **filtered,
  aggregated and searched across all samples**, for example "every
  permeability measurement above 100 mD".
- It also stores **models derived from lab data** as inputs for reservoir
  simulation: fluid models, saturation functions and rock physics models.
- **Like the Wellbore DDMS, it manages its records itself:** its endpoints
  create and version the records, and the record's ACL protects the results.
- **You must parse lab reports first.** ADME doesn't extract values from PDFs;
  RAFS accepts results already in its content-schema format.
- **Not in ACZ**, but the results are **already tables**, so a lakehouse copy
  works well here.

## 1. What it is and what's inside

Logs and seismic measure rock **indirectly**. Samples are the rock and fluid
themselves, brought to the surface:
- **cores and core plugs:** cylinders of rock cut from the wellbore, and small
  plugs drilled from them for testing;
- **cuttings:** rock chips carried up while drilling;
- **fluid samples:** oil, gas or water captured downhole or at the surface.

Laboratories then measure them: how porous the rock is, how easily fluids flow
through it, how the oil behaves under pressure. These results are **ground
truth** that logs and models are calibrated against, and they're expensive:
a core can't be taken again once the well is finished. For the background, see
[02 §6](../fundamentals/02-energy-domain-primer.md#6-measuring-along-the-wellbore).

As in every DDMS, there are two parts:
- **Records** describe the samples, the lab reports, each analysis and the
  models derived from them.
- **Bulk data** is the table of results of each analysis, often many rows
  (one per plug, per pressure step or per test condition), stored as Parquet.

**Data types:**

| Type | What it is | Values stored separately in the DDMS? |
|---|---|---|
| **Sample** (master data) | One physical sample: a core plug, a fluid sample | No: record only |
| **SampleContainer** (master data) | What holds the sample: a core box, a bottle | No: record only |
| **SampleAcquisitionJob** (master data) | The operation that took the samples, for example a coring run | No: record only |
| **SampleChainOfCustodyEvent** (master data) | Where the sample went and who had it: shipped, sub-sampled, stored | No: record only |
| **GenericSite**, **GenericFacility** (master data) | Places and facilities the samples relate to | No: record only |
| **SamplesAnalysesReport** (WPC) | One lab report, linked to its original file | No: it describes the report and points to the file |
| **SamplesAnalysis** (WPC) | One analysis of one type on one or more samples | **Yes:** the table of results |
| **FluidModel** (WPC) | A model of how the reservoir fluid behaves, often derived from fluid sample analyses: black-oil tables, equations of state or correlations | **Yes:** its tables |
| **SaturationFunctionSet** (WPC) | How relative permeability and capillary pressure vary with fluid saturation | **Yes:** its tables |
| **ReservoirSimulationRockPhysicsModel** (WPC) | Rock compressibility, thermal and other rock properties a simulator needs | **Yes:** its tables |

The last three are **inputs for reservoir simulation**, usually built from the
analysis results above.

**The chain from well to number:**

```text
Well ─▶ Wellbore                              (Storage records, created first)
          └─ Sample: core plug at 2,431 m      ─ SampleContainer, chain-of-custody events
               └─ SamplesAnalysesReport: "Lab X, routine core analysis, 2026" ─▶ original PDF (File service)
                    └─ SamplesAnalysis: routine core analysis ─▶ table of results (Parquet)
                          plug   depth (m)   porosity   permeability (mD)   grain density (g/cm³)
                          P-01   2431.2      0.21       145                 2.65
                          P-02   2431.5      0.18        62                 2.66
```

### Analysis types

The M26 code has a content schema for 44 analysis types. Grouped:

| Family | Examples | What it tells you |
|---|---|---|
| **Routine core analysis** | Porosity, permeability, grain density; core gamma; crushed rock analysis | How much fluid the rock holds, and how easily it flows |
| **Special core analysis** | Relative permeability, capillary pressure, wettability, electrical properties, NMR, rock compressibility | How oil, gas and water flow together; calibration of logs |
| **Fluid properties (PVT)** | Constant composition expansion, constant volume depletion, differential liberation, multi-stage separator, swelling, slim tube, vapour–liquid equilibrium | How the oil and gas behave as pressure and temperature change: the inputs for simulation |
| **Fluid composition** | Atmospheric flash, gas composition, stock-tank oil, water analysis | What the fluids are made of |
| **Geochemistry** | Gas chromatography, GC-MS, isotopes, bulk pyrolysis, vitrinite reflectance | Where the hydrocarbons came from, and how mature the source rock is |
| **Mineralogy** | XRD, XRF, cation exchange capacity | Which minerals the rock contains |
| **Geomechanics** | Triaxial, uniaxial, tensile strength | How strong the rock is, for drilling and production |

`GET …/samplesanalysis/analysistypes` lists the types your instance supports,
and `…/{type}/data/schema` returns the content schema of each.

### What users actually read

| What | Example | Who uses it |
|---|---|---|
| **Results per plug** | Porosity and permeability at each depth | Petrophysicists, calibrating logs |
| **Results under conditions** | Relative permeability at each saturation step | Reservoir engineers, for simulation |
| **Fluid behaviour** | Bubble point, oil volume factor from PVT tests | Reservoir and production engineers |
| **Aggregates** | Average porosity of all plugs in a zone | Studies and reviews |
| **Results across samples** | Every permeability measurement above 100 mD | Analogue studies, ML |
| **The original report** | The lab's PDF | Anyone who needs to check the source |

### Where the data lives, and how many copies

**RAFS is part of ADME.** It uses the same partition, sign-in, entitlement
groups and legal tags, and its storage is managed by Microsoft.

| Copy | Where | Why |
|---|---|---|
| **The original report** (PDF, spreadsheet) | File service, linked from the SamplesAnalysesReport | Evidence, and the source for checking values |
| **The parsed results** | RAFS, as Parquet behind each SamplesAnalysis | Querying, filtering, comparing across samples |

### How it relates to ADME's catalogue

**Like the Wellbore DDMS, RAFS manages the records itself.** Its endpoints
create and version the sample, report and analysis records in Storage, and
writing results updates the analysis record, as a new version. Records and
results stay in step, and one ACL protects both.

**On top, RAFS has its own search over results.** Search finds the records;
RAFS's search endpoints also look **inside** the result tables of every
analysis of one type.

| | Wellbore DDMS | Seismic DDMS | Reservoir DDMS | RAFS DDMS |
|---|---|---|---|---|
| **Catalogue records** | Created by the DDMS | Created by you; conversion workflows add those of converted copies | Built by the DDMS on request, ingested by you | **Created by the DDMS** |
| **Access unit** | The record's ACL | Subproject or dataset | Dataspace | **The record's ACL** |
| **Stays in step with the catalogue** | Automatically | Only through your pipeline | When you re-ingest the manifest | **Automatically** |
| **Search inside the bulk data** | No | No | No | **Yes, per analysis type** |

### What isn't in RAFS

| Data | Where it lives instead |
|---|---|
| Well logs, including core gamma recorded as a log | [Wellbore DDMS](02-wellbore.md) |
| Core photos, CT scans, thin-section images | File service, described by Document or other records |
| Raw instrument files from the lab | File service, if they're kept at all |
| Properties in a reservoir model | [Reservoir DDMS](04-reservoir.md) |

## 2. What users and applications can do

| Operation | Endpoint (under `/api/rafs-ddms/v2`) | Example |
|---|---|---|
| Create and update records | `POST …/masterdata`, `…/samplesanalysesreport`, `…/samplesanalysis` | A sample, its report, its analysis |
| Read records and versions | `GET …/{type}/{id}`, `…/{id}/versions`, `…/versions/{version}` | A report as it was before a correction |
| List analysis types and schemas | `GET …/samplesanalysis/analysistypes`, `…/{type}/data/schema` | Which columns a PVT test needs |
| **Write results** | `POST …/samplesanalysis/{id}/data/{type}` (JSON or Parquet) | The table of one analysis |
| **Read results** | `GET …/samplesanalysis/{id}/data/{type}/{content_id}` | One analysis's table |
| **Filter and aggregate** | `…?columns_filter=…&rows_filter=…&columns_aggregation=…` | Only porosity and permeability; only rows above 10 °C; the mean |
| **Search results across samples** | `GET …/samplesanalysis/{type}/search/data` | Every relative-permeability table, filtered by value |
| Get the original report | `GET …/samplesanalysesreport/{id}/source` | The report's source file, from its `Datasets`, for example the lab's PDF |
| Simulation-input models | `…/fluidmodel`, `…/saturationfunctionset`, `…/reservoirsimulationrockphysicsmodel`: records, data, content schema and search | A fluid model for a simulation study |
| Soft delete | `DELETE …/{type}/{id}` | A record loaded by mistake |

**Filter syntax:**
- `columns_filter=SampleID,Porosity`: return only these columns.
- `rows_filter=TestTemperature.Value,gt,10`: one condition per request,
  written as column, operator, value.
- `columns_aggregation=TestTemperature.Value,mean`: aggregate a column, for
  example `mean` or `count`.

Writing results **creates a new version** of the analysis record, so earlier
results stay readable.

## 3. How data gets into the DDMS

```text
Lab report (PDF, spreadsheet) ──▶ File service (original + dataset record)
             └──▶ parse into the content-schema format
                    ──▶ Sample records ──▶ SamplesAnalysesReport ──▶ SamplesAnalysis ──▶ results
```

**RAFS doesn't read lab reports.** Report formats differ by laboratory, so
each pipeline turns them into the content schema of the analysis type itself.
RAFS then **validates** the table against that schema, and rejects data that
doesn't fit.

| Way in | What it does | Use when |
|---|---|---|
| **Your own parser** | Reads the lab's spreadsheet or report, maps columns and units to the content schema, writes records and results | Most cases: the usual path |
| **Structured delivery from the lab** | The lab delivers results directly in the content-schema format, or close to it | When you can agree a format with your laboratories; the cheapest path long-term |
| **Manual entry** | A data manager keys in values from an old PDF | Small volumes of legacy reports |

**Check:** whether any extraction tooling fits your report formats. Values
read from PDFs by OCR or AI tools still need review before loading.

**Whichever way in, keep the order:**
1. The master hierarchy through Storage: Organisation → Field → Well →
   Wellbore.
2. The sample master data through RAFS: sites, acquisition jobs, containers,
   samples.
3. The SamplesAnalysesReport, pointing to the original file.
4. The SamplesAnalysis records, then their results.

## 4. When new or changed data arrives

| Situation | What to do in ADME | Why |
|---|---|---|
| **A new lab report** | Original file, new report record, new analysis records and their results | New content |
| **Corrected values** from the lab | Re-write the results of the **same analysis** | A new version; the old values stay readable |
| **A re-analysis** of the same sample, for example by another lab | A **new analysis** (and report) | Both results stay, with their source |
| **A sample moved, sub-sampled or consumed** | A chain-of-custody event | The sample's history stays traceable |
| **The same report again** | Nothing. Your checksum → dataset ID mapping detects it | Avoids duplicates |

**Nothing happens automatically.** RAFS doesn't watch the File service or parse
new reports. Your pipeline, or the data manager, starts each load.

## 5. Typical needs it answers, and what to point users to

| The need | Point to |
|---|---|
| Lab results are locked in PDF reports | RAFS, after parsing the reports; the PDFs stay in the File service |
| Petrophysicists calibrate logs with core data | Results per plug, by depth, read with filters |
| Reservoir engineers need PVT and relative permeability for simulation | The analysis results, or the fluid models and saturation function sets built from them |
| Comparing results across wells or fields | Search across samples, per analysis type |
| "Where is this core plug, and what's been done to it?" | Sample and chain-of-custody records |
| Dashboards or ML across all lab data | A lakehouse copy (§6) |

## 6. Alternatives, and when each wins

| Option | Wins when | Falls short when |
|---|---|---|
| **File service: reports only** | Reports kept as evidence, rarely consulted | Anyone needs the values: each report must be opened and read |
| **RAFS DDMS** | Results are reused: compared, filtered, fed to models | — |
| **Lakehouse copy** | Analytics and ML across all samples, joined with logs and production | As the home of record: no validation, no link to the samples and reports, ADME's access rules don't apply |

**Without RAFS, lab data can still be stored in ADME:** the reports go into the
File service as documents. They're then an **archive**: findable by their
records, but the numbers inside can't be queried.

**The lakehouse fits this data well.** Unlike seismic volumes or reservoir
models, analysis results are **already tables**, so they translate directly
into Delta tables. ACZ doesn't export RAFS data, so build the copy with your
own export, for example by reading each analysis type through the
search-data endpoint, which can also filter by indexing date for incremental
runs
([DI-08 §4](../data-integration/08-export-to-analytics-and-ai.md#4-your-own-export-when-acz-isnt-available)).
Keep RAFS as the home of record: it validates the data and links each value to
its sample, report and well.

## 7. Governance and access

- **Calling the API** needs `users.datalake.viewers` (read) or
  `users.datalake.editors` (write).
- **Reading results** also needs membership of a data group in the
  **analysis record's ACL**. The results have no separate ACL.
- **Keep the scopes aligned:** the original report file, the report record, the
  analyses and the samples should carry the same data groups and legal tags.
  Lab data often comes with contracts and partner restrictions, so check the
  legal tag before loading
  ([DG-03](../data-governance/03-legal-tags.md)).

## 8. Limits in ADME

- **Generally available** since January 2026
  ([06 §3](../fundamentals/06-ddms-and-extensions.md#3-the-four-ddms-in-adme)).
- **No report parsing** in the DDMS (§3).
- **Filters take one row condition per request.**
- **Check:** RAFS needs its reference values and schemas in the partition.
  Confirm they're present on your instance, and which content-schema versions
  it supports (`…/analysistypes`, `…/data/schema`).

## Common misconceptions

- *"Uploading a lab report makes its values available."* It stores the file.
  The values reach RAFS only when someone parses them into the content schema.
- *"RAFS replaces the lab report."* Keep the original; RAFS holds the parsed
  results and links back to it.
- *"A corrected value overwrites the old one."* It creates a new version; the
  old values stay readable.
- *"Lab data is too small to need a DDMS."* The value isn't size. It's
  validation, links to samples and wells, and querying across thousands of
  results.
- *"Search finds the numbers."* Search finds the records. RAFS's own search
  endpoints look inside the result tables.

## Go deeper

- [RAFS DDMS tutorial](https://learn.microsoft.com/azure/energy-data-services/tutorial-rock-and-fluid-samples-ddms) and [DDMS overview](https://learn.microsoft.com/azure/energy-data-services/overview-ddms) (Microsoft Learn)
- [RAFS DDMS project](https://community.opengroup.org/osdu/platform/domain-data-mgmt-services/rock-and-fluid-sample/rafs-ddms-services) (OSDU community)
- In this guide: the propagation of access in [DG-02 §4](../data-governance/02-access-design.md#4-acls-on-every-record-the-propagation-rule), and the three copies in [DDMS-01](01-choosing-file-service-ddms-or-lakehouse.md)
