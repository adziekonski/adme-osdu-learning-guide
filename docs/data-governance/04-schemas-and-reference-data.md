# DG-04 · Schemas and Reference Data

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). Builds on
> [fundamentals 03 §3–§4a](../fundamentals/03-data-model.md#3-kinds-and-schemas)
> (what schemas, kinds and reference data are). This doc is about **choosing,
> extending and maintaining** them.

## TL;DR

- **Use the OSDU well-known schemas (`osdu:wks:…`)** wherever they fit. That's where
  interoperability with applications comes from.
- Choose **one schema version per entity type** for your loads. Applications and
  searches should use **wildcards** for minor and patch versions
  (`…Wellbore:1.*.*`), because several versions coexist.
- **Extra fields:**
  - `data.ExtensionProperties` holds them, but they are **not searchable**;
  - if they must be searchable, add a **side-car** custom kind that points to the
    OSDU record;
  - **never** add undeclared fields to a well-known record.
- **Reference data:**
  - you own your **extensions to OPEN lists** and your **LOCAL lists**;
  - create them as records;
  - keep the translation of source-system codes in a **versioned mapping
    specification**, and quarantine codes without a translation;
  - **retire** values instead of deleting them.
- Every schema and value you add is **per partition**. Keep them as code and
  apply them to every partition and environment.

## 1. Choosing schema versions

- **What's on your instance:** each milestone upgrade brings the well-known
  schema versions current at that time. List what's registered with the Schema
  service, for example:

  ```http
  GET /api/schema-service/v1/schema?authority=osdu&source=wks&entityType=master-data--Wellbore&latestVersion=true
  ```
- **Pick one version per entity type** for new loads, and record the choice in
  your mapping specification. Moving to a newer version is a planned change
  (§4), not something each pipeline decides on its own.
- **Versions coexist.** Records keep the kind they were written with, and Search
  has a separate index per kind version. Query with wildcards for minor and patch
  versions (`osdu:wks:master-data--Wellbore:1.*.*`) so you find all of them, and
  don't hard-code full versions in applications.
- **Optional platform rule:** OSDU's reference list `PreferredSchemaVersion`
  (LOCAL) can record your preferred, minimum and maximum kind per entity type,
  for applications that check it.

## 2. Where do extra fields go?

| Need | Where | Searchable? | Notes |
|---|---|---|---|
| A field the well-known schema already has | The schema's own field | Yes | Check the latest version first; many "missing" fields exist under another name |
| Extra attributes that are kept but not searched (source-system details, raw values) | `data.ExtensionProperties` | **No** | Every schema has it; content isn't validated or indexed |
| Extra attributes you must **search** on | A **side-car** custom kind that holds them and points to the OSDU record by ID | Yes | OSDU's own pattern for domain extensions (for example WellPlanningWell next to Well) |
| A business object OSDU doesn't model at all | A full **custom kind** | Yes | Last resort; applications won't understand it |
| A field that's useful to everyone | Propose it to the OSDU data-definitions community | — | Meanwhile, park it in `ExtensionProperties` |

**Never add undeclared properties directly into `data`** of a well-known record.
OSDU states that they aren't indexed, so they're invisible to Search, and they
break the meaning of the standard schema.

**`ExtensionProperties` vs a side-car.** Values in `ExtensionProperties` are
**kept** (Storage returns them) but you **can't search by them**. A side-car is
a **second record** of your own kind, with your fields declared in its schema
(so they're indexed), and a link to the standard record:

```text
osdu:wks:master-data--Wellbore:1.5.1                contoso:welldb:master-data--WellboreSideCar:1.0.0
┌───────────────────────────────────┐                ┌───────────────────────────────────────────┐
│ id: …:master-data--Wellbore:NO-1A │ ◀─ WellboreID ─│ id: …:master-data--WellboreSideCar:NO-1A    │
│ standard OSDU fields only         │                │ LegacyStatus, CostCentre, AssetTeam …       │
│ (every application understands it)│                │ (your fields: indexed, searchable)          │
└───────────────────────────────────┘                └───────────────────────────────────────────┘
```

- It's **not inheritance.** The standard record stays a standard
  `osdu:wks:…` record, so applications and searches still find it. A
  "Wellbore copy with extra fields" under your own kind would no longer be an
  OSDU Wellbore.
- To find wellbores by your fields: search the side-car kind, collect the
  `WellboreID`s, then fetch the wellbores. That's the usual two-step.
- Write the standard record first, then the side-car with the same key and the
  same ACL and legal tags. Retire both together.
- Declare the link with `x-osdu-relationship`, and mark the schema with
  `x-osdu-side-car-type-to`, as OSDU does for `WellPlanningWellbore`.

## 3. Custom kinds

- **Naming:** `<authority>:<source>:<group-type>--<EntityType>:<major>.<minor>.<patch>`,
  for example `contoso:welldb:master-data--WellboreSideCar:1.0.0`.
  - Authority and source in lowercase; the entity type in PascalCase.
  - Never use the reserved authority `system-meta-data`.
  - Avoid kinds that differ only in letter case. Some services treat them as
    equal, others don't.
- **Reuse OSDU fragments:** a custom schema can include registered OSDU
  fragments with `$ref` by kind, for example the shared fields of master data.
  That gives your kind the usual envelope and common properties.
- **Lifecycle:** register it with `POST /api/schema-service/v1/schema` in status
  **DEVELOPMENT** (editable) and switch it to **PUBLISHED** (immutable) before
  production use. Your schemas are **INTERNAL**, so register them in **every
  partition and environment**.
- **Versioning:**
  - patch for fixes;
  - minor for new optional fields;
  - major for anything breaking.

  The Schema service rejects breaking changes in a minor or patch release.
- **Declare links and searchable arrays** in the schema: see the next
  subsection.

```json
{ "schemaInfo": {
    "schemaIdentity": { "authority": "contoso", "source": "welldb",
      "entityType": "master-data--WellboreSideCar",
      "schemaVersionMajor": 1, "schemaVersionMinor": 0, "schemaVersionPatch": 0 },
    "status": "DEVELOPMENT" },
  "schema": { "…": "JSON Schema, with $ref to OSDU fragments and a WellboreID relationship" } }
```

### Declaring links and searchable arrays

Two schema annotations tell the platform how to treat certain fields. They don't
change your data.

**1. `x-osdu-relationship`: "this text is a link to another record"**

A field like `WellboreID` holds another record's ID. Without the annotation, the
platform sees only a string.

```text
                         WITHOUT annotation          WITH x-osdu-relationship + pattern
"WellboreID": "…"   →    just text                   documented as a link to a Wellbore
  wrong format?          accepted                    rejected (pattern)
  tools, apps,           can't tell it's a link      can follow it
  index augmentation
```

Whether the **target exists** is a separate check: manifest ingestion checks
references, and direct Storage writes don't ([fundamentals 05 §3](../fundamentals/05-platform-services.md#two-ways-into-storage)).

```json
"WellboreID": {
  "type": "string",
  "pattern": "^[\\w\\-\\.]+:master-data\\-\\-Wellbore:[\\w\\-\\.\\:\\%]+:[0-9]*$",
  "x-osdu-relationship": [ { "GroupType": "master-data", "EntityType": "Wellbore" } ]
}
```

**Rule:** annotate **every** field that holds another record's ID, as OSDU's own
schemas do.

**2. `x-osdu-indexing`: how Search handles an array of objects**

A record holds a list, for example formation tops:

```text
Markers: [ { MarkerName: "Top Brent", MarkerMeasuredDepth: 2000 },
           { MarkerName: "Top Ness",  MarkerMeasuredDepth: 3500 } ]

Question: "Is Top Brent deeper than 3,000 m?"   (correct answer for this record: NO)

 no hint (default) │ array is stored and shown in results, but can't be queried
                   │ → the question can't be asked
 "flattened"       │ all values mixed: {Top Brent, Top Ness} + {2000, 3500}
                   │ → finds "Top Brent" and a depth > 3,000: WRONG MATCH
 "nested"          │ each element kept separate; query with nested(data.Markers, …)
                   │ → Top Brent is at 2,000: correctly NO MATCH
```

```json
"Markers": { "type": "array", "x-osdu-indexing": { "type": "nested" }, "items": { "…": "…" } }
```

**Rule:**
- `nested` when users combine conditions **on the same element** (name **and**
  depth);
- `flattened` when they filter on one field at a time;
- no hint only for arrays nobody needs to search.

## 4. Moving to newer schema versions

- **Minor and patch versions** are backward compatible. Old records stay valid
  and searchable, so upgrading them is optional and can be gradual.
- **Your options for existing records:**
  1. **Leave them.** Search finds them with version wildcards.
  2. **Re-write them through your pipeline** with the new kind and the **same ID**.
     That creates a new version of each record and is the safest route, because
     your mapping and validation run again.
  3. **Bulk-change the kind** with Storage's patch API (up to 100 records per
     call). Validate the records against the new schema first.
- **Major versions** can rename or restructure fields. OSDU publishes migration
  guides and machine-readable upgrade rules (JOLT specifications). OSDU's Schema
  Upgrade service isn't part of ADME, but it can be deployed separately.
- **After each milestone upgrade**, test new schema versions on a non-production
  instance first ([DG-01 §4](01-partitions-and-environments.md#4-environments)).

## 5. Managing reference data

**What you own:**
- your **additions to OPEN lists**, for example a company-specific wellbore reason;
- your **LOCAL lists**, for example curve families or status codes;
- the synced LOCAL starter values, if you opted in when creating the instance.

FIXED and OPEN values from OSDU are synced and upgraded by ADME.

**Creating a value:** a value is a normal record. Load it before any data that
uses it.

```json
{ "id": "contoso-prod-main:reference-data--WellboreReason:ContosoInfill",
  "kind": "osdu:wks:reference-data--WellboreReason:1.0.0",
  "acl": { "owners": ["data.referencedata-contoso.owners@…"], "viewers": ["data.referencedata-contoso.viewers@…"] },
  "legal": { "legaltags": ["contoso-prod-main-referencedata-legal"], "otherRelevantDataCountries": ["NO"] },
  "data": { "Code": "ContosoInfill", "Name": "Infill (Contoso)",
            "Description": "Company-specific infill well",
            "NameAlias": [ { "AliasName": "INF", "AliasNameTypeID": "…:reference-data--AliasNameType:WellDBCode:" } ] } }
```

- **IDs:** `<partition>:reference-data--<Type>:<Code>`. Special characters in the
  code are URL-encoded, for example `Gamma%20Ray` or `g%2Fcm3`.
- **Groups:** use separate ACL groups for your own values, for example
  `data.referencedata-contoso.*`. Make them **readable by all partition users**,
  and keep writing restricted to data managers.

**Mapping source codes:** your source systems use their own codes (for example
WellDB writes `INF` for an infill well), and the pipeline must translate each one
into a reference ID.
- **Keep the mapping in a specification under source control**: source
  system, source code, reference value. It's the single source of truth for your
  pipelines.
- **Optionally, publish long-lived, simple mappings on the values themselves**,
  so everyone can see and search them. Add a `NameAlias` to the value:
  `AliasName` = the source code, `AliasNameTypeID` = a label for the source
  system (one value per system in the OPEN `AliasNameType` list, for example
  `WellDBCode`). Skip this for one-off migrations, context-dependent codes and
  FIXED lists.
- **Codes without a translation go to quarantine.** Don't load a guessed or
  default value. A data manager adds the mapping (or a new value), then the
  record is reloaded.

**Retiring a value:** set `InactiveIndicator: true` and **don't delete** it, because
existing records still point to it. If there's a replacement, follow OSDU's
convention in `Description`:
`DEPRECATED: Please use Code="NewCode" with Name="New name" instead.`

## 6. Roles and process

| Role | Responsibility |
|---|---|
| **Data architect** | Chooses schema versions; approves `ExtensionProperties` use, side-cars and custom kinds; plans upgrades |
| **Data managers** | Maintain your reference values and source-code mappings; retire values |
| **Pipelines** | Use the agreed kinds and mappings; quarantine unknown codes |
| **Platform team** | Apply schemas and values to every partition and environment as code; test after milestone upgrades |

## Common mistakes

- **Adding unknown fields to well-known records.** They're silently not
  indexed.
- **Expecting `ExtensionProperties` to be searchable.** It isn't; use a side-car
  kind if you need search.
- **Hard-coding full schema versions** in applications and queries.
- **Creating custom kinds for things OSDU already models.** You lose
  interoperability.
- **Deleting reference values.** Existing references break; set
  `InactiveIndicator` instead.
- **Mapping source codes only inside pipeline code.** Keep the mapping as a
  versioned specification, and publish long-lived codes as `NameAlias`.

## Go deeper

- [OSDU Data Definitions: extensions, side-cars, schema upgrade, version management](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/Guides/Chapters/93-OSDU-Schemas.md) (OSDU community)
- [OSDU Data Definitions: introduction, trade-offs and extensions](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/Guides/Chapters/01-Introduction.md) (OSDU community)
- [Schema service API](https://community.opengroup.org/osdu/platform/system/schema-service/-/blob/master/docs/docs/api.md) (OSDU community)
- [Reference data values in ADME](https://learn.microsoft.com/azure/energy-data-services/concepts-reference-data-values) (Microsoft Learn)
- [Schema upgrade resources and worked examples](https://community.opengroup.org/osdu/data/data-definitions/-/tree/master/Examples/WorkedExamples/Upgrades) (OSDU community)
