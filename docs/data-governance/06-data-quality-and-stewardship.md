# DG-06 · Data Quality and Stewardship

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). This doc covers
> how to **check, record and improve** data quality, and who is responsible.

## TL;DR

- ADME enforces only the **envelope**: ACL, legal tags and, through manifest
  ingestion, the schema and references. Whether data is **complete, correct and
  fit for use** is your job.
- Check quality in **layers**: file transport, structure, references, meaning
  (units, CRS), governance, platform (indexed, searchable), and business
  acceptance.
- **Record** quality **on the data**, so users can see it:
  - **`TechnicalAssurances`**: trust levels such as Certified, Trusted, Suitable,
    Unsuitable or Unevaluated, with usage notes;
  - **curation status**: is the record still being worked on, or done?
  - optionally, OSDU's **DataQuality** records for detailed rule results.
- **Reconcile** every load against the source, and explain every difference.
- Make it **someone's job**: owners, stewards and custodians, a quarantine
  backlog, and a few KPIs on a dashboard.

## 1. Quality dimensions

OSDU's reference list `DataRuleDimensionType` names the usual dimensions. Use
the same words in your rules and reports:

| Dimension | Question | Example rule |
|---|---|---|
| **Completeness** | Is everything needed there? | A Wellbore has a `WellID`; a well log has curve units |
| **Conformance** | Does it follow the format and value rules? | Units and CRSs are valid reference values; dates are valid |
| **Consistency** | Do related values agree? | A log's depth range lies within the wellbore's total depth |
| **Accuracy** | Is it actually right? | Well position matches the regulator's register within 10 m |
| **Timeliness** | Is it current? | Daily production reported within 2 days |
| **Availability**, **Processability** | Can it be used? | The file exists, its checksum matches, and it opens |

## 2. Quality gates in the pipeline

