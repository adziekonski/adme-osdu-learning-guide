# DG-05 · Identity, Naming and Versioning

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). Builds on
> [fundamentals 03 §2 and §5](../fundamentals/03-data-model.md#2-anatomy-of-a-record)
> (record IDs, versions, references). This doc is about **choosing IDs** and
> handling **re-runs, changes, duplicates and deletions**.

## TL;DR

- **An ID can never change**, so build it from something that never changes:
  `<partition>:<group-type>--<Entity>:<key>`, where the key is a **stable source
  key**, never a display name.
- **Master data** (wells, wellbores, fields): **you** set the ID from stable keys.
  **Files and WPCs:** generated IDs are fine, as long as your pipeline **remembers**
  them.
- Keep the source system's own identifiers in **`NameAliases`** on master
  records. They're your link back to the source and your key for matching
  duplicates.
- **Re-runs:** write with the same IDs and `skipdupes=true`. Unchanged records
  are skipped; changed ones get a new version.
- **Updating vs a new record:**
  - a master record keeps its ID and gains versions;
  - new **content** (a reprocessed log, a new file) is a **new** record.
- Plan for **deletes and duplicates** from the start. Soft deletes can be
  undone; purges can't.

## 1. Anatomy of a good ID

```text
contoso-prod-main : master-data--Wellbore : WELLDB-WB-101
└── partition ──┘   └── group + entity ─┘   └─ key ─────┘
```

- **Allowed characters in the key:** letters, digits, `_ - . :` and `%`
  (for URL-encoding). Spaces, slashes and other characters must be encoded or
  normalised. For example, `NO 1234/5-A` becomes `NO-1234-5-A` (normalised) or
  `NO%201234%2F5-A` (encoded).
- **Pick one normalisation rule** (case, separators, padding) and apply it
  everywhere. Never rely on `WB-101` and `wb-101` being treated as the same
  record.
- **What makes a good key:**

| Key source | Good? | Why |
|---|---|---|
| Source system's internal, never-reused ID (`WELLDB-WB-101`) | ✔ Best | Stable, unique, traceable to the source |
| Official identifier (national well ID, UWI) | ✔ Good, if every object has one | Stable and shared across systems, but often missing for old or planned wells |
| Display name (`North Star`, `NO 1234/5-A`) | ✘ | Names change, collide and contain awkward characters |
| File name (`WB-101-GR.las`) | ✘ | Renamed, copied, re-sent |
| Random GUID generated on every run | ✘ for master data | Every re-run creates duplicates |

- **Prefix with the source** (`WELLDB-…`) when different systems could produce
  the same key. Leave it out when the key is an official ID that you want to
  share across systems.

## 2. Who sets the ID, per group type

| Group type | Who sets the ID | Recommended |
|---|---|---|
| **Master data** (Field, Well, Wellbore, Organisation…) | **You** | A deterministic key from a stable source or official ID (§1) |
| **Reference data** (your values) | **You** | `<Type>:<Code>`, for example `WellboreReason:ContosoInfill` |
| **Work-product-components** (WellLog, Document…) | You or the system | Generated is fine. For safe re-runs, either derive the key deterministically (source system + source key, or a hash of them) or store generated IDs in your pipeline's mapping table |
| **Datasets** (files) | Usually the File service | Store the mapping from file checksum to dataset ID, so an unchanged file isn't uploaded and registered again |

**Your pipeline's mapping table**, from source key to OSDU ID, is worth keeping
in any case. It's the basis for re-runs, reconciliation (counting source vs
target) and lineage.

## 3. Source identifiers: `NameAliases`

Every master record should carry the identifiers the object has **in other
systems**:

```jsonc
"data": {
  "FacilityName": "NO 1234/5-A",
  "NameAliases": [
    { "AliasName": "WB-101",       "AliasNameTypeID": "…:reference-data--AliasNameType:WellDBId:" },
    { "AliasName": "1234/5-A",     "AliasNameTypeID": "…:reference-data--AliasNameType:RegulatoryIdentifier:" },
    { "AliasName": "NS-A (Petrel)", "AliasNameTypeID": "…:reference-data--AliasNameType:PetrelName:" }
  ],
  "Source": "WellDB"
}
```

- **`AliasName`** is the other system's identifier or name. **`AliasNameTypeID`**
  says which system or kind of name it is. Add one alias type per source system to
  the OPEN `AliasNameType` list, and use OSDU's own types such as
  `RegulatoryIdentifier` where they fit.
