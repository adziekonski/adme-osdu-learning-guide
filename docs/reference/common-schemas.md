# Common Schemas and Their Fields

> **Facts as of:** October 2026. Versions are those of **OSDU M26**, the
> milestone ADME runs; fields added in newer OSDU versions are marked. Your
> instance's Schema service shows the exact list.

> **Reference:** look up fields here; don't read it in order.

Every field of the 12 schemas you meet most often.
[Fundamentals 04](../fundamentals/04-entities-and-files.md) explains what these
entities are and how they connect.

## How to read this page

- **The schema makes you fill almost nothing.** Every record needs `kind`,
  `acl` and `legal`. Inside `data`, only **WellboreTrajectory**,
  **ProductionValues** and **File.Generic** have required fields.
- **"Not required" isn't "not needed".**
  - A Wellbore without `WellID` is an orphan.
  - A Well without a usable location never appears in a map search: it needs
    `Wgs84Coordinates`, or original coordinates with a CRS the indexer can
    convert.
  - A WellLog without `Datasets` has no data behind it.
- **Three tiers per schema:**
  - **Required:** the schema rejects the record without it.
  - **Recommended:** a practical minimum for a useful record. This is our
    recommendation, not an OSDU rule.
  - **Other optional:** every other field the schema defines.
- **Notation:**
  - **(ref: X)** means a value from reference list X;
  - **→ X** means a link to another record of type X.

  Both hold full OSDU IDs ending with `:`, not free text.
- **Fill what the source really has.** A typical migration fills 10–20 fields
  per entity. Never invent values to make a record look complete.

```text
Field ◄─GeoContexts── Well ◄─WellID── Wellbore ◄─WellboreID── WellLog / WellboreTrajectory / WellboreMarkerSet
                                                                   │ Datasets
Document ── Datasets ──► File.Generic ◄────────────────────────────┘
WorkProduct ── Components ──► any WPC
ProductionValues ── ReportingEntityID ──► Field, Well, Wellbore, Reservoir…
```

## 1. Shared blocks: learn these once

Most fields come from a few shared blocks. The tables in §2–§4 list only each
schema's **own** fields, plus the shared ones we recommend.

| Block | Used by | All fields |
|---|---|---|
| **Envelope** (outside `data`) | Every record | **Required:** `kind`, `acl`, `legal`. Also: `id`, `version`, `tags`, `ancestry`, `meta`, plus system fields (`createUser`, `createTime`…) |
| **Common resources** | Every record | `Source`, `ExistenceKind`, `ResourceSecurityClassification`, `ResourceCurationStatus`, `ResourceLifecycleStatus`, `ResourceHomeRegionID`, `ResourceHostRegionIDs`, `TechnicalAssuranceID` |
| **Master** | Field, Well, Wellbore, Organisation | `NameAliases`, `SpatialLocation`, `GeoContexts`, `VersionCreationReason`, `TechnicalAssurances`, `TechnicalAssuranceTypeID` |
| **Facility** | Well, Wellbore | `FacilityName`, `FacilityID`, `FacilityTypeID`, `FacilityDescription`, `FacilityNameAliases`, `FacilityOperators`, `InitialOperatorID`, `CurrentOperatorID`, `DataSourceOrganisationID`, `OperatingEnvironmentID`, `FacilityStates`, `FacilityEvents`, `FacilitySpecifications` |
| **WPC** | All work-product-components | `Name`, `Description`, `Datasets`, `DDMSDatasets`, `Artefacts`, `NameAliases`, `CreationDateTime`, `Tags`, `SpatialPoint`, `SpatialArea`, `GeoContexts`, `SubmitterName`, `BusinessActivities`, `AuthorIDs`, `LineageAssertions`, `TechnicalAssurances`, `IsExtendedLoad`, `IsDiscoverable` |
| **Reference value** | Every reference-data type | `Code`, `Name`, `Description`, `NameAlias`, `ID`, `InactiveIndicator`, `AttributionAuthority`, `AttributionPublication`, `AttributionRevision`, `CommitDate` |

Every schema also has **`ExtensionProperties`**, the bucket for anything else.
It's stored, but not searchable.

## 2. Master data

### [Field](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/master-data/Field.1.1.0.md) 1.1.0

| Tier | Fields |
|---|---|
| Required | — |
| Recommended | `FieldName`, `NameAliases`, `SpatialLocation` (outline), `Source` |
| Other optional | `FieldID`, `FieldNameAliases`, `FieldDescription`, `EffectiveDate`, `TerminationDate`, + master and common blocks |

### [Well](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/master-data/Well.1.4.0.md) 1.4.0