| Layer | What to check | Who checks |
|---|---|---|
| **Transport** | File uploaded completely; checksum matches the source | Your pipeline |
| **Structure** | Record matches its schema | Manifest ingestion, or your pipeline for direct Storage writes |
| **References** | Parent records and reference values exist | Manifest ingestion, or your pipeline |
| **Meaning** | Units, CRS and depth references are present and plausible; codes are mapped | **Your pipeline** (the platform doesn't check this) |
| **Governance** | Correct partition, ACL groups and legal tag | Your pipeline (rules); Storage (existence and validity) |
| **Platform** | Stored, **indexed** (`index.statusCode` = 200), searchable, usable in the DDMS | Your post-load check |
| **Business** | A domain expert accepts completeness and meaning | Data steward or owner (sampling) |

A record that fails a gate goes to **quarantine** with a reason. It doesn't go
in with a guessed value.

## 3. Recording quality on the data

**Minimum, recommended for every master record and WPC:**
- **`TechnicalAssurances[]`**: how far the record can be trusted.

  ```jsonc
  "TechnicalAssurances": [ {
    "TechnicalAssuranceTypeID": "…:reference-data--TechnicalAssuranceType:Trusted:",
    "Reviewers": [ { "Name": "Petrophysics QC team" } ],
    "AcceptableUsage":   [ { "WorkflowUsageTypeID": "…:reference-data--WorkflowUsageType:…:",
                             "WorkflowPersonaTypeID": "…:reference-data--WorkflowPersonaType:…:" } ],
    "UnacceptableUsage": [ { "WorkflowUsageTypeID": "…:reference-data--WorkflowUsageType:…:" } ],
    "EffectiveDate": "2026-10-01",
    "Comment": "Depth-shifted to drilling depth; GR normalised" } ]
  ```

  The levels (FIXED list) are **Certified**, **Trusted**, **Suitable**,
  **Unsuitable** and **Unevaluated**. The usage lists say *for what* the data is
  or isn't trusted, for example "fine for regional mapping, not for well
  placement". Use this array; the older single field `TechnicalAssuranceID` is
  deprecated, as are the free-text `WorkflowUsage` and `WorkflowPersona`. The array
  is indexed as nested, so you can **search** by trust level and usage.
- **`ResourceCurationStatus`**: `Created` → `Curating` → `Curated`, so users can
  tell work in progress from finished data.

**Optional, detailed:** OSDU models **rules and their results** as data:
- rules as reference values, `DataQualityRule`, grouped into `DataQualityRuleSet`;
  both are LOCAL lists, so you define your own;
- per evaluation, a **`DataQuality`** record: which record, which rules, scores
  per dimension and a total score;
- per record, one **`DataQualitySummary`** with the latest results.

ADME stores these records, but **doesn't run the rules**. Run them with your own
engine (Fabric or Databricks notebooks, a pipeline step) and write the results
back. Start with `TechnicalAssurances`; add `DataQuality` records when you need
scores that can be audited.

## 4. Reconciliation after every load

Compare **source and target**, not just "the job succeeded":

| Check | How |
|---|---|
| **Counts** per entity type, asset and wave | Source extract vs Storage or Search counts |
| **Identity** | Every source key appears in the source-to-ID mapping table ([DG-05 §2](05-identity-naming-and-versioning.md#2-who-sets-the-id-per-group-type)) |
| **Files** | Sizes and checksums match |
| **Key values** | Sample important fields: names, depths, coordinates |
| **Relationships** | Children per parent; unresolved references = 0 or explained |
| **Governance** | Distribution of ACL groups and legal tags matches the design |
| **Searchability** | Records in Storage vs records found in Search; `index.statusCode` ≠ 200 = 0 or explained |
| **Rejects** | Quarantined and skipped items listed with reasons |

**Every difference gets a status:** *fixed*, *accepted*, *quarantined* or
*excluded*. Keep the report as evidence.

Combine **automated checks across all records** with **expert samples** of edge
cases: deviated and multilateral wellbores, unusual CRSs, large files,
restricted data.

## 5. Monitoring: a small set of KPIs

Export the metadata to Fabric or Databricks (see fundamentals doc 08) and track a
few indicators per partition and asset:

| KPI | Why it matters |
|---|---|
| % master records with WGS 84 location | Map search and geospatial analytics |
| % wellbores with a definitive trajectory | MD → TVD conversion |
| % well logs with curve units and curve families | Searching across vendors |
| % records with `TechnicalAssurances` ≠ Unevaluated | Coverage of quality review |
| Records with `index.statusCode` ≠ 200 | Hidden, unsearchable data |
| Quarantine backlog (count, age) | Is anyone fixing problems? |
| Legal tags expiring in the next 90 days | Data about to disappear ([DG-03](03-legal-tags.md)) |
| Candidate duplicates | Identity hygiene ([DG-05 §5](05-identity-naming-and-versioning.md#5-duplicates)) |

## 6. Roles

| Role | Typically | Responsible for |
|---|---|---|
| **Data owner** | Asset or discipline manager | Accountable for a data domain: approves rules and access ([DG-02](02-access-design.md)); agrees legal tags with legal and contracts ([DG-03](03-legal-tags.md)) |
| **Data steward** | Senior data manager or domain expert | Defines quality rules, works the quarantine, sets `TechnicalAssurances`, approves merges |
| **Data custodian** | Platform and data-engineering team | Runs pipelines, gates, reconciliation and monitoring; keeps configuration as code |
| **Data consumers** | Geoscientists, engineers, data scientists, apps | Report issues; respect `UnacceptableUsage` |

A simple loop keeps it alive:

```text
pipeline gates → quarantine → steward fixes mapping, source or value → reload → reconcile → KPIs
```

## Common mistakes

- **"Loaded" treated as "good".** A successful job says nothing about
  completeness or meaning.
- **Defaults instead of quarantine.** Guessed units, CRSs or codes poison every
  later use.
- **Quality known only to experts.** Write it on the record
  (`TechnicalAssurances`), so search results and applications can show it.
- **Reconciling counts only.** Missing relationships, wrong ACLs and unindexed
  records slip through.
- **A quarantine nobody owns.** It just grows; assign stewards and track its age.

## Go deeper

- [OSDU Data Definitions: schema catalogue (DataQuality, DataQualitySummary, TechnicalAssurance)](https://community.opengroup.org/osdu/data/data-definitions/-/tree/master/E-R) (OSDU community)
- [OSDU worked examples: data quality](https://community.opengroup.org/osdu/data/data-definitions/-/tree/master/Examples/WorkedExamples/DataQuality) (OSDU community)
- [Indexing and search workflows (index status)](https://learn.microsoft.com/azure/energy-data-services/concepts-index-and-search) (Microsoft Learn)
- [Troubleshoot manifest ingestion](https://learn.microsoft.com/azure/energy-data-services/troubleshoot-manifest-ingestion) (Microsoft Learn)