- **`Source`** records the system the record came from (free text).
- Labels for **object identifiers** (`WellDBId`) are different from labels for
  **reference codes** (`WellDBCode` in DG-04), even for the same system. One
  says "this is WellDB's ID for this wellbore", the other "this is how WellDB
  writes this value".
- `NameAliases` is indexed as **nested**, so you can search "which Wellbore has
  WellDB ID `WB-101`?". That makes it the link back to every source system.

## 4. Re-runs and versions

- **Re-loading the same data:** write with the **same IDs** and the Storage
  option **`skipdupes=true`**:
  - records whose content is identical are **skipped** (no new version);
  - changed records get a **new version**.

  Without the option, every run adds a version to every record.
- **New version or new record?**

| Change | Do | Why |
|---|---|---|
| A fact about a real-world object changes (well status, operator, name) | **New version** of the same master record | Identity stays; history is kept |
| Metadata about content changes (remarks, quality flag, status) | **New version** of the same WPC | Same content, better description |
| The **content** changes (reprocessed log, corrected trajectory, new file) | **New WPC and new dataset** | The old content stays available and traceable |
| Only the ACL, legal tags or `tags` change | Metadata update (no new version) | See [DG-02 §6](02-access-design.md#6-running-it-over-time) |

- **Pointing to versions:**
  - `…:WB-101:` (trailing colon) means **the latest version**, the normal choice;
  - `…:WB-101:1727712345678901` **pins** one version. Use it for **lineage**
    (`ancestry.parents`) and interpretations that must refer to exactly what was
    used.
- **Version growth:** versions pile up over the years. If needed, purge old
  versions of a record (the latest is always kept).

## 5. Duplicates

The same well often exists in several systems, under different names.

1. **Before loading, match** on strong keys: official IDs, the same source ID, or
   a combination of name and location within a tolerance.
2. **One object, one ID.** Pick the key from the most authoritative source, and
   put the other systems' identifiers in `NameAliases`.
3. **Found a duplicate after loading?**
   - keep the survivor;
   - add the duplicate's identifiers to the survivor's `NameAliases`;
   - **re-point** children and references (wellbores, logs…) to the survivor;
   - **delete** the duplicate.
4. **Report** match results and unresolved candidates to data managers. Don't
   merge automatically when you're unsure.

## 6. Deletions

| Action | Effect | Reversible? |
|---|---|---|
| **Soft delete** (`POST /records/{id}:delete` for one record; `POST /records/delete` for up to 500) | Record disappears from Search and normal reads | **Yes**: writing the same ID again restores it |
| **Purge versions** | Physically removes old versions (not the latest) | No |
| **Purge record** | Physically removes the record | No |

- **Source deletions:** your pipeline must **detect** objects that disappeared
  in the source, and decide whether to soft-delete them or flag them for review.
- **Children:** deleting a Well doesn't delete its Wellbores or logs. Nothing
  cascades, so handle children explicitly.

## 7. IDs in manifests

- **With deterministic IDs** (recommended for master data), a manifest simply
  contains the final IDs, and records can reference each other directly.
- **With generated IDs**, records in the same manifest can't know each other's
  final IDs. OSDU manifests therefore support temporary **surrogate keys**: a
  record refers to another record in the same manifest by a placeholder, and
  ingestion replaces it with the real ID. Check the exact syntax in the manifest
  ingestion documentation for your milestone.

## Common mistakes

- **Names or file names as keys.** They change and collide.
- **New random IDs on every run.** Duplicates everywhere.
- **Re-running without `skipdupes`.** Thousands of meaningless versions.
- **Overwriting content in place** (a reprocessed log as a new version of the old
  one). History and lineage are lost; create a new WPC instead.
- **No mapping table from source to OSDU IDs.** Re-runs, reconciliation and
  lineage become guesswork.
- **Forgetting children when deleting or merging.** You get orphaned or
  mis-linked records.

## Go deeper

- [OSDU Data Definitions: Group Type chapter (identity per group type)](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/Guides/Chapters/02-GroupType.md) (OSDU community)
- [Storage service API: create/update, `skipdupes`, versions, delete, purge](https://community.opengroup.org/osdu/platform/system/storage/-/blob/master/docs/docs/api.md) (OSDU community)
- [Manifest-based ingestion concepts](https://learn.microsoft.com/azure/energy-data-services/concepts-manifest-ingestion) (Microsoft Learn)