| Tier | Fields |
|---|---|
| Required | — |
| Recommended | **Identity:** `FacilityName`, `NameAliases`, `Source`. **Location and context:** `SpatialLocation`, `GeoContexts` (the field). **Operator:** `CurrentOperatorID` (→ Organisation). **Depth reference:** `VerticalMeasurements`, `DefaultVerticalMeasurementID`. **Status:** `StatusSummaryID` (ref: WellStatusSummary). **Events:** `FacilityEvents` (for example the spud date) |
| Other optional | **Status facets** (see below): `BusinessIntentionID` (ref: WellBusinessIntention), `RoleID` (ref: WellRole), `ConditionID` (ref: WellCondition), `OutcomeID` (ref: WellBusinessIntentionOutcome), `InterestTypeID` (ref: WellInterestType). **Historical interest:** `HistoricalInterests`, `WasBusinessInterestFinancialOperated`, `WasBusinessInterestFinancialNonOperated`, `WasBusinessInterestObligatory`, `WasBusinessInterestTechnical`. **Location detail:** `DefaultVerticalCRSID` (ref: CoordinateReferenceSystem), `LocationUncertainty`, and `WellReferencePoint` (added in 1.5.0, OSDU M27; not on ADME yet). **Offshore structure:** `WellSiteStructureID` (→ WellSiteStructure), `SlotID`. **Shared blocks:** the rest of the Facility block, plus the master and common blocks |

**The status facets in plain words:**
- **BusinessIntention:** why it was drilled (explore, appraise, develop).
- **Role:** what it does now (producer, injector, observation).
- **Condition:** its operating state (active, inactive, abandoned).
- **Outcome:** whether the intention was achieved (achieved or unachieved).
- **InterestType:** your company's interest in the well (operated,
  non-operated joint venture, competitor…).
- **StatusSummary:** one combined label.

Map to the separate facets only if the source tracks status that way. Otherwise
use `StatusSummaryID` alone.

### [Wellbore](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/master-data/Wellbore.1.5.1.md) 1.5.1

| Tier | Fields |
|---|---|
| Required | — (in practice, **`WellID`**) |
| Recommended | **Identity and parent:** `FacilityName`, `WellID` (→ Well), `NameAliases`. **Depth reference:** `VerticalMeasurements`, `DefaultVerticalMeasurementID`. **Trajectory:** `TrajectoryTypeID` (ref: WellboreTrajectoryType), `DefinitiveTrajectoryID` (→ WellboreTrajectory). **Bottom-hole location:** `ProjectedBottomHoleLocation`, `GeographicBottomHoleLocation`. **Status and product:** `StatusSummaryID`, `PrimaryProductTypeID` (ref: WellProductType) |
| Other optional | **Drilling:** `SequenceNumber`, `KickOffWellbore` (→ Wellbore, the parent of a sidetrack), `DrillingReasons`, `WellboreReasonID` (ref: WellboreReason), `WellboreTrajectoryTypeID`. **Geology:** `TargetFormation` (ref: GeologicalFormation), `FormationNameAtTotalDepth`, `PrimaryMaterialID` (ref: MaterialType). **Products:** `SecondaryProductTypeID`, `TertiaryProductTypeID`, `ShowProductTypeID`, `FluidDirectionID` (ref: WellFluidDirection). **Status facets (as for Well):** `BusinessIntentionID`, `RoleID`, `ConditionID`, `OutcomeID`, `InterestTypeID`. **Historical interest:** `HistoricalInterests`, `WasBusinessInterestFinancialOperated`, `WasBusinessInterestFinancialNonOperated`, `WasBusinessInterestObligatory`, `WasBusinessInterestTechnical`. **Cost:** `WellboreCosts`. **Shared blocks:** the Facility, master and common blocks |

### [Organisation](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/master-data/Organisation.1.2.0.md) 1.2.0

| Tier | Fields |
|---|---|
| Required | — |
| Recommended | `OrganisationName`, `OrganisationTypeID` (ref: OrganisationType), `NameAliases` |
| Other optional | `OrganisationID`, `OrganisationNameAliases`, `ParentOrganisationID` (→ Organisation), `InternalOrganisationIndicator`, `OrganisationDescription`, `OrganisationPurposeDescription`, `EffectiveDate`, `TerminationDate`, + master and common blocks |

## 3. Work-product-components

### [WellLog](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/work-product-component/WellLog.1.5.0.md) 1.5.0

