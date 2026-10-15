# Glossary

> **Reference:** look up terms here; don't read it in order.

One-line definitions of the terms used in these docs. Follow the link for the
full explanation.

| Term | Meaning | See |
|---|---|---|
| **ACL** | Access control list on a record: `owners` (read, update, delete) and `viewers` (read) entitlement groups | [03](../fundamentals/03-data-model.md#6-governance-is-carried-by-every-record), [05](../fundamentals/05-platform-services.md#who-may-do-what-the-three-kinds-of-groups), [DG-02](../data-governance/02-access-design.md#4-acls-on-every-record-the-propagation-rule) |
| **ACL scope** | A set of data groups (for example `data.pl001.viewers` and `.owners`) that decides who sees and changes one slice of records, such as one licence; the lightest way to separate data inside a partition | [DG-01](../data-governance/01-partitions-and-environments.md#1-three-levels-of-separation), [DG-02](../data-governance/02-access-design.md#2-designing-data-scopes) |
| **ACZ (Analytics Consumption Zone)** | ADME-only, preview feature that continuously exports chosen kinds as Delta Parquet tables into your own Data Lake storage | [08](../fundamentals/08-analytics-and-ai.md#2-getting-data-out) |
| **ADME** | Azure Data Manager for Energy: Microsoft's fully managed Azure service that runs the OSDU Data Platform | [01](../fundamentals/01-why-osdu-and-adme.md#3-what-adme-is) |
| **Airflow / DAG** | The workflow engine behind ingestion; a DAG is one workflow definition, for example `Osdu_ingest` | [05](../fundamentals/05-platform-services.md#two-ways-into-storage) |
| **ancestry** | Record field listing the records it was derived from. It drives lineage and legal inheritance. | [03](../fundamentals/03-data-model.md#5-relationships) |
| **Azimuth / inclination** | Compass direction / angle from vertical of a wellbore at a survey station | [02](../fundamentals/02-energy-domain-primer.md#3-wells-and-wellbores) |
| **Basin, play, prospect** | Large sedimentary region; geological idea of how oil or gas is trapped; specific undrilled target | [02](../fundamentals/02-energy-domain-primer.md#2-where-things-are-from-basin-to-reservoir) |
| **Bin grid** | The layout of a 3D seismic survey's inline/crossline grid | [02](../fundamentals/02-energy-domain-primer.md#7-seismic-in-one-page) |
| **CRS** | Coordinate reference system: what a pair of coordinates means on Earth (geographic or projected) | [02](../fundamentals/02-energy-domain-primer.md#5-location-coordinates-need-a-crs) |
| **CSV parser** | OSDU workflow that turns CSV rows into records of a registered custom schema | [07](../fundamentals/07-what-goes-in-what-comes-out.md#4-the-built-in-routes) |
| **Custom kind** | A schema you register yourself, under your own authority (for example `contoso:welldb:…`), for data the well-known schemas don't model; a last resort | [DG-04](../data-governance/04-schemas-and-reference-data.md#3-custom-kinds) |
| **Data group** | Entitlement group named `data.*` that appears in record ACLs and decides who can see or change those records; changes rarely | [DG-02](../data-governance/02-access-design.md#1-the-three-layer-pattern) |
| **Data owner / steward / custodian** | Data roles: the owner is accountable for a data domain, the steward defines quality rules and fixes issues, the custodian runs the pipelines and platform | [DG-06](../data-governance/06-data-quality-and-stewardship.md#6-roles) |
| **Data partition** | Isolated data space in an ADME instance with its own groups, legal tags and reference data; identified by its partition ID, usually `{instance}-{partition}`; use the exact ID the instance lists | [05](../fundamentals/05-platform-services.md#1-how-you-talk-to-adme), [DG-01](../data-governance/01-partitions-and-environments.md#1-three-levels-of-separation) |
| **Dataset (group type)** | Record describing a file or file collection (name, size, checksum, location), with no business meaning | [03](../fundamentals/03-data-model.md#4d-dataset-the-file-itself) |
| **Datum** | Model of the Earth's shape plus its anchor point (for example WGS 84, ED50); also the zero point for depths (KB, MSL…) | [02](../fundamentals/02-energy-domain-primer.md#5-location-coordinates-need-a-crs) |
| **DDMS** | Domain Data Management Service: a domain-specific service with optimised bulk storage and API (Wellbore, Seismic, Reservoir, Rock & Fluid Samples) | [06](../fundamentals/06-ddms-and-extensions.md#2-what-a-ddms-is-and-why-it-exists) |
| **Deviated wellbore** | Wellbore deliberately drilled at an angle; **horizontal** wellbores run roughly flat along the reservoir | [02](../fundamentals/02-energy-domain-primer.md#3-wells-and-wellbores) |
| **DLIS / LAS** | Well-log file formats: binary, many logs per file / text, usually one log per file | [02](../fundamentals/02-energy-domain-primer.md#6-measuring-along-the-wellbore) |
| **Document** | WPC describing a report, scan or other document; the file itself is a dataset | [04](../fundamentals/04-entities-and-files.md#1-the-entities-you-meet-first) |
| **EDS (External Data Services)** | Pulls metadata from another OSDU platform on a schedule; files stay at the source until "naturalised" | [06](../fundamentals/06-ddms-and-extensions.md#4-other-extensions-and-adme-specific-additions) |
| **Entitlements** | OSDU service and model for groups: `users.*` (bundles), `service.*` (API permissions), `data.*` (record access) | [05](../fundamentals/05-platform-services.md#who-may-do-what-the-three-kinds-of-groups), [DG-02](../data-governance/02-access-design.md#1-the-three-layer-pattern) |
| **EPSG code** | Standard ID for a CRS, for example `EPSG:4326` = WGS 84 | [02](../fundamentals/02-energy-domain-primer.md#5-location-coordinates-need-a-crs) |
| **ETP** | Energistics Transfer Protocol: fast WebSocket protocol used by the Reservoir DDMS | [06](../fundamentals/06-ddms-and-extensions.md#3-the-four-ddms-in-adme) |
| **ExtensionProperties** | Free-form field in every schema for extra attributes; stored and returned, but not validated or searchable | [DG-04](../data-governance/04-schemas-and-reference-data.md#2-where-do-extra-fields-go) |
| **Field** | Proven oil or gas accumulation being developed or produced | [02](../fundamentals/02-energy-domain-primer.md#2-where-things-are-from-basin-to-reservoir) |
| **FIXED / OPEN / LOCAL** | How a reference value list is governed: FIXED by OSDU only, OPEN as an OSDU baseline you can add to, LOCAL entirely yours | [03](../fundamentals/03-data-model.md#4a-reference-data-the-lookup-values), [DG-04](../data-governance/04-schemas-and-reference-data.md#5-managing-reference-data) |
| **Formation top / marker** | Depth where a rock layer starts in a wellbore (`WellboreMarkerSet.Markers[]`) | [02](../fundamentals/02-energy-domain-primer.md#6-measuring-along-the-wellbore) |
| **GeoContexts** | Shared field linking a record to its field, country, basin, play or prospect | [03](../fundamentals/03-data-model.md#5-relationships) |
| **Group type** | Category of an entity type (reference, master, work-product-component, dataset, work product); the prefix before `--` in `kind` and `id` | [03](../fundamentals/03-data-model.md#4-group-types) |
| **Horizon / fault** | Interpreted layer boundary / fracture surface picked on seismic | [02](../fundamentals/02-energy-domain-primer.md#7-seismic-in-one-page) |
| **Index augmenter** | Configuration that adds derived fields to search documents, for example a wellbore's name on its logs | [05](../fundamentals/05-platform-services.md#after-storage-indexing-and-search) |
| **Indexer** | Internal service that turns stored records into search documents, one index per kind | [05](../fundamentals/05-platform-services.md#after-storage-indexing-and-search) |
| **Instance** | One ADME resource. It sets the region, network, encryption keys, tier and upgrade timing, and holds one or more data partitions | [DG-01](../data-governance/01-partitions-and-environments.md#1-three-levels-of-separation) |
| **kind** | Schema identifier of a record: `{authority}:{source}:{group-type}--{Entity}:{version}` | [03](../fundamentals/03-data-model.md#2-anatomy-of-a-record) |
| **Legal tag** | Named policy (country of origin, contract, expiry…) every record must reference; expired tags make data unavailable | [03](../fundamentals/03-data-model.md#6-governance-is-carried-by-every-record), [DG-03](../data-governance/03-legal-tags.md#1-what-a-legal-tag-contains) |
| **Manifest** | JSON package of reference data, master data, datasets, WPCs and a work product, loaded and validated by the manifest ingestion workflow | [07](../fundamentals/07-what-goes-in-what-comes-out.md#4-the-built-in-routes) |
| **Master data** | Real-world things and business activities with stable identities (Well, Wellbore, Field…) | [03](../fundamentals/03-data-model.md#4b-master-data-the-real-world-things) |
| **MD / TVD / TVDSS** | Measured depth along the hole / true vertical depth / TVD below mean sea level | [02](../fundamentals/02-energy-domain-primer.md#4-depth-always-ask-along-what-and-from-where) |
| **meta[]** | Record field declaring the units and CRS used by its properties | [03](../fundamentals/03-data-model.md#2-anatomy-of-a-record) |
| **Milestone** | OSDU platform release (M1…M26); ADME runs M26 as of October 2026 | [01](../fundamentals/01-why-osdu-and-adme.md#2-what-osdu-is) |
| **Mnemonic / curve family** | Vendor's short curve name (`GR`) / shared classification of what it measures (Gamma Ray) | [02](../fundamentals/02-energy-domain-primer.md#6-measuring-along-the-wellbore) |
| **NameAliases / AliasNameType** | The identifiers an object has in other systems (`AliasName`), each labelled with its system or kind of name (`AliasNameTypeID`); searchable, so they link a record back to its sources | [DG-05](../data-governance/05-identity-naming-and-versioning.md#3-source-identifiers-namealiases) |
| **OpenVDS / ZGY / SEG-Y** | Compressed cloud-friendly seismic formats / the classic seismic interchange format | [02](../fundamentals/02-energy-domain-primer.md#7-seismic-in-one-page) |
| **OSDU** | Industry standard (data model + APIs) and open-source data platform, governed by the OSDU Forum at The Open Group | [01](../fundamentals/01-why-osdu-and-adme.md#2-what-osdu-is) |
| **otherRelevantDataCountries** | Record-level legal field: the country the data was ingested from (or, for derived data, created in) | [DG-03](../data-governance/03-legal-tags.md#3-using-tags-on-records) |
| **Petrel Data Services** | SLB's (non-OSDU) extension on ADME for storing Petrel projects and extracting their data | [06](../fundamentals/06-ddms-and-extensions.md#4-other-extensions-and-adme-specific-additions) |
| **Quality gate / quarantine** | A check a record must pass when it's loaded; a record that fails goes to quarantine with a reason, instead of being loaded with guessed values | [DG-06](../data-governance/06-data-quality-and-stewardship.md#2-quality-gates-in-the-pipeline) |
| **Reconciliation** | Comparing what a load sent with what actually landed (counts, IDs, checksums, indexing) after every run | [DG-06](../data-governance/06-data-quality-and-stewardship.md#4-reconciliation-after-every-load) |
| **Record** | JSON document with an envelope (`id`, `kind`, `acl`, `legal`…) and a `data` payload | [03](../fundamentals/03-data-model.md#2-anatomy-of-a-record) |
| **Reference data** | Controlled lists of values (units, CRSs, types…), each value a record; governed as FIXED, OPEN or LOCAL | [03](../fundamentals/03-data-model.md#4a-reference-data-the-lookup-values), [DG-04](../data-governance/04-schemas-and-reference-data.md#5-managing-reference-data) |
| **RESQML** | Energistics exchange standard for reservoir and earth models | [06](../fundamentals/06-ddms-and-extensions.md#3-the-four-ddms-in-adme) |
| **Schema** | JSON Schema registered in the Schema service that defines a kind | [03](../fundamentals/03-data-model.md#3-kinds-and-schemas) |
| **Side-car** | A second record of your own kind that holds extra searchable fields and points to a standard record by ID | [DG-04](../data-governance/04-schemas-and-reference-data.md#2-where-do-extra-fields-go) |
| **Sidetrack** | New wellbore drilled from a kick-off point in an existing one; linked with `KickOffWellbore` | [02](../fundamentals/02-energy-domain-primer.md#3-wells-and-wellbores) |
| **skipdupes** | Storage option for re-runs: records whose content hasn't changed are skipped, so no new version is created | [DG-05](../data-governance/05-identity-naming-and-versioning.md#4-re-runs-and-versions) |
| **Soft delete / purge** | A soft delete hides a record from search and normal reads, and can be undone; a purge removes a record or its old versions permanently | [DG-05](../data-governance/05-identity-naming-and-versioning.md#6-deletions) |
| **Storage service** | Core service that saves, versions and returns records by ID | [05](../fundamentals/05-platform-services.md#2-the-services-and-which-are-osdu-or-adme) |
| **Surrogate key** | A placeholder ID inside a manifest, so records in the same manifest can refer to each other before their real IDs exist | [DG-05](../data-governance/05-identity-naming-and-versioning.md#7-ids-in-manifests) |
| **Team group** | Entitlement group named `users.*` for a team; it's a member of data groups and API bundles, so people changes never touch records | [DG-02](../data-governance/02-access-design.md#1-the-three-layer-pattern) |
| **TechnicalAssurances** | Record field saying how far the record can be trusted: assurance level, reviewers, and acceptable and unacceptable uses | [DG-06](../data-governance/06-data-quality-and-stewardship.md#3-recording-quality-on-the-data) |
| **Tier** | ADME offering level: Developer (non-production) or Standard (production, SLA) | [01](../fundamentals/01-why-osdu-and-adme.md#tiers) |
| **Trajectory** | 3D path of a wellbore from survey stations (MD, inclination, azimuth) | [04](../fundamentals/04-entities-and-files.md#3-worked-example-a-wellbore-with-its-trajectory-and-a-log) |
| **users.data.root** | Entitlement group that is an implicit owner of every record | [03](../fundamentals/03-data-model.md#6-governance-is-carried-by-every-record) |
| **users@ / users.datalake.\*** | The mandatory "member of this partition" group / predefined bundles of API permissions (viewers, editors, admins, ops) | [05](../fundamentals/05-platform-services.md#how-membership-works) |
| **Version (record)** | Every change to a record's data creates a new version. A reference ending in `:` means the latest version; a version number pins one | [DG-05](../data-governance/05-identity-naming-and-versioning.md#4-re-runs-and-versions) |
| **Well / wellbore** | Surface installation and business asset / one drilled path; a well can have several wellbores | [02](../fundamentals/02-energy-domain-primer.md#3-wells-and-wellbores) |
| **WellLog / curve** | WPC describing measurements along a wellbore / one measured property in it | [02](../fundamentals/02-energy-domain-primer.md#6-measuring-along-the-wellbore) |
| **WGS 84 / ED50** | Global satellite-based datum used by GPS and OSDU search / older European regional datum | [02](../fundamentals/02-energy-domain-primer.md#5-location-coordinates-need-a-crs) |
| **WITSML / PRODML** | Energistics XML standards for drilling and production data | [04](../fundamentals/04-entities-and-files.md#2-files-formats-and-where-they-land) |
| **wks** | "Well-known schema": a schema published by OSDU (authority `osdu`, source `wks`) | [03](../fundamentals/03-data-model.md#2-anatomy-of-a-record) |
| **Work product** | Record grouping the WPCs delivered together; used for audit | [03](../fundamentals/03-data-model.md#4e-work-product-the-delivery-bundle) |
| **WPC (work-product-component)** | Record describing the business meaning of digital content (log, trajectory, document…) and linking it to master data and files | [03](../fundamentals/03-data-model.md#4c-work-product-component-wpc-what-the-data-means) |
