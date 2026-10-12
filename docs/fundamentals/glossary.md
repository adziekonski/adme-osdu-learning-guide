# Glossary

One-line definitions of the terms used in these docs. Follow the link for the
full explanation.

| Term | Meaning | See |
|---|---|---|
| **ACL** | Access control list on a record: `owners` (read, update, delete) and `viewers` (read) entitlement groups | [03](03-data-model.md#6-governance-is-carried-by-every-record), [05](05-platform-services.md#who-may-do-what-the-three-kinds-of-groups) |
| **ACZ (Analytics Consumption Zone)** | ADME-only, preview feature that continuously exports chosen kinds as Delta Parquet tables into your own Data Lake storage | [08](08-analytics-and-ai.md#2-getting-data-out) |
| **ADME** | Azure Data Manager for Energy: Microsoft's fully managed Azure service that runs the OSDU Data Platform | [01](01-why-osdu-and-adme.md#3-what-adme-is) |
| **Airflow / DAG** | The workflow engine behind ingestion; a DAG is one workflow definition, for example `Osdu_ingest` | [05](05-platform-services.md#two-ways-into-storage) |
| **ancestry** | Record field listing the records it was derived from. It drives lineage and legal inheritance. | [03](03-data-model.md#5-relationships) |
| **Azimuth / inclination** | Compass direction / angle from vertical of a wellbore at a survey station | [02](02-energy-domain-primer.md#3-wells-and-wellbores) |
| **Basin, play, prospect** | Large sedimentary region; geological idea of how oil or gas is trapped; specific undrilled target | [02](02-energy-domain-primer.md#2-where-things-are-from-basin-to-reservoir) |
| **Bin grid** | The layout of a 3D seismic survey's inline/crossline grid | [02](02-energy-domain-primer.md#7-seismic-in-one-page) |
| **CRS** | Coordinate reference system: what a pair of coordinates means on Earth (geographic or projected) | [02](02-energy-domain-primer.md#5-location-coordinates-need-a-crs) |
| **CSV parser** | OSDU workflow that turns CSV rows into records of a registered custom schema | [07](07-what-goes-in-what-comes-out.md#4-the-built-in-routes) |
| **Data partition** | Isolated data space in an ADME instance with its own groups, legal tags and reference data; identified by its partition ID, usually `{instance}-{partition}`; use the exact ID the instance lists | [05](05-platform-services.md#1-how-you-talk-to-adme) |
| **Dataset (group type)** | Record describing a file or file collection (name, size, checksum, location), with no business meaning | [03](03-data-model.md#4d-dataset-the-file-itself) |
| **Datum** | Model of the Earth's shape plus its anchor point (for example WGS 84, ED50); also the zero point for depths (KB, MSL…) | [02](02-energy-domain-primer.md#5-location-coordinates-need-a-crs) |
| **DDMS** | Domain Data Management Service: a domain-specific service with optimised bulk storage and API (Wellbore, Seismic, Reservoir, Rock & Fluid Samples) | [06](06-ddms-and-extensions.md#2-what-a-ddms-is-and-why-it-exists) |
| **Deviated wellbore** | Wellbore deliberately drilled at an angle; **horizontal** wellbores run roughly flat along the reservoir | [02](02-energy-domain-primer.md#3-wells-and-wellbores) |
| **DLIS / LAS** | Well-log file formats: binary, many logs per file / text, usually one log per file | [02](02-energy-domain-primer.md#6-measuring-along-the-wellbore) |
| **Document** | WPC describing a report, scan or other document; the file itself is a dataset | [04](04-entities-and-files.md#1-the-entities-you-meet-first) |
| **EDS (External Data Services)** | Pulls metadata from another OSDU platform on a schedule; files stay at the source until "naturalised" | [06](06-ddms-and-extensions.md#4-other-extensions-and-adme-specific-additions) |
| **Entitlements** | OSDU service and model for groups: `users.*` (bundles), `service.*` (API permissions), `data.*` (record access) | [05](05-platform-services.md#who-may-do-what-the-three-kinds-of-groups) |
| **EPSG code** | Standard ID for a CRS, for example `EPSG:4326` = WGS 84 | [02](02-energy-domain-primer.md#5-location-coordinates-need-a-crs) |
| **ETP** | Energistics Transfer Protocol: fast WebSocket protocol used by the Reservoir DDMS | [06](06-ddms-and-extensions.md#3-the-four-ddms-in-adme) |
| **Field** | Proven oil or gas accumulation being developed or produced | [02](02-energy-domain-primer.md#2-where-things-are-from-basin-to-reservoir) |
| **Formation top / marker** | Depth where a rock layer starts in a wellbore (`WellboreMarkerSet.Markers[]`) | [02](02-energy-domain-primer.md#6-measuring-along-the-wellbore) |
| **GeoContexts** | Shared field linking a record to its field, country, basin, play or prospect | [03](03-data-model.md#5-relationships) |
| **Group type** | Category of an entity type (reference, master, work-product-component, dataset, work product); the prefix before `--` in `kind` and `id` | [03](03-data-model.md#4-group-types) |
| **Horizon / fault** | Interpreted layer boundary / fracture surface picked on seismic | [02](02-energy-domain-primer.md#7-seismic-in-one-page) |
| **Index augmenter** | Configuration that adds derived fields to search documents, for example a wellbore's name on its logs | [05](05-platform-services.md#after-storage-indexing-and-search) |
| **Indexer** | Internal service that turns stored records into search documents, one index per kind | [05](05-platform-services.md#after-storage-indexing-and-search) |
| **kind** | Schema identifier of a record: `{authority}:{source}:{group-type}--{Entity}:{version}` | [03](03-data-model.md#2-anatomy-of-a-record) |
| **Legal tag** | Named policy (country of origin, contract, expiry…) every record must reference; expired tags make data unavailable | [03](03-data-model.md#6-governance-is-carried-by-every-record) |
| **Manifest** | JSON package of reference data, master data, datasets, WPCs and a work product, loaded and validated by the manifest ingestion workflow | [07](07-what-goes-in-what-comes-out.md#4-the-built-in-routes) |
| **Master data** | Real-world things and business activities with stable identities (Well, Wellbore, Field…) | [03](03-data-model.md#4b-master-data-the-real-world-things) |
| **MD / TVD / TVDSS** | Measured depth along the hole / true vertical depth / TVD below mean sea level | [02](02-energy-domain-primer.md#4-depth-always-ask-along-what-and-from-where) |
| **meta[]** | Record field declaring the units and CRS used by its properties | [03](03-data-model.md#2-anatomy-of-a-record) |
| **Milestone** | OSDU platform release (M1…M26); ADME runs M26 as of October 2026 | [01](01-why-osdu-and-adme.md#2-what-osdu-is) |
| **Mnemonic / curve family** | Vendor's short curve name (`GR`) / shared classification of what it measures (Gamma Ray) | [02](02-energy-domain-primer.md#6-measuring-along-the-wellbore) |
| **OpenVDS / ZGY / SEG-Y** | Compressed cloud-friendly seismic formats / the classic seismic interchange format | [02](02-energy-domain-primer.md#7-seismic-in-one-page) |
| **OSDU** | Industry standard (data model + APIs) and open-source data platform, governed by the OSDU Forum at The Open Group | [01](01-why-osdu-and-adme.md#2-what-osdu-is) |
| **Petrel Data Services** | SLB's (non-OSDU) extension on ADME for storing Petrel projects and extracting their data | [06](06-ddms-and-extensions.md#4-other-extensions-and-adme-specific-additions) |
| **Record** | JSON document with an envelope (`id`, `kind`, `acl`, `legal`…) and a `data` payload | [03](03-data-model.md#2-anatomy-of-a-record) |
| **Reference data** | Controlled lists of values (units, CRSs, types…), each value a record; governed as FIXED, OPEN or LOCAL | [03](03-data-model.md#4a-reference-data-the-lookup-values) |
| **RESQML** | Energistics exchange standard for reservoir and earth models | [06](06-ddms-and-extensions.md#3-the-four-ddms-in-adme) |
| **Schema** | JSON Schema registered in the Schema service that defines a kind | [03](03-data-model.md#3-kinds-and-schemas) |
| **Sidetrack** | New wellbore drilled from a kick-off point in an existing one; linked with `KickOffWellbore` | [02](02-energy-domain-primer.md#3-wells-and-wellbores) |
| **Storage service** | Core service that saves, versions and returns records by ID | [05](05-platform-services.md#2-the-services-and-which-are-osdu-or-adme) |
| **Tier** | ADME offering level: Developer (non-production) or Standard (production, SLA) | [01](01-why-osdu-and-adme.md#tiers) |
| **Trajectory** | 3D path of a wellbore from survey stations (MD, inclination, azimuth) | [04](04-entities-and-files.md#3-worked-example-a-wellbore-with-its-trajectory-and-a-log) |
| **users.data.root** | Entitlement group that is an implicit owner of every record | [03](03-data-model.md#6-governance-is-carried-by-every-record) |
| **users@ / users.datalake.\*** | The mandatory "member of this partition" group / predefined bundles of API permissions (viewers, editors, admins, ops) | [05](05-platform-services.md#how-membership-works) |
| **Well / wellbore** | Surface installation and business asset / one drilled path; a well can have several wellbores | [02](02-energy-domain-primer.md#3-wells-and-wellbores) |
| **WellLog / curve** | WPC describing measurements along a wellbore / one measured property in it | [02](02-energy-domain-primer.md#6-measuring-along-the-wellbore) |
| **WGS 84 / ED50** | Global satellite-based datum used by GPS and OSDU search / older European regional datum | [02](02-energy-domain-primer.md#5-location-coordinates-need-a-crs) |
| **WITSML / PRODML** | Energistics XML standards for drilling and production data | [04](04-entities-and-files.md#2-files-formats-and-where-they-land) |
| **wks** | "Well-known schema": a schema published by OSDU (authority `osdu`, source `wks`) | [03](03-data-model.md#2-anatomy-of-a-record) |
| **Work product** | Record grouping the WPCs delivered together; used for audit | [03](03-data-model.md#4e-work-product-the-delivery-bundle) |
| **WPC (work-product-component)** | Record describing the business meaning of digital content (log, trajectory, document…) and linking it to master data and files | [03](03-data-model.md#4c-work-product-component-wpc-what-the-data-means) |