| Tier | Fields |
|---|---|
| Required | — |
| Recommended | **Links:** `WellboreID` (→ Wellbore), `Name`, `Datasets` (the file) or `DDMSDatasets` (curves in the Wellbore DDMS). **Depth range:** `TopMeasuredDepth`, `BottomMeasuredDepth`, `ReferenceCurveID`, `SamplingDomainTypeID` (ref: WellLogSamplingDomainType). **Curves:** `Curves[]`. **Acquisition:** `ServiceCompanyID` (→ Organisation), `LogServiceDateInterval` |
| Other optional | **Classification:** `WellLogTypeID` (ref: LogType), `WellLogClassID` (ref: WellLogClass). **Log run:** `LogSource`, `LogActivity`, `LogRun`, `LogVersion`, `LoggingService`, `PassNumber`, `LoggingDirection`, `ActivityType`, `ToolStringDescription`, `ConveyanceMethodID` (ref: ConveyanceMethod), `CompanyID` (→ Organisation), `WellLogAcquisitionDetails`. **Hole and fluid:** `HoleTypeLogging`, `DrillingFluidProperty`, `WellboreFluidTypeID` (ref: WellboreFluidType). **Depth reference:** `VerticalMeasurementID`, `VerticalMeasurement`, `SeismicReferenceElevation`, `ZeroTime`. **Sampling:** `SamplingInterval`, `SamplingStart`, `SamplingStop`, `IsRegular`, `NullValue`, `CandidateReferenceCurveIDs`. **File origin:** `FrameIdentifier`, `LogicalFileIdentifier`. **Notes:** `LogRemark`, `Remarks`. **Shared blocks:** the WPC and common blocks |

**Each item in `Curves[]`:**
- **Recommended:** `Mnemonic` (for example `GR`), `CurveUnit`,
  `LogCurveFamilyID`, `TopDepth`, `BaseDepth`, `DepthUnit`.
- **Also defined:**
  - identity and version: `CurveID`, `CurveVersion`, `DateStamp`,
    `CurveDescription`;
  - quality and interpretation: `CurveQuality`, `InterpreterName`,
    `IsProcessed`;
  - sampling: `IsRegularlySampled`, `SamplingStart`, `SamplingStop`,
    `NullValue`, `DepthCoding`, `Interpolate`;
  - classification: `LogCurveTypeID`, `LogCurveMainFamilyID`,
    `LogCurveBusinessValueID`, `CurveSampleTypeID`, `NumberOfColumns`.

### [WellboreTrajectory](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/work-product-component/WellboreTrajectory.1.4.0.md) 1.4.0

| Tier | Fields |
|---|---|
| **Required** | **`WellboreID`, `TopDepthMeasuredDepth`, `BaseDepthMeasuredDepth`, `VerticalMeasurement`** (the depth reference) |
| Recommended | `Name`, `Datasets`, `AzimuthReferenceType` (ref; true, grid or magnetic north), `CalculationMethodType` (ref; for example minimum curvature), `ProjectedCRSID` (ref: CoordinateReferenceSystem), `IsDefinitive` |
| Other optional | **Survey:** `SurveyType`, `SurveyToolTypeID` (ref), `SurveyReferenceIdentifier`, `SurveyVersion`, `AcquisitionDate`, `AcquisitionRemark`, `ServiceCompanyID`, `CompanyID`, `StartDateTime`, `EndDateTime`, `ActiveIndicator`, `IsFinal`, `IsMemory`. **Depths and tie-in:** `TieMeasuredDepth`, `TieTrueVerticalDepth`, `ExtrapolatedMeasuredDepth`, `ExtrapolatedMeasuredDepthRemark`, `GeographicCRSID`. **Station data:** `AvailableTrajectoryStationProperties`. **Processing:** `AppliedOperations`, `AppliedOperationsDateTime`, `AppliedOperationsUser`, `AppliedOperationsRemarks`. **Geometry:** `SurfaceGridConvergence`, `SurfaceScaleFactor`, `Tortuosity`, `VerticalSectionAzimuth`, `VerticalSectionOriginNS`, `VerticalSectionOriginEW`. **Magnetic and gravity models:** `GeoMagneticModelID`, `MagneticFieldCalculationDate`, `MagneticDeclinationUsed`, `MagneticDeclinationUncertainty`, `MagneticDipAngleReference`, `MagneticDipAngleUncertainty`, `TotalMagneticFieldStrength`, `TotalMagneticFieldStrengthUncertainty`, `GravityModelID`, `TotalGravityFieldStrength`, `TotalGravityFieldStrengthUncertainty`. **Shared blocks:** the WPC and common blocks |

The survey stations themselves live in the file or the DDMS.
`AvailableTrajectoryStationProperties` only describes which columns exist.

