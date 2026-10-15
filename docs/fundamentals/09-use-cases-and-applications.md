# 09 · Use Cases and Applications

> **Facts as of:** October 2026. Vendor support changes
> often, so check current ADME support with each vendor and on Azure Marketplace.

## TL;DR

- Most people never "use ADME" directly. They use **applications** (interpretation,
  modelling, data management, BI, AI) that read and write ADME through the OSDU APIs.
- The core business value is **one governed, searchable subsurface data
  foundation**. Every other use case builds on it: finding data faster, retiring
  legacy stores, swapping applications, sharing with partners, and feeding
  analytics and AI.
- The quickest **AI wins** are usually **data-management** ones: log QC, curve
  harmonisation, well matching across sources, document metadata extraction.
  They improve everything users find in ADME.
- The ecosystem has **domain application vendors** (SLB, Halliburton, AspenTech…),
  **specialist data and seismic tools** (Bluware, INT, Interica, Cegal, Katalyst,
  RoQC) and **system integrators** (Accenture, EPAM, Wipro…).
- Public examples, from published sources only (2022–2026): **Aker BP**,
  **Equinor** and **TotalEnergies**.

## 1. Who uses ADME, and how

| Role | What they do | How they touch ADME |
|---|---|---|
| **Geoscientists** (geologists, geophysicists) | Interpret seismic and wells, build subsurface models | Through domain applications such as Petrel, rarely directly |
| **Petrophysicists** | Evaluate rock and fluids from logs and cores | Log applications; the Wellbore DDMS behind them |
| **Drilling and reservoir engineers** | Plan wells, model and forecast reservoirs | Planning and simulation applications |
| **Data managers** | Load, QC, classify and govern data; answer data requests | Ingestion tools, Admin UI, search portals, APIs |
| **Data scientists and analysts** | BI, ML and AI on subsurface and operational data | Lakehouse copies ([doc 08](08-analytics-and-ai.md)), APIs |
| **Platform and IT teams** | Provision, secure and integrate the platform | Azure portal, Entitlements, monitoring |
| **Partners and regulators** | Receive or exchange agreed data sets | Exported data packages, External Data Services, or a dedicated partition |

## 2. Business use cases

| Use case | What it looks like | ADME capabilities used | Value |
|---|---|---|---|
| **Subsurface data foundation** | Consolidate wells, logs, seismic and reports from legacy databases, project files and file shares into one governed platform | Data model, ingestion workflows, DDMS, entitlements, legal tags | One source of truth; retire legacy stores |
| **Data discovery** | "Show me everything around this well or field": logs, trajectories, tops, seismic, reports | Search (text, map, nested), `GeoContexts`, relationships | Much faster search for data |
| **Well planning with offset wells** | Gather nearby wells' trajectories, logs, formation tops and drilling reports for a new well design | Spatial search, Wellbore DDMS, Documents | Safer, faster planning that uses all prior knowledge |
| **Seismic library management** | Catalogue surveys and volumes, stream them to interpretation, manage storage cost | Seismic DDMS, OpenVDS/ZGY conversion, storage tiers (preview) | Faster loading of huge volumes; lower storage cost |
| **Reservoir modelling and field development** | Share models, grids and scenarios between modelling and simulation tools | Reservoir DDMS (RESQML), interpretations | Less copying between tools; traceable decisions |
| **Application choice and vendor exchange** | Several vendors' tools work on the same data | OSDU APIs and schemas | Less lock-in; quicker onboarding of new tools |
| **Partner, joint-venture and divestment data** | Share or hand over an agreed data set with the right legal terms | Partitions, legal tags, ACL groups, External Data Services | Controlled, auditable sharing |
| **Production context** | Link reported volumes, well tests and downtime to wells and reservoirs | `ProductionValues`, `FlowTest`, `DowntimeEvent` + external time series | Subsurface and production views in one place |
| **Energy transition** | Assess and monitor CO₂ storage sites; potentially geothermal wells | The same well, seismic and model data | Reuse of oil and gas data and skills for new energy |

## 3. AI and ML use cases

How to run these, and the governance rules, are in [doc 08](08-analytics-and-ai.md).

| Use case | Inputs from ADME | Result, written back as |
|---|---|---|
| **Log quality control**: spikes, gaps, bad depth shifts | WellLog curves (Wellbore DDMS) | Quality flags (`TechnicalAssurances`), corrected log as a new WellLog |
| **Missing-curve prediction**, for example sonic from GR, density and neutron | Curves from many wells | Predicted curve as a new WellLog with `ancestry` |
| **Facies / lithology classification** | Curves + formation tops as labels | Interpreted curve or marker set |
| **Curve-family harmonisation**: mnemonic → standard family | `Curves[].Mnemonic` across vendors | Updated `LogCurveFamilyID` values |
| **Well matching (entity resolution)** across sources | Well and Wellbore names, aliases, coordinates | Merged wells with `NameAliases[]` |
| **Document metadata extraction**: OCR + LLM | Document records + PDF scans | Document type, dates, well links filled in |
| **Seismic fault and horizon detection** | Seismic volumes (OpenVDS from the Seismic DDMS) | `SeismicFault` / `SeismicHorizon` interpretations |
| **Production forecasting, anomaly detection** | ADME context + time series **outside** ADME | Usually stays in analytics |

**Generative AI scenarios:**
- **Document assistant:** "What did the end-of-well report say about losses in
  well X?" Retrieval over Document files, with citations and permission
  filtering.
