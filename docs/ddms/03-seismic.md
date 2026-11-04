# DDMS-03 · Seismic DDMS

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). What the Seismic
> DDMS holds, what users can do with it, how data gets in, and when it's the
> right choice. The loading steps themselves are in
> [DI-05 §5](../data-integration/05-files-and-bulk-data.md#5-seismic-the-seismic-ddms).

## TL;DR

- Seismic is often the **largest and most expensive** subsurface data a company
  owns: one 3D volume ranges from tens of GB to several TB, surveys cost a lot
  to acquire, and they're reused and reprocessed for decades.
- The Seismic DDMS, also called **Seismic Store**, holds datasets of **any
  size**. Converted to **OpenVDS** or **ZGY**, a volume can be read **in
  parts**: one slice, one section or one region, without downloading the rest.
- **Seismic Store is part of ADME, but not searchable.** Catalogue records
  point to its `sd://` paths: you create them for the original, and the
  conversion workflows add them for converted copies. Converted copies sit next
  to the original SEG-Y, so a volume is typically stored two or three times.
- **Access is set per subproject:** the same for all its datasets, or per
  dataset.
- **Seismic isn't in ACZ, and doesn't belong in Delta tables.** Applications
  and ML read volumes through OpenVDS or ZGY, from the DDMS or from array copies
  in your lake.
- **Storage tiers** (preview) move rarely used surveys to cheaper storage.

## 1. What it is and what's inside

Seismic is an image of the subsurface made with sound: the "ultrasound scan" of
the Earth that shows where the rock layers and traps are. For the background
and the vocabulary, see
[02 §7](../fundamentals/02-energy-domain-primer.md#7-seismic-in-one-page).

As in every DDMS, there are two parts:
- **Records** are the catalogue: the survey, the processing project, the
  processed volume and its grid. They're ordinary Storage records, searchable
  like any other.
- **Bulk data** is the volume itself: the traces, often hundreds of gigabytes,
  kept in Seismic Store under an `sd://` path.

**Data types:**

| Type | What it is | Values stored separately in the DDMS? |
|---|---|---|
| **SeismicAcquisitionSurvey** (master data) | The survey campaign: where, when, how it was recorded | No: record only |
| **SeismicProcessingProject** (master data) | One processing of a survey; the same survey may be reprocessed years later | No: record only |
| **SeismicTraceData** (WPC) | One processed volume or 2D line: its domain (time or depth), its range, its grid | No: it describes the volume, lists the SEG-Y in `Datasets[]` and the converted copies in `Artefacts[]` |
| **SeismicBinGrid** (WPC) | The geometry of a 3D survey's inline and crossline grid | No: record only |
| **FileCollection.SEGY** (dataset) | A SEG-Y file in Seismic Store | **Yes:** the SEG-Y bytes |
| **FileCollection.Bluware.OpenVDS** (dataset) | The same volume converted to OpenVDS | **Yes:** the OpenVDS blocks |
| **FileCollection.Slb.OpenZGY** (dataset) | The same volume converted to ZGY | **Yes:** the ZGY file |

**How it's organised:**

```text
Seismic Store
└── tenant: your data partition, registered once
    └── subproject: a working area with its own admins, viewers and legal tag
        └── datasets, under paths:
            sd://<tenant>/<subproject>/north-sea-3d/2019-reprocessing/
            ├── survey.segy        ← original           ← FileCollection.SEGY record
            ├── survey.vds         ← converted copy     ← FileCollection.Bluware.OpenVDS record
            └── survey.zgy         ← converted copy     ← FileCollection.Slb.OpenZGY record
                                       all three referenced by one SeismicTraceData record
```

### Where the data lives, and how many copies

**Seismic Store is ADME's storage for seismic, not a separate product.** Its
bytes sit in ADME's Microsoft-managed blob storage, like files from the File
service, but in their own area, managed by their own service. A SEG-Y loaded
into Seismic Store isn't also stored in the File service.

| Shared with the rest of ADME | Its own |
|---|---|
| The same instance, billing and Microsoft-managed Azure storage | Its own **API** (`/seistore-svc/api/v3`) and tools (`sdutil`) |
| The same partition, registered as its tenant | Its own **list of datasets**: paths, sizes, locks, tags |
| The same sign-in and tokens | Its own **permission model**: subprojects, uniform or per dataset |
| The same entitlement groups and legal tags | **Not in Search:** only your catalogue records are |

**The copies come from conversion, and all of them live in Seismic Store:**

| Copy | Why it's kept |
|---|---|
| **SEG-Y** (the original) | Source of truth, exchange with partners, re-processing. Once converted, it's rarely read, so it's a good candidate for the Cool or Cold tier |
| **OpenVDS and/or ZGY** | Fast reading in parts, by applications and ML |

So a volume is typically stored **two or three times**. Conversion buys reading
speed, not storage savings.

### How it relates to ADME's catalogue

**The Wellbore DDMS is part of ADME's catalogue. The Seismic DDMS is a separate
store that shares ADME's security.** Seismic Store keeps its own list of
datasets (path, size, lock, tags, permissions). These aren't OSDU records, so
Search doesn't see them until you create records that point to them.

**From a search result to the bytes:**

```text
CATALOGUE (Storage + Search: OSDU records, searchable)
SeismicAcquisitionSurvey ◀── SeismicProcessingProject ◀── SeismicTraceData
                                                              │
                    ┌──────────── Datasets[] ─────────────────┼──── Artefacts[] (ConvertedContent) ────┐
          FileCollection.SEGY            FileCollection.Bluware.OpenVDS        FileCollection.Slb.OpenZGY
          (you create it)                (the conversion workflow creates it and its link)
                    │ sd:// path                              │ sd:// path                   │ sd:// path
────────────────────┼─────────────────────────────────────────┼──────────────────────────────┼─────
SEISMIC STORE (bulk data: its own dataset list, not searchable)
      sd://tenant/subproject/…/survey.segy        …/survey.vds                   …/survey.zgy
```

1. **Search** finds the SeismicTraceData: which survey, which processing, time
   or depth, which grid.
2. Its **`Datasets[]`** lists the original SEG-Y. Its **`Artefacts[]`** lists
   the converted copies, with the role "ConvertedContent".
3. Each **dataset record** points into Seismic Store: `FileCollectionPath`
   holds the `sd://` folder, and each `FileSourceInfos[].FileSource` names a
   file relative to it.
4. The application **reads** from Seismic Store at that path: in parts with
   OpenVDS or ZGY, or whole with `sdutil`.

**Who creates the links:** your pipeline creates the records for the survey,
the processing, the trace data and the original SEG-Y, with the right paths.
The **conversion workflows** then create the record of the converted copy and
add it to a new version of the SeismicTraceData (§3). Seismic Store itself
never writes records.

**Compared with the Wellbore DDMS:**

| | Wellbore DDMS ([DDMS-02](02-wellbore.md)) | Seismic DDMS |
|---|---|---|
| **Who creates the records** | The DDMS, through its typed endpoints | **You**, through the Storage API or manifest ingestion; the conversion workflows add the records of converted copies |
| **Writing data** | Updates the record automatically, as a new version | Doesn't touch the record |
| **Deleting data** | Record and data stay consistent | The record stays and points to nothing, unless you delete it too |
| **Access** | One ACL: the record's | Two: subproject or dataset groups for the bytes, record ACLs for the catalogue. Keep them aligned (§7) |
| **Finding the data** | Search finds the WellLog | Search finds only the records that exist: yours, plus those the conversion workflows add |
| **Shared with ADME** | Partition, sign-in, entitlement groups, legal tags | The same (see [above](#where-the-data-lives-and-how-many-copies)) |

In practice, seismic needs **more discipline from pipelines and data
managers**: every dataset needs matching records, aligned access, and cleanup
on both sides.

### The formats: why conversion matters

| Format | What it is | Best for |
|---|---|---|
| **SEG-Y** | The industry interchange standard. Traces are stored one after another, so reading a time slice means reading every trace in the file | Delivery and archive: the **original** |
| **OpenVDS** | A format that stores the volume in **compressed blocks** ("bricks") for random access. Designed for volumes up to petabytes, from 2D lines to pre-stack data with up to 6 dimensions. Open source, with C++, Python and Java libraries | Applications and ML that read slices, sections or regions |
| **ZGY** | SLB's block-based format for 3D interpretation data, open-sourced as **OpenZGY**, with C++ and Python libraries | **Petrel** and similar interpretation tools |

Block-based formats are what make seismic usable from the cloud: an
application asks for the blocks it needs and starts drawing immediately,
instead of waiting for terabytes to arrive.

### What users actually read

| What | What it is | Who uses it |
|---|---|---|
| **Inline or crossline section** | A vertical slice along one axis of the survey grid | Interpreters, following a layer or a fault |
| **Time or depth slice** | A horizontal cut through the volume at one time or depth | Interpreters, mapping features across the area |
| **Sub-volume** | A region around a target, for example a prospect | Detailed interpretation, ML inference |
| **Attribute volume** | A volume computed from the original, for example amplitude or coherence | Interpreters, highlighting faults or fluids |
| **2D line** | A single line from an older or regional survey | Regional studies |
| **Training patches** | Small cubes cut from volumes, with labels | ML on seismic |

### What isn't in the Seismic DDMS

| Data | Where it lives instead |
|---|---|
| Interpretations: horizons and faults | Their own records (`SeismicHorizon`, `SeismicFault` and the `*Interpretation` types) with files, or as RESQML objects in the [Reservoir DDMS](04-reservoir.md), which can build `HorizonInterpretation` and `FaultInterpretation` catalogue records for them. **Check:** which your applications use |
| Acquisition and processing reports | File service, described by Document records |
| Wells, logs, trajectories | [Wellbore DDMS](02-wellbore.md) |
| Survey metadata and inventories | The catalogue records above; searchable, and exportable to a lake |

## 2. What users and applications can do

| Operation | Endpoint (under `/seistore-svc/api/v3`) or tool | Example |
|---|---|---|
| Upload and download datasets of any size | `sdutil cp`, or `…/utility/upload-connection-string` and `…/download-connection-string` | `sdutil cp survey.segy sd://<tenant>/<subproject>/raw/survey.segy` |
| Read parts of a volume | OpenVDS or OpenZGY libraries, from applications or Python | One inline, one slice, one region |
| List and inspect | `…/utility/ls`, dataset size endpoints | What's in a subproject, and how big |
| Copy inside a subproject | `…/utility/cp` | A working copy, without downloading |
| Lock a dataset | `…/dataset/…/lock` and `…/unlock` | Open a dataset for writing without others changing it at the same time |
| Change storage tier | `…/operation/change-tier` | Move an old survey to Cool or Cold |
| Delete many datasets | `…/operation/bulk-delete` | Clean up a whole path |
| Restore one dataset | `…/operation/restore`: ADME-specific, on request | Recover a dataset deleted or overwritten by mistake |
| Storage reports | `…/analytics/job`: ADME-specific preview, on request | Scheduled reports of object count, size and size per tier, per subproject |

- **Credentials are short-lived:** the upload and download endpoints return an
  Azure SAS URL valid for about **one hour** for the subproject or dataset.
  Request them just before the transfer.
- **Restore:** restores a single dataset's metadata and files to a point in
  time within a fixed **30 days**, one restore per partition at a time. It's a
  preview, on request, on the Standard tier
  ([Microsoft Learn](https://learn.microsoft.com/azure/energy-data-services/tutorial-seismic-restore-dataset)).
- **Seismic File Metadata:** ADME also runs this related service, which
  annotates seismic data with dimensions, value channels and key/value pairs.

## 3. How data gets into the DDMS

```text
SEG-Y ──sdutil or API──▶ Seismic Store (subproject)
   │                         └──conversion workflow──▶ OpenVDS / ZGY dataset
   └──▶ catalogue records (Storage): FileCollection.SEGY, SeismicTraceData, bin grid, work product, survey, project
                                     + FileCollection.OpenVDS / OpenZGY for each conversion (created by the workflow)
```

| Way in | What it does | Use when |
|---|---|---|
| **`sdutil`** | Command-line Python tool: uploads, downloads, lists and manages datasets | Loads of individual surveys; scripts |
| **The API with upload credentials** | Your pipeline gets a SAS URL and writes with any Azure storage tool or SDK | Automated pipelines, large parallel uploads |
| **Conversion workflows** | `segy-to-vds-conversion` and `segy-to-zgy-conversion`, started through the Workflow API. They read your records, write the converted dataset into Seismic Store, create its dataset record and add it to a new version of the SeismicTraceData | After the SEG-Y and its records are loaded, to make it readable in parts |
| **Petrel Data Services** | Microsoft describes round-trip liberation for seismic between Petrel projects and the DDMS | Seismic that lives in Petrel projects ([DDMS-06](06-petrel-data-services.md)) |

**Both conversion workflows are on Microsoft's list of ADME services.** They
need their input records first: the SEG-Y dataset, the SeismicTraceData, its
bin grid and the work product that groups them, as Microsoft's tutorials show.
**Check:** the record handling described here comes from the OSDU code;
confirm on your instance which records the workflows create.

**Whichever way in, keep the order:**
1. Register the partition as a seismic tenant (once), and create the
   subproject.
2. Upload the SEG-Y.
3. Create the catalogue records that point to it: the SEG-Y dataset, the trace
   data, the bin grid and the work product, plus the survey and processing
   project.
4. Convert. The workflow registers the converted dataset next to the original
   and links it from the trace data.

**Copying SEG-Y files to a storage account with AzCopy isn't ingestion.** The
files are neither in Seismic Store nor catalogued, so nobody can find or govern
them.

## 4. When new or changed data arrives

| Situation | What to do in ADME | Why |
|---|---|---|
| **A new survey** | New survey record, new processing project, new SeismicTraceData, new datasets | It's new content |
| **A reprocessing** of an existing survey | New processing project and new SeismicTraceData, with lineage to the earlier processing. The old volume stays | Interpreters must know which version their work is based on |
| **A conversion** to OpenVDS or ZGY | A new dataset **next to** the SEG-Y. The workflow adds it to a new version of the SeismicTraceData, under `Artefacts[]` | Keep the original; the conversion is a copy optimised for reading |
| **A survey no longer in active use** | Move its datasets to Cool or Cold | Lower storage cost, still available |
| **The same file again** | Nothing. Check your dataset paths and checksums before uploading | Avoids paying twice for terabytes |

**Nothing happens automatically.** Seismic Store itself doesn't create records,
start conversions or move tiers. Your pipeline, or the data manager, starts
each step; only a conversion workflow, once started, writes records, for the
converted copy.

## 5. Typical needs it answers, and what to point users to

| The need | Point to |
|---|---|
| Interpreters wait for large surveys to load | Seismic DDMS, with volumes converted to OpenVDS or ZGY |
| Seismic is scattered across file shares, tapes and project folders | Seismic Store with catalogue records, so every survey is findable |
| Several interpretation tools need the same volumes | One copy in the Seismic DDMS instead of one per tool |
| Old surveys are rarely used but expensive to keep | Storage tiers: Cool or Cold (preview; check availability on your tier) |
| ML on seismic: fault or horizon detection | OpenVDS reads from the DDMS, or training patches copied to your lake as arrays (§6) |
| Seismic that lives in Petrel projects | Petrel Data Services liberation into the DDMS |

## 6. Alternatives, and when each wins

| Option | Wins when | Falls short when |
|---|---|---|
| **File service or plain SEG-Y files** | Small files kept as evidence, rarely read | Anything large, or read often or in parts. Files of 5 GB or more need AzCopy, and are still read whole |
| **Seismic DDMS** | The default for seismic that is used: governed, catalogued, readable in parts | Analytics over survey metadata (use the catalogue records) |
| **Lakehouse copy** | Survey inventories and metadata as tables; **training patches** for ML as array files (OpenVDS, Zarr, NumPy) | As Delta tables of traces: a volume is a 3D array, and models read 3D blocks |

**Without the Seismic DDMS, seismic can still be stored in ADME,** through the
File or Dataset service. The SEG-Y files then sit in ADME's managed storage,
registered as dataset records, but only as an **archive**:
- they can only be downloaded **whole**: terabytes to see one slice;
- interpretation applications that read through the Seismic DDMS can't stream
  them;
- there are no conversion workflows, storage tiers or subprojects.

**Why not keep seismic only in the lake?** Volumes can sit in your own ADLS or
OneLake as array files (OpenVDS, Zarr) and be processed with Databricks,
Fabric or Azure ML. What you give up when the lake is the **main** home:

| You lose | Why it matters |
|---|---|
| ADME's governance | Access groups and legal tags don't apply in the lake; you rebuild access control there |
| The catalogue | Seismic isn't findable next to wells, logs and documents, unless you keep both in step |
| Interpretation applications | Tools such as Petrel read ZGY through the Seismic DDMS, not from your lake |
| A managed export | ACZ doesn't export seismic, so you build and maintain the copy yourself |

So: **lake copies for analytics and ML, yes; the lake as the main home for
seismic, usually no.** The Seismic DDMS stays the governed source that
applications use.

**ML on seismic: compute and storage are separate choices.** The compute can
be Databricks, Azure ML or Fabric. The data is read in an array format, either:
- **directly from the Seismic DDMS** with OpenVDS: no extra copy, access
  enforced by ADME, good for inference over whole surveys; or
- **from array copies in your lake**: better for training, which reads the same
  patches many times, but the copy needs its own access control
  ([DI-08 §5](../data-integration/08-export-to-analytics-and-ai.md#5-re-applying-access-in-the-copy)).

Training on copies and running inference from the DDMS is a common
combination. Results such as faults or horizons go back to ADME as new records
([08 §5](../fundamentals/08-analytics-and-ai.md#5-recommendations-what-to-use-for-which-workload)).

## 7. Governance and access

- **Access is set per subproject,** not on each record:
  - each subproject has **admins** (write) and **viewers** (read) groups, and a
    **legal tag**;
  - its access policy is **uniform** (the default: the same for all datasets)
    or **dataset** (access set per dataset).
- **Name your own data groups when you create a subproject.** If you don't,
  Seismic Store creates new groups for it (`data.sdms.<tenant>.<subproject>.<id>.admin`
  and `.viewer`), which then have to be managed and aligned separately.
- **The catalogue records have their own ACLs.** Point them at the **same data
  groups** as the subproject, so that whoever finds a survey in search can also
  read it, and nobody else can
  ([DG-02 §4](../data-governance/02-access-design.md#4-acls-on-every-record-the-propagation-rule)).
- **Design subprojects around access**, for example one per licence or
  confidentiality level, rather than one per survey.
- **SAS URLs are secrets:** keep them in memory, never in logs.

## 8. Limits in ADME

- **Seismic DDMS v3 only;** the v4 APIs aren't available
  ([06 §5](../fundamentals/06-ddms-and-extensions.md#5-not-in-adme-as-of-m26)).
- **Each partition must be registered as a seismic tenant** before use.
- **Storage tiers:** a preview, available by default on the Developer tier.
  Moving data out of a tier too early can incur early-deletion charges.
- **Restore:** a preview, on request, on the Standard tier.
- **Conversions:** both SEG-Y conversion workflows are available (§3).

## Common misconceptions

- *"Converting to OpenVDS saves storage."* The original SEG-Y is kept, so the
  conversion adds a copy. It saves **reading time**, not storage.
- *"Uploading SEG-Y makes it searchable."* Only catalogue records make seismic
  findable. Seismic Store holds the bytes.
- *"Seismic can go into Delta tables like logs."* A volume is a 3D array.
  Analytics and ML read it in blocks.
- *"Seismic Store is a separate system outside ADME."* It's a service inside
  ADME, using the same partition, sign-in, groups and legal tags. Only its
  dataset list and permission model are its own.
- *"Each dataset has a record ACL like everything else."* Seismic Store access
  is set per subproject, or per dataset with the dataset policy. The catalogue
  records' ACLs are separate, so keep them aligned.
- *"A reprocessed survey replaces the old one."* It's new content: a new
  processing project and new trace data. The old version stays for the
  interpretations based on it.

## Go deeper

- [Seismic DDMS tutorial](https://learn.microsoft.com/azure/energy-data-services/tutorial-seismic-ddms) and [sdutil tutorial](https://learn.microsoft.com/azure/energy-data-services/tutorial-seismic-ddms-sdutil) (Microsoft Learn)
- [Convert SEG-Y to OpenVDS](https://learn.microsoft.com/azure/energy-data-services/how-to-convert-segy-to-ovds), [convert SEG-Y to ZGY](https://learn.microsoft.com/azure/energy-data-services/how-to-convert-segy-to-zgy) and [change storage tiers](https://learn.microsoft.com/azure/energy-data-services/tutorial-seismic-change-tier) (Microsoft Learn)
- [OpenVDS](https://community.opengroup.org/osdu/platform/domain-data-mgmt-services/seismic/open-vds) and [OpenZGY](https://community.opengroup.org/osdu/platform/domain-data-mgmt-services/seismic/open-zgy) (OSDU community)
- In this guide: loading steps in [DI-05 §5](../data-integration/05-files-and-bulk-data.md#5-seismic-the-seismic-ddms), and the three copies in [DDMS-01](01-choosing-file-service-ddms-or-lakehouse.md)