### [WellboreMarkerSet](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/work-product-component/WellboreMarkerSet.1.5.1.md) 1.5.1

| Tier | Fields |
|---|---|
| Required | — |
| Recommended | `WellboreID`, `Name`, `VerticalMeasurement`, `Markers[]` |
| Other optional | `StratigraphicColumnID` (→ StratigraphicColumn), `StratigraphicColumnRankInterpretationID`, `AvailableMarkerProperties`, `MarkerProperties`, `TimeSeries`, + WPC and common blocks |

**Each item in `Markers[]`:**
- **Recommended:** `MarkerName`, `MarkerMeasuredDepth`, `MarkerTypeID` or
  `FeatureName`.
- **Also defined:**
  - identity: `MarkerID`, `InterpretationID`, `MarkerObservationNumber`,
    `MarkerInterpreter`, `MarkerDate`;
  - depth: `MarkerSubSeaVerticalDepth`, `PositiveVerticalDelta`,
    `NegativeVerticalDelta`;
  - geology: `FeatureTypeID`, `SurfaceDipAngle`, `SurfaceDipAzimuth`,
    `GeologicalAge`, `Missing`.

### [Document](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/work-product-component/Document.2.0.0.md) 2.0.0

| Tier | Fields |
|---|---|
| Required | — |
| Recommended | `Name`, `Datasets`, `TaxonomyIDs` (ref: DocumentTaxonomy), `DatePublished`, `Summary`, `Keywords` |
| Other optional | **Content:** `SubTitle`, `Subject`, `NumberOfPages`, `Languages`, `TableOfContents`. **Dates and retention:** `DateModified`, `DateIssued`, `DateReviewDue`, `RetentionPeriod`. **Organisations:** `ProducedByOrganisationID`, `ProducedForOrganisationID` (→ Organisation). **Shared blocks:** the WPC and common blocks |

Document has **no dedicated Well or Wellbore field**. Agree a convention for
linking reports to wells before loading, for example `GeoContexts`,
`LineageAssertions` or tags.

### [SeismicTraceData](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/work-product-component/SeismicTraceData.1.6.0.md) 1.6.0

| Tier | Fields |
|---|---|
| Required | — |
| Recommended | **Links:** `Name`, `Datasets`, `PrincipalAcquisitionProjectID` (→ SeismicAcquisitionSurvey), `ProcessingProjectID` (→ SeismicProcessingProject), `BinGridID` (→ SeismicBinGrid). **Geometry:** `SeismicDomainTypeID` (ref; time or depth), `SampleInterval`, `SampleCount`, `InlineMin` / `InlineMax`, `CrosslineMin` / `CrosslineMax`, `TraceDomainUOM` (ref: UnitOfMeasure). **Footprint:** `SpatialArea` |
| Other optional | **Geometry:** `LiveTraceOutline`, `HorizontalCRSID`, `SeismicLineGeometryID`, `Seismic2DName`, `TraceCount`, `TraceLength`, `InlineIncrement`, `CrosslineIncrement`, `FirstShotPoint`, `LastShotPoint`, `FirstCMP`, `LastCMP`, `StartTime`, `EndTime`, `StartDepth`, `EndDepth`, `VerticalDatumOffset`, `VerticalMeasurementTypeID`, `ReplacementVelocity`, `ShiftApplied`, `CoveragePercent`, `AdditionalTraceHeaderRanges`⁺. **Processing:** `SeismicTraceDataDimensionalityTypeID`, `SeismicMigrationTypeID`, `SeismicStackingTypeID`, `SeismicAttributeTypeID`, `SeismicFilteringTypeID`, `SeismicProcessingStageTypeID`, `ProcessingParameters`, `NotionalSeismicProcessingProductID`, `VelocityModelID`⁺, `Difference`. **Signal:** `Phase`, `Polarity`, `SeismicPhaseID`, `SeismicPolarityID`, `Precision`, `RangeAmplitudeMin`, `RangeAmplitudeMax`, `SeismicWaveTypeID`. **Pre-stack:** `IsPreStack`, `GatherTypeID`, `SortOrderID`, `StackAngleRangeMin` / `StackAngleRangeMax`, `StackAzimuthRangeMin` / `StackAzimuthRangeMax`, `StackOffsetRangeMin` / `StackOffsetRangeMax`, `StackAngleRanges`, `StackAzimuthRanges`, `StackOffsetRanges`. **Time-lapse (4D):** `TimeLapse`, `TimeLapseAssociations`. **Interpretation:** `Preferred2DInterpretationSetID`, `Preferred3DInterpretationSetID`. **Other:** `TextualFileHeader`, `TraceRelationFileID`, `DomainTypeID`⁺, `Contracts`⁺ (→ Agreement). **Shared blocks:** the WPC and common blocks |

