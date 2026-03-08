# DOMAIN-DRIVEN DESIGN DOCUMENT

## StormCore Weather Engine

**Domain Model, Bounded Contexts, and Ubiquitous Language**

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | DRAFT |
| **Author** | Rob-otix AI Ltd |
| **Date** | March 2026 |

---

## Table of Contents

- [1. Ubiquitous Language](#1-ubiquitous-language)
- [2. Bounded Contexts](#2-bounded-contexts)
- [3. Domain Model](#3-domain-model)
- [4. Repository Interfaces](#4-repository-interfaces)
- [5. Domain Services](#5-domain-services)

---

## 1. Ubiquitous Language

The following terms are used consistently across all code, documentation, and communication within the StormCore domain.

| Term | Definition |
|------|-----------|
| ClaimQuery | A request to verify weather conditions at a specific location and time for insurance purposes |
| WeatherObservation | A measured or derived weather data point at a specific location and time |
| RadarFrame | A single processed radar scan representing reflectivity or velocity at a point in time |
| MRMS Product | A processed multi-radar mosaic dataset from NOAA covering CONUS |
| MESH | Maximum Estimated Size of Hail — the primary hail sizing product from MRMS |
| Reflectivity (dBZ) | Radar measure of precipitation intensity; > 55 dBZ typically indicates severe weather |
| LightningStrike | A recorded cloud-to-ground or intra-cloud lightning event with location and timestamp |
| SevereWarning | An NWS-issued alert polygon with type, geometry, valid time window, and source |
| QPE | Quantitative Precipitation Estimate — radar-derived rainfall accumulation |
| NEXRAD | Next Generation Radar — the US national weather radar network of 160+ stations |
| WeatherEvidence | A structured package of weather data and provenance compiled for a specific claim |
| DataProvenance | The source, resolution, timestamp, and confidence of a weather data product |
| IngestLag | The delay between a weather event occurring and data being available in StormCore |
| PointInTimeQuery | A query for weather conditions at a precise lat/lon and UTC timestamp |
| RadarTile | An XYZ map tile rendered from radar data at a specific zoom/x/y and timestamp |
| StrikeCluster | A spatial grouping of lightning strikes within a defined radius and time window |

---

## 2. Bounded Contexts

StormCore is divided into four bounded contexts, each with clear ownership and integration contracts.

### 2.1 Ingest Context

**Responsibility:** Owns the process of acquiring raw weather data from external sources and persisting normalised records to the data store. Has no knowledge of how data is queried or presented.

- **Core concepts:** RawDataPacket, IngestJob, DataSource, ParsedFrame, IngestLag, BackfillJob
- **Anti-corruption layer:** Each external data source (MRMS, NEXRAD, NWS, Blitzortung) has its own adapter that translates external formats into StormCore internal types. No external format leaks beyond the adapter boundary.

### 2.2 Weather Store Context

**Responsibility:** Owns the persistent model of weather observations, radar frames, lightning strikes, and alerts. Provides the canonical query interface for all weather data.

- **Core concepts:** RadarFrame, WeatherObservation, LightningStrike, SevereWarning, MeshHailRecord, QPEAccumulation

### 2.3 Claims Query Context

**Responsibility:** Translates insurance claim verification requests into weather store queries, applies confidence scoring, and compiles WeatherEvidence packages. Is the primary integration point for AI-Claims-LLC.

- **Core concepts:** ClaimQuery, WeatherEvidence, ConfidenceScore, HailVerification, LightningVerification, WindVerification, DataProvenance

### 2.4 Tile Rendering Context

**Responsibility:** Renders weather data into map-compatible tile formats for visualisation. Owned separately from query because tile generation has different performance characteristics and caching strategies.

- **Core concepts:** RadarTile, TileRequest, TileCache, ColourScale, RadarAnimation

---

## 3. Domain Model

### 3.1 Aggregates

#### RadarFrame (Aggregate Root)

```rust
pub struct RadarFrame {
    pub id: RadarFrameId,           // UUID
    pub product: MrmsProduct,       // REFL_QC, MESH, QPE_01H, etc.
    pub valid_time: DateTime<Utc>,
    pub grid: RadarGrid,            // Encoded GRIB2 grid data
    pub resolution_km: f32,
    pub source: DataSource,
    pub ingested_at: DateTime<Utc>,
}

pub enum MrmsProduct {
    ReflectivityQC,        // Base reflectivity, quality-controlled
    MeshHail,              // Maximum Estimated Size of Hail
    QPE1Hour,              // 1-hour precipitation accumulation
    QPE24Hour,             // 24-hour precipitation accumulation
    RotationTracks1Hr,     // Mesocyclone rotation tracks
    PrecipRate,            // Instantaneous precipitation rate
}
```

#### LightningStrike (Aggregate Root)

```rust
pub struct LightningStrike {
    pub id: StrikeId,
    pub location: GeoPoint,         // WGS84 lat/lon
    pub occurred_at: DateTime<Utc>,
    pub strike_type: StrikeType,    // CloudToGround, IntraCloud
    pub peak_current_ka: Option<f32>,
    pub source_network: String,     // "blitzortung", "entln"
    pub ingested_at: DateTime<Utc>,
}
```

#### SevereWarning (Aggregate Root)

```rust
pub struct SevereWarning {
    pub id: WarningId,
    pub warning_type: WarningType,  // TornadoWarning, SevereThunderstorm, FlashFlood, etc.
    pub polygon: GeoPolygon,        // WGS84 polygon
    pub issued_at: DateTime<Utc>,
    pub expires_at: DateTime<Utc>,
    pub nws_id: String,
    pub headline: String,
    pub source: String,             // NWS office identifier
}
```

#### ClaimQuery (Aggregate Root — Claims Context)

```rust
pub struct ClaimQuery {
    pub id: ClaimQueryId,
    pub claim_reference: String,
    pub location: GeoPoint,
    pub event_time: DateTime<Utc>,
    pub radius_km: f32,
    pub query_type: ClaimQueryType,
    pub requested_products: Vec<WeatherProduct>,
    pub created_at: DateTime<Utc>,
}

pub enum ClaimQueryType {
    HailDamage,
    LightningStrike,
    WindDamage,
    FloodDamage,
    TornadoDamage,
    General,
}
```

#### WeatherEvidence (Aggregate Root — Claims Context)

```rust
pub struct WeatherEvidence {
    pub id: EvidenceId,
    pub claim_query_id: ClaimQueryId,
    pub radar_conditions: Option<RadarConditions>,
    pub hail_record: Option<HailRecord>,
    pub lightning_summary: Option<LightningSummary>,
    pub severe_warnings: Vec<ActiveWarning>,
    pub surface_observations: Vec<StationObservation>,
    pub confidence: ConfidenceScore,
    pub provenance: Vec<DataProvenance>,
    pub compiled_at: DateTime<Utc>,
}

pub struct ConfidenceScore {
    pub overall: f32,               // 0.0 - 1.0
    pub radar: Option<f32>,
    pub lightning: Option<f32>,
    pub surface_obs: Option<f32>,
    pub rationale: String,
}
```

### 3.2 Value Objects

| Value Object | Description |
|-------------|-------------|
| GeoPoint | WGS84 lat/lon pair, immutable, validated on construction |
| GeoPolygon | Ordered ring of GeoPoints forming a closed polygon |
| RadarGrid | Encoded raster grid with CRS, bounding box, and resolution |
| DataProvenance | Immutable record of source, resolution, valid time, and ingest time |
| ConfidenceScore | Float 0-1 with rationale string, per-product breakdown |
| HailRecord | MESH value (mm), probability (%), size class (none/small/large/giant) |
| LightningSummary | Strike count, nearest distance (km), max peak current (kA) |
| RadarConditions | Max dBZ, precipitation rate, storm type classification |
| TimeWindow | UTC start/end pair, enforces start < end |

### 3.3 Domain Events

| Event | Triggered When |
|-------|---------------|
| RadarFrameIngested | New MRMS or NEXRAD frame successfully decoded and stored |
| StrikeIngested | Blitzortung strike received, parsed, and written to store |
| WarningIssued | New NWS severe warning parsed and polygon stored |
| WarningExpired | NWS warning expiry time passed or cancellation received |
| ClaimQueryReceived | AI-Claims-LLC submits a new weather verification request |
| WeatherEvidenceCompiled | ClaimQuery processing complete, evidence package ready |
| IngestGapDetected | Expected data product absent beyond tolerable lag threshold |
| BackfillJobCompleted | Historical NEXRAD backfill run finished for a date range |

---

## 4. Repository Interfaces

```rust
pub trait RadarFrameRepository: Send + Sync {
    async fn store(&self, frame: RadarFrame) -> Result<()>;
    async fn query_nearest(
        &self,
        point: GeoPoint,
        time: DateTime<Utc>,
        product: MrmsProduct,
    ) -> Result<Option<RadarFrame>>;
    async fn query_range(
        &self,
        point: GeoPoint,
        window: TimeWindow,
        product: MrmsProduct,
    ) -> Result<Vec<RadarFrame>>;
}

pub trait LightningRepository: Send + Sync {
    async fn store_batch(&self, strikes: Vec<LightningStrike>) -> Result<u64>;
    async fn query_radius(
        &self,
        point: GeoPoint,
        radius_km: f32,
        window: TimeWindow,
    ) -> Result<Vec<LightningStrike>>;
    async fn count_radius(
        &self,
        point: GeoPoint,
        radius_km: f32,
        window: TimeWindow,
    ) -> Result<u64>;
}

pub trait WarningRepository: Send + Sync {
    async fn store(&self, warning: SevereWarning) -> Result<()>;
    async fn query_active_at(
        &self,
        point: GeoPoint,
        time: DateTime<Utc>,
    ) -> Result<Vec<SevereWarning>>;
    async fn expire_warnings(&self, now: DateTime<Utc>) -> Result<u64>;
}

pub trait ClaimQueryRepository: Send + Sync {
    async fn store(&self, query: ClaimQuery) -> Result<()>;
    async fn find_by_id(&self, id: ClaimQueryId) -> Result<Option<ClaimQuery>>;
    async fn store_evidence(&self, evidence: WeatherEvidence) -> Result<()>;
}
```

---

## 5. Domain Services

### 5.1 WeatherEvidenceService

Orchestrates the compilation of a WeatherEvidence package from a ClaimQuery. Queries multiple repositories, applies confidence scoring, and assembles provenance records.

```rust
pub struct WeatherEvidenceService {
    radar: Arc<dyn RadarFrameRepository>,
    lightning: Arc<dyn LightningRepository>,
    warnings: Arc<dyn WarningRepository>,
    confidence: ConfidenceScoringService,
}

impl WeatherEvidenceService {
    pub async fn compile(&self, query: &ClaimQuery) -> Result<WeatherEvidence> {
        let window = TimeWindow::around(query.event_time, Duration::hours(2));
        let (radar, strikes, warnings) = tokio::join!(
            self.radar.query_nearest(query.location, query.event_time, MrmsProduct::ReflectivityQC),
            self.lightning.query_radius(query.location, query.radius_km, window),
            self.warnings.query_active_at(query.location, query.event_time),
        );
        // ... compile and score
    }
}
```

### 5.2 ConfidenceScoringService

Applies domain rules to assign confidence scores to compiled evidence. Rules encode insurance domain knowledge about what constitutes strong vs weak evidence for each claim type.

**Examples:**
- MESH > 25mm + active severe warning = **HIGH** hail confidence
- dBZ > 55 within 5km + active TOR warning = **HIGH** tornado confidence

### 5.3 TileRenderingService

Converts radar grid data into PNG tiles for a given XYZ coordinate and timestamp. Maintains a tile cache keyed by `(product, z, x, y, timestamp_bucket)`. Applies NWS-standard colour scales.
