# DG-03 · Legal Tags

> **Facts as of:** October 2026 (ADME on OSDU milestone M26). Builds on
> [fundamentals 03 §6](../fundamentals/03-data-model.md#6-governance-is-carried-by-every-record)
> (what a legal tag is). This doc is about **designing** legal tags and running
> them over time.

## TL;DR

- A legal tag is a **named, reusable policy**: country of origin, contract,
  expiry date, data type, classifications. **Every record** must reference at
  least one valid tag.
- Design **one tag per distinct legal situation**, for example "Norwegian
  licence data under contract X, confidential, until 2032". Not one tag per
  record, and not one tag for everything.
- **Most properties can't be changed later.** Only the description, contract ID,
  expiry date and extension properties can be updated. Get the country, data type
  and classifications right, or create a new tag and re-tag the records.
- **Expiry matters:** when a tag expires, its records, **and records derived from
  them**, stop being served until the tag is renewed. Monitor expiry dates.
- Legal and contracts teams **decide**, data managers **create**, and pipelines
  only **reference** existing tags. Keep the tag list as code, next to the groups.

## 1. What a legal tag contains

| Property | Meaning | Allowed values (OSDU defaults; check your partition) | Updatable? |
|---|---|---|---|
| `name` | Unique name; ADME prefixes it with the partition ID (`<partition-id>-`) | 3–100 characters, letters, digits, hyphens | No |
| `description` | Free text | Any | **Yes** |
| `countryOfOrigin` | Where the data **originally came from** (not where it was sent from) | ISO alpha-2 codes from the partition's allowed list | No |
| `contractId` | The governing contract | 3–40 characters, or `Unknown` / `No Contract Related` | **Yes** |
| `expirationDate` | Last valid day (inclusive) | `yyyy-MM-dd` in the future. May be empty (treated as never) only when the contract is `Unknown` or `No Contract Related`; required for, e.g., third-party data | **Yes** |
| `originator` | Supplier or client the data came from | Name | No |
| `dataType` | Ownership type | `Public Domain Data`, `First Party Data`, `Second Party Data`, `Third Party Data`, `Transferred Data` | No |
| `securityClassification` | Sensitivity | `Public`, `Private`, `Confidential` (no `Secret` data) | No |
| `exportClassification` | Export control | For example `EAR99`, `0A998` | No |
| `personalData` | Personal data present? | `Personally Identifiable`, `No Personal Data` (no sensitive personal data) | No |
| `extensionProperties` | Your own attributes, for example the contract-system ID or owner | Any JSON | **Yes** |

`GET /api/legal/v1/legaltags:properties` returns the values allowed in **your**
partition.

## 2. Designing the set of tags

**Granularity:** create a new tag whenever any of these differ:
- country of origin;
- contract (or contract ID and expiry);
- data type (own, partner, third-party, public);
- security classification;
- personal data or export classification.

| Example tag (after the ADME prefix) | Use for |
|---|---|
| `no-own-confidential` | Own Norwegian data, no contract, confidential |
| `no-pl001-jv-confidential` | Data under the PL001 joint-venture agreement |
| `gb-vendorx-seismic-2031` | Licensed third-party seismic from the UK (ISO code GB), contract with expiry 2031 |
| `public-no-sodir` | Public data from a government portal, here the Norwegian Offshore Directorate |

**Naming:** make names **descriptive and stable**, for example
`<country>-<source or contract>-<classification>[-<expiry year>]`. Put the
details and owner in `description` and `extensionProperties`.

**Tags and ACLs are independent.** The legal tag says whether the data **may be
stored and used at all**; ACL groups ([DG-02](02-access-design.md)) say **who**
may use it. A licence often drives both: `data.pl001.viewers` for access and
`…-no-pl001-jv-confidential` for legal status.

## 3. Using tags on records

```json
"legal": {
  "legaltags": ["contoso-prod-main-no-pl001-jv-confidential"],
  "otherRelevantDataCountries": ["NO"]
}
```

- **`legaltags`:** one or more tags. Use **several** when a record genuinely falls
  under several agreements.
- **`otherRelevantDataCountries`** (on the record, not the tag): the country the
  data was **ingested from**, or for derived records where they were
  **created**. The platform adds the country where it's **stored**. List it
  even if it's the same as the tag's country of origin; the two fields record
  different facts. It's required, and its valid codes come from a separate
  allowed list (`GET /api/legal/v1/legaltags:properties`).
- **Derived data:** for records computed from others (interpretations, ML
  results, merged datasets), list the parents in `ancestry.parents`. OSDU
  requires the derived record to carry the **union of the parents' tags**, and
  Storage fills the legal section from the parents. If any parent's tag
  expires, the derived record becomes invalid too.
- **Pipelines** should choose a tag by **rule** (for example source system +
  licence → tag) and check it with `POST /api/legal/v1/legaltags:validate` before
  loading. Unknown legal status goes to a **quarantine**, never to a permissive
  default tag.

## 4. Countries of origin: not all are allowed

Each partition allows only some countries as a tag's `countryOfOrigin`. In
OSDU's default list, only about **23 of 245** are allowed, including US, GB, NO,
CA and AU. Many others, such as **NL, DE, BR, AE, SA and IN**, are blocked
until Microsoft unlocks them per partition through a **support ticket**. That's
the customer's decision and risk, so get legal approval first.

**Before onboarding data from a new country**, check
`GET /api/legal/v1/legaltags:properties`. If the country is missing, raise the
ticket early: tag creation, and therefore ingestion, will fail until it's done.

## 5. Changing and expiring tags

| Situation | What to do |
|---|---|
| **Contract extended** | Update `expirationDate` (and `contractId` if needed) |
| **Wrong country, data type or classification** | These can't be changed. Create a new tag and replace it on the records with the Storage metadata-update API (up to 500 records per call; no new version). |
| **Tag expired** | Records using it, and their derivatives, are no longer returned by Storage or Search. **Renew** the expiry date to restore them; a once-a-day job updates the tag's status. |
| **Data must go** | Expiry hides data but doesn't erase it. Delete or purge the records if they must be removed. |

**Monitor expiry:**
- Run a scheduled check that lists tags expiring in the next 30, 60 or 90 days
  (`GET /api/legal/v1/legaltags`, compare `expirationDate`) and any invalid
  ones (`?valid=false`), and alerts the tag owners.
- OSDU defines "changed" and "about to expire" notifications, but their delivery
  depends on the platform deployment. On ADME, don't rely on them; build your
  own check.

**Side effects to know:**
- ADME's reference-value upgrades **skip** reference records whose legal tag has
  expired.
- Copies outside ADME (Fabric, Databricks) **don't** expire automatically. Your
  export must remove or hide them when a tag becomes invalid
  ([fundamentals 08 §4](../fundamentals/08-analytics-and-ai.md#4-governance-when-data-leaves-adme)).

## 6. Roles and process

| Role | Responsibility |
|---|---|
| **Legal and contracts** | Decide which tags exist and their properties; own expiry dates |
| **Data managers** (`users.datalake.editors`, which includes legal editing) | Create and update tags; map sources to tags; handle quarantine |
| **Ingestion pipelines** | Reference existing tags only; validate before loading; never create tags on the fly |
| **Platform team** | Expiry monitoring; support tickets to unlock countries of origin |

Keep the tag definitions in source control with the groups. Apply them per
partition and environment with an idempotent script: create missing tags, report
drift. Remember that most properties can't be changed after creation.

```http
POST /api/legal/v1/legaltags
data-partition-id: contoso-prod-main

{ "name": "no-pl001-jv-confidential",
  "description": "PL001 joint venture data; owner: Legal NO",
  "properties": {
    "countryOfOrigin": ["NO"], "contractId": "PL001-JOA",
    "expirationDate": "2032-12-31", "originator": "Contoso",
    "dataType": "First Party Data", "securityClassification": "Confidential",
    "exportClassification": "EAR99", "personalData": "No Personal Data" } }
```

ADME stores this as `contoso-prod-main-no-pl001-jv-confidential`.

## Common mistakes

- **One universal tag for everything.** Expiry or a legal change then hits all
  data at once, and the tag says nothing.
- **One tag per file or record.** Unmanageable, and there's nothing to reuse.
- **A default "permissive" tag for unknown data.** It's a compliance risk; use a
  quarantine instead.
- **Ignoring expiry dates.** Data disappears without warning, including derived
  results built on it.
- **Fixing a wrong classification by editing the tag.** Not possible; plan for
  a new tag and re-tagging.
- **Forgetting copies.** ADME enforces tags only inside ADME.

## Go deeper

- [Manage legal tags](https://learn.microsoft.com/azure/energy-data-services/how-to-manage-legal-tags) and [enable restricted countries of origin](https://learn.microsoft.com/azure/energy-data-services/how-to-enable-legal-tags-restricted-country-of-origin) (Microsoft Learn)
- [OSDU Legal service: properties, derivatives, updates, notifications](https://community.opengroup.org/osdu/platform/security-and-compliance/legal/-/blob/master/docs/docs/api.md) (OSDU community)
- [Reference data values and expired legal tags](https://learn.microsoft.com/azure/energy-data-services/concepts-reference-data-values) (Microsoft Learn)