⁺ Added in version 1.8.0 (OSDU M27); not on ADME yet.

### [ProductionValues](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/work-product-component/ProductionValues.1.1.1.md) 1.1.1

This is the version on ADME (OSDU M26). OSDU's version 2.0.0 arrived with M27;
see the note below the table.

| Tier | Fields |
|---|---|
| **Required** | **`ReportingEntityID`** (→ Field, Reservoir, ReservoirSegment, Well, Wellbore, IsolatedInterval or WellboreOpening), **`StartDateTime`**, **`NominalPeriodIDs`** (ref: ReportingPeriod, for example day or month), **`PropertyIDs`** (ref: ProductionPropertyType), **`ProductIDs`** (ref: ProductKind) |
| Recommended | `Name`, `EndDateTime`, `Datasets` or `DDMSDatasets` (where the values are), `QuantityMethodIDs` (ref: QuantityMethod; how the values were obtained), `VolumeFlowMeasurementTypeIDs` (ref: VolumeFlowMeasurementType; direction of flow into or out of the entity), `MeasurementConditions` (pressure and temperature the values refer to) |
| Other optional | `DurationContextIDs` (ref: DurationContext), `DispositionIDs` (ref: DispositionType), `ValueContexts` (links to the events behind the values, such as downtime), `AmendReasonID` (ref: AmendReason), `ReservoirModelScenarioID` (for forecasts), `ObservationInterval`, `ProductionValuesObservationDescription` (a table that describes the value columns, or holds the values themselves), + WPC and common blocks |

**Where the numbers are:**
- The record says **what** is reported: which well or field, from when, for
  which period, and which properties and products.
- The values themselves are usually in an attached **file** (`Datasets`) or a
  **DDMS** (`DDMSDatasets`).
- `ProductionValuesObservationDescription` describes their columns, or holds a
  small table of values inside the record.

**Version 2.0.0** (OSDU M27, not yet on ADME):
- Lists become single values, for example `NominalPeriodIDs` →
  `NominalPeriodID`.
- A new **`ProductionMetricValues[]`** has one entry per metric: property,
  product, unit, and where its values are (a dataset file and column, or a DDMS
  dataset).
- **`Remarks`** and **`ProductionMetricValues`** become required.
- OSDU publishes a
  [migration guide](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/Guides/MigrationGuides/M27.0/work-product-component--ProductionValues.1.1.1.md).

## 4. Dataset and work product

### [File.Generic](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/dataset/File.Generic.1.1.0.md) 1.1.0

| Tier | Fields |
|---|---|
| **Required** | **`DatasetProperties`**, which in practice contains **`FileSourceInfo.FileSource`**: the value the File service returns after upload |
| Recommended | `Name`, `Checksum`, `TotalSize`, `EncodingFormatTypeID` (ref: EncodingFormatType) |
| Other optional | **Dataset:** `Description`, `SchemaFormatTypeID`, `Endian`. **Inside `FileSourceInfo`:** `Name`, `FileSize`, `EncodingFormatTypeID`, `Checksum`, `ChecksumAlgorithm`, `PreloadFilePath`, `PreloadFileCreateUser`, `PreloadFileCreateDate`, `PreloadFileModifyUser`, `PreloadFileModifyDate`. **Shared blocks:** the common block |

### [WorkProduct](https://community.opengroup.org/osdu/data/data-definitions/-/blob/master/E-R/work-product/WorkProduct.1.0.0.md) 1.0.0

| Tier | Fields |
|---|---|
| Required | — (in a manifest it's mandatory whenever WPCs are loaded) |
| Recommended | `Name`, `Components[]` (the IDs of its WPCs) |
| Other optional | `Description`, `CreationDateTime`, `Tags`, `SpatialPoint`, `SpatialArea`, `SubmitterName`, `BusinessActivities`, `AuthorIDs`, `LineageAssertions`, `Annotations`, `IsExtendedLoad`, `IsDiscoverable`, + common block |

## Not listed here

- **Sub-fields of most nested objects.** For example `SpatialLocation`,
  `VerticalMeasurements`, `FacilityEvents` and `GeoContexts`: follow the schema
  link in each heading. This page expands only curves, markers and file source
  information.
- **Other entity types.** Drilling, samples, reservoir and production types are
  in the [OSDU schema catalogue](https://community.opengroup.org/osdu/data/data-definitions/-/tree/master/E-R).