- **Catalogue assistant:** "Which wellbores in field Alpha have density logs below
  3,000 m?" An agent that calls ADME Search as the user.
- **Analytics assistant:** "How many wells per field lack a definitive
  trajectory?" A Fabric data agent over curated tables.

**Where to start:** the data-management use cases (log QC, harmonisation, well
matching, document metadata) are usually the **quickest wins**. They need little
domain modelling, show results fast, and improve every later use case.

## 4. The application ecosystem

| Category | Examples (from Microsoft's ADME partner page and GA announcement) | Role |
|---|---|---|
| **Domain applications** | **SLB** (Petrel; Petrel Data Services on ADME), **Halliburton** (subsurface applications), **AspenTech** (asset optimisation) | Interpretation, modelling, planning, optimisation: the daily tools of geoscientists and engineers |
| **Seismic and visualisation** | **Bluware** (seismic data and interpretation; originator of OpenVDS), **INT** (IVAAP visualisation) | Fast access to and viewing of large subsurface data |
| **Data management and discovery** | **Interica** (OneView), **Cegal** (Cetegra), **Katalyst**, **RoQC** (LogQA, ML-based log QC) | Scanning legacy projects, QC, cataloguing, managed data services |
| **System integrators** | **Accenture**, **EPAM**, **Wipro** (WINS ingestion framework) | Migration, ingestion, governance design, integration |
| **Microsoft** | Fabric, Power BI, Azure ML, Microsoft Foundry | Analytics, BI and AI on top of ADME ([doc 08](08-analytics-and-ai.md)) |
| **OSDU community tools** | Admin UI, Geospatial Consumption Zone | Administration and map services ([doc 05](05-platform-services.md)) |

**What "works with ADME" means:** the application uses the OSDU APIs and
schemas, and should be tested against ADME's current milestone. Some vendors also ship
extensions that run inside ADME (Petrel Data Services). Ask vendors which
milestone and which DDMS they support.

## 5. Public customer examples

Only **published** sources are used here. The descriptions paraphrase what each
source states, and nothing more.

| Customer | What the source says | Source |
|---|---|---|
| **Aker BP** | With Microsoft, SLB and Halliburton, develops "Data Mesh", a framework on top of ADME that connects data from SLB, Halliburton and the Norwegian Petroleum Directorate in one tool. Field development today needs data from **about 150 applications** collected manually, and Data Mesh aims to solve this. Microsoft's GA announcement adds that integrating ADME with SLB and Halliburton applications helped Aker BP speed time-to-market. | [Aker BP press release](https://akerbp.com/en/aker-bp-collaborates-with-microsoft-slb-and-halliburton-to-streamline-data-sharing/) (Sep 2023); [ADME GA announcement](https://www.microsoft.com/en-us/microsoft-cloud/blog/energy-and-resources/2023/07/19/announcing-general-availability-of-azure-data-manager-for-energy/) (Jul 2023) |
| **Equinor** | Chose ADME, aligned with the OSDU standard, to break down data silos across business functions, applications and countries. It uses ADME to automate parts of its data workflows and deliver trusted data to employees. Microsoft already named Equinor as a customer when ADME launched in preview. The GA announcement adds that, as an early adopter, Equinor integrated previously siloed data and gained insights such as **suitable locations for carbon capture and storage**. | [Microsoft customer story](https://www.microsoft.com/en/customers/story/25408-equinor-microsoft-azure-data-manager-for-energy) (Oct 2025); [preview announcement](https://www.microsoft.com/en-us/microsoft-cloud/blog/energy-and-resources/2022/09/20/announcing-microsoft-energy-data-services-enable-your-data-to-do-more-in-the-cloud/) (Sep 2022); [GA announcement](https://www.microsoft.com/en-us/microsoft-cloud/blog/energy-and-resources/2023/07/19/announcing-general-availability-of-azure-data-manager-for-energy/) (Jul 2023) |
| **TotalEnergies** | Chose ADME for secure, real-time global data access with Azure-native compliance controls. Unified data from **10 applications**, cut critical operations **from days to hours**, and onboarded affiliates **in weeks instead of months**. Microsoft's GA announcement called it an early adopter, reporting a deployment that **completes in two hours with a single click**. | [Microsoft customer story](https://www.microsoft.com/en/customers/story/26344-totalenergies-microsoft-azure-data-manager-for-energy/) (Apr 2026); [GA announcement](https://www.microsoft.com/en-us/microsoft-cloud/blog/energy-and-resources/2023/07/19/announcing-general-availability-of-azure-data-manager-for-energy/) (Jul 2023) |

## Common misconceptions

- *"Users will work in ADME."* They work in applications. ADME's value appears
  when those applications and analytics use it.
- *"Loading data is the use case."* Loading is the foundation. Value comes from
  discovery, reuse, application choice and analytics on top.
- *"AI needs a perfect data foundation first."* AI is one of the best tools for
  **building** that foundation: QC, harmonisation, metadata extraction.
- *"Any OSDU application works with any ADME instance."* Check milestone and DDMS
  support per application.

## Go deeper

- [ADME partners](https://learn.microsoft.com/azure/energy-data-services/resources-partner-solutions) (Microsoft Learn)
- [Announcing general availability of ADME](https://www.microsoft.com/en-us/microsoft-cloud/blog/energy-and-resources/2023/07/19/announcing-general-availability-of-azure-data-manager-for-energy/) (Microsoft)
- Customer sources are linked in §5.
