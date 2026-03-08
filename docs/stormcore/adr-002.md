# ARCHITECTURE REQUIREMENTS DOCUMENT

## StormCore Weather Engine

**System Architecture, Technology Stack, and Infrastructure Design**

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | DRAFT |
| **Author** | Rob-otix AI Ltd |
| **Date** | March 2026 |

---

## Table of Contents

- [1. Architecture Overview](#1-architecture-overview)
- [2. Repository Structure](#2-repository-structure)
- [3. Component Architecture](#3-component-architecture)
- [4. Data Architecture](#4-data-architecture)
- [5. API Design](#5-api-design)
- [6. TypeScript SDK](#6-typescript-sdk)
- [7. Infrastructure](#7-infrastructure)
- [8. Tile Rendering Architecture](#8-tile-rendering-architecture)
- [9. Observability and Monitoring](#9-observability-and-monitoring)
- [10. Security](#10-security)
- [11. Build Order and Milestones](#11-build-order-and-milestones)

---

## 1. Architecture Overview

StormCore is a Rust-core, TypeScript-edge weather data engine. The architecture separates concerns along performance boundaries: Rust handles all CPU-intensive ingest, decoding, spatial processing, and tile rendering; TypeScript handles the API surface, SDK, and integration with AI-Claims-LLC.

### Architecture Principles

- **Rust** for anything that touches raw data, binary formats, or needs sub-millisecond processing
- **TypeScript** for the developer-facing surface — clean, typed, familiar in the claims-monorepo context
- **Event-driven ingest** — S3 notifications and WebSocket feeds drive data flow, not polling where avoidable
- **TimescaleDB** as the primary store — time-series first, PostGIS spatial queries, compression for cold data
- **Tile cache aggressively** — radar tiles at a given time never change, cache indefinitely
- **Observability first** — every ingest job emits lag metrics, every query emits latency and cache hit/miss

---

## 2. Repository Structure

```
stormcore/
├── Cargo.toml                    # Workspace root
├── crates/
│   ├── mrms-ingest/              # MRMS GRIB2 S3 polling + decode
│   ├── nexrad-decode/            # NEXRAD Level II binary parse
│   ├── radar-composite/          # Grid compositing + PNG tile render
│   ├── lightning-ingest/         # Blitzortung WebSocket consumer
│   ├── alerts-ingest/            # NWS CAP alert polling
│   ├── surface-ingest/           # ASOS/METAR feed parser
│   ├── weather-store/            # TimescaleDB repositories
│   ├── claims-engine/            # WeatherEvidence compilation
│   ├── tile-server/              # Axum tile serving + cache
│   └── stormcore-api/            # Axum REST API (main binary)
├── packages/
│   ├── stormcore-client/         # TypeScript SDK
│   └── stormcore-types/          # Shared TypeScript types
├── migrations/                   # TimescaleDB SQL migrations
├── docker/
│   ├── docker-compose.yml        # Local dev stack
│   └── Dockerfile.stormcore      # Multi-stage Rust build
└── scripts/
    ├── backfill.sh               # NEXRAD historical backfill
    └── validate_ingest.sh        # Data quality checks
```

---

## 3. Component Architecture

### 3.1 Ingest Pipeline

```
// MRMS ingest flow
S3 noaa-mrms-pds bucket
  → S3 event notification (SQS or polling)
  → mrms-ingest crate
      ├── Download GRIB2 file (bytes)
      ├── Decompress (flate2)
      ├── Decode GRIB2 grid (eccodes-rs or cfgrib)
      ├── Project to WGS84 (proj crate)
      ├── Write RadarFrame to TimescaleDB
      └── Emit RadarFrameIngested domain event

// NEXRAD ingest flow
S3 noaa-nexrad-level2 bucket
  → poll for new objects (2-min cadence per station)
  → nexrad-decode crate
      ├── nexrad-decode crate (community, Level II format)
      ├── Extract reflectivity + velocity sweeps
      ├── Interpolate to uniform grid (radar-composite)
      └── Write to TimescaleDB

// Lightning ingest flow
Blitzortung WebSocket (ws://ws.blitzortung.org)
  → lightning-ingest crate (tokio-tungstenite)
      ├── Parse JSON strike messages
      ├── Batch accumulate (100ms windows)
      └── Bulk insert to TimescaleDB

// NWS Alerts flow
https://api.weather.gov/alerts/active
  → alerts-ingest crate (reqwest, 60s poll)
      ├── Parse CAP GeoJSON
      ├── Decode polygon geometries (geo crate)
      ├── Upsert SevereWarning records
      └── Mark expired warnings
```

### 3.2 Cargo.toml Dependencies by Crate

| Crate | Key Dependencies |
|-------|-----------------|
| mrms-ingest | aws-sdk-s3, tokio, flate2, eccodes-rs, sqlx, prost |
| nexrad-decode | nexrad-decode (community), tokio, aws-sdk-s3, sqlx |
| radar-composite | image, colorgrad, geo, proj, ndarray, rayon |
| lightning-ingest | tokio-tungstenite, serde_json, sqlx, tokio |
| alerts-ingest | reqwest, serde, geo, geojson, sqlx, tokio |
| weather-store | sqlx (postgres + runtime-tokio), geo, chrono, uuid |
| claims-engine | weather-store, tokio, serde, chrono, geo |
| tile-server | axum, image, colorgrad, redis, tower-http, tokio |
| stormcore-api | axum, tower, tokio, serde, tracing, weather-store, claims-engine |

---

## 4. Data Architecture

### 4.1 TimescaleDB Schema

```sql
-- Hypertable: radar frames (partitioned by valid_time)
CREATE TABLE radar_frames (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product     TEXT NOT NULL,               -- 'REFL_QC', 'MESH', 'QPE_01H', etc.
    valid_time  TIMESTAMPTZ NOT NULL,
    grid_data   BYTEA NOT NULL,              -- compressed grid (zstd)
    bbox        GEOMETRY(Polygon, 4326) NOT NULL,
    resolution_km REAL NOT NULL,
    source      TEXT NOT NULL,
    ingested_at TIMESTAMPTZ DEFAULT NOW()
);
SELECT create_hypertable('radar_frames', 'valid_time');
CREATE INDEX ON radar_frames (product, valid_time DESC);

-- Hypertable: lightning strikes
CREATE TABLE lightning_strikes (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location    GEOMETRY(Point, 4326) NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    strike_type TEXT,
    peak_current_ka REAL,
    source_network  TEXT NOT NULL,
    ingested_at TIMESTAMPTZ DEFAULT NOW()
);
SELECT create_hypertable('lightning_strikes', 'occurred_at');
CREATE INDEX ON lightning_strikes USING GIST (location);
CREATE INDEX ON lightning_strikes (occurred_at DESC);

-- Hypertable: severe warnings
CREATE TABLE severe_warnings (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nws_id      TEXT UNIQUE NOT NULL,
    warning_type TEXT NOT NULL,
    polygon     GEOMETRY(Polygon, 4326) NOT NULL,
    issued_at   TIMESTAMPTZ NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,
    headline    TEXT,
    source      TEXT
);
CREATE INDEX ON severe_warnings USING GIST (polygon);
CREATE INDEX ON severe_warnings (issued_at, expires_at);

-- Compression policies (TimescaleDB)
SELECT add_compression_policy('radar_frames', INTERVAL '7 days');
SELECT add_compression_policy('lightning_strikes', INTERVAL '30 days');
```

### 4.2 Key Spatial Queries

```sql
-- Point-in-time radar query (< 5ms with index)
SELECT * FROM radar_frames
WHERE product = 'REFL_QC'
  AND valid_time BETWEEN $1 - INTERVAL '5 minutes' AND $1 + INTERVAL '5 minutes'
  AND ST_Contains(bbox, ST_SetSRID(ST_MakePoint($lon, $lat), 4326))
ORDER BY ABS(EXTRACT(EPOCH FROM (valid_time - $1)))
LIMIT 1;

-- Lightning radius query
SELECT * FROM lightning_strikes
WHERE ST_DWithin(location::geography, ST_SetSRID(ST_MakePoint($lon, $lat), 4326)::geography, $radius_m)
  AND occurred_at BETWEEN $start AND $end;

-- Active warning at point and time
SELECT * FROM severe_warnings
WHERE ST_Contains(polygon, ST_SetSRID(ST_MakePoint($lon, $lat), 4326))
  AND issued_at <= $time
  AND expires_at >= $time;
```

---

## 5. API Design

### 5.1 REST Endpoints (Axum)

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| POST | /v1/claims/query | Full weather evidence for claim | WeatherEvidence JSON |
| GET | /v1/weather/point | Current/historical point conditions | WeatherObservation JSON |
| GET | /v1/hail/query | MESH hail record at point + time | HailRecord JSON |
| GET | /v1/lightning/query | Lightning summary in radius + window | LightningSummary JSON |
| GET | /v1/alerts/active | Active NWS warnings at point | SevereWarning[] JSON |
| GET | /tiles/radar/:z/:x/:y | Radar tile (current) | PNG image |
| GET | /tiles/radar/:z/:x/:y?t= | Radar tile at timestamp | PNG image |
| GET | /tiles/lightning/:z/:x/:y | Lightning density tile | PNG image |
| GET | /health | Ingest lag + system health | HealthReport JSON |

### 5.2 Claims Query Request/Response

```json
// POST /v1/claims/query
{
  "claimReference": "CLM-2024-00441",
  "lat": 35.4676,
  "lon": -97.5164,
  "timestamp": "2024-06-15T19:32:00Z",
  "radiusKm": 5,
  "queryType": "HailDamage"
}
```

```json
// Response
{
  "evidenceId": "ev_01j9x...",
  "claimReference": "CLM-2024-00441",
  "radarConditions": {
    "maxReflectivityDbz": 68.2,
    "precipRateMmHr": 42.1,
    "stormType": "SEVERE_THUNDERSTORM"
  },
  "hailRecord": {
    "meshMm": 38.4,
    "probability": 0.94,
    "sizeClass": "LARGE",
    "sizeClassLabel": "Golf ball (38mm)"
  },
  "lightningSummary": {
    "strikeCount": 47,
    "nearestDistanceKm": 0.8,
    "windowMinutes": 120
  },
  "severeWarnings": [
    {
      "type": "SevereThunderstormWarning",
      "headline": "SEVERE THUNDERSTORM WARNING including golf ball hail",
      "issuedAt": "2024-06-15T19:15:00Z",
      "expiresAt": "2024-06-15T20:00:00Z"
    }
  ],
  "confidence": {
    "overall": 0.97,
    "radar": 0.99,
    "lightning": 0.95,
    "rationale": "MESH 38mm exceeds large hail threshold; active SVR warning confirmed hail; 68dBZ reflectivity consistent with hail-producing storm"
  },
  "provenance": [
    { "source": "NOAA MRMS", "product": "MESH", "validTime": "2024-06-15T19:30:00Z", "resolutionKm": 1.0 },
    { "source": "NWS OUN", "product": "SevereWarning", "issuedAt": "2024-06-15T19:15:00Z" }
  ],
  "compiledAt": "2026-03-08T14:22:01Z"
}
```

---

## 6. TypeScript SDK

```typescript
// packages/stormcore-client/src/index.ts
import type { ClaimQuery, WeatherEvidence, HealthReport } from '@stormcore/types';

export class StormCoreClient {
  private baseUrl: string;
  private apiKey: string;

  constructor(config: { baseUrl: string; apiKey: string }) {
    this.baseUrl = config.baseUrl;
    this.apiKey = config.apiKey;
  }

  async queryClaim(query: ClaimQuery): Promise<WeatherEvidence> {
    const res = await fetch(`${this.baseUrl}/v1/claims/query`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-API-Key': this.apiKey },
      body: JSON.stringify(query)
    });
    if (!res.ok) throw new StormCoreError(await res.text(), res.status);
    return res.json();
  }

  radarTileUrl(z: number, x: number, y: number, timestamp?: Date): string {
    const t = timestamp ? `?t=${timestamp.toISOString()}` : '';
    return `${this.baseUrl}/tiles/radar/${z}/${x}/${y}${t}`;
  }

  async health(): Promise<HealthReport> {
    const res = await fetch(`${this.baseUrl}/health`);
    return res.json();
  }
}

// Usage in claims-monorepo
const stormcore = new StormCoreClient({
  baseUrl: process.env.STORMCORE_URL!,
  apiKey: process.env.STORMCORE_API_KEY!
});

const evidence = await stormcore.queryClaim({
  claimReference: claim.id,
  lat: claim.location.lat,
  lon: claim.location.lon,
  timestamp: claim.incidentDate,
  radiusKm: 5,
  queryType: 'HailDamage'
});

if (evidence.confidence.overall > 0.85) {
  await flagClaimWeatherVerified(claim.id, evidence);
}
```

---

## 7. Infrastructure

### 7.1 Local Development

```yaml
# docker-compose.yml (key services)
services:
  timescaledb:
    image: timescale/timescaledb-ha:pg16-latest
    environment:
      POSTGRES_DB: stormcore
      POSTGRES_PASSWORD: stormcore
    volumes:
      - timescale_data:/home/postgres/pgdata/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  stormcore:
    build:
      context: .
      dockerfile: docker/Dockerfile.stormcore
    environment:
      DATABASE_URL: postgresql://postgres:stormcore@timescaledb/stormcore
      REDIS_URL: redis://redis:6379
      AWS_DEFAULT_REGION: us-east-1
    ports:
      - "3000:3000"
    depends_on:
      - timescaledb
      - redis
```

### 7.2 Production Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary data store | TimescaleDB on RDS or self-hosted | Time-series + PostGIS in one |
| Tile cache | Redis (ElastiCache) | Sub-ms tile lookups, TTL management |
| Ingest runtime | Rust binary on ECS Fargate | Always-on, low memory |
| API runtime | Rust/Axum on ECS Fargate | Single binary, minimal overhead |
| S3 access | AWS SDK (open data buckets) | MRMS + NEXRAD, no cost |
| Observability | Prometheus + Grafana | Ingest lag, query latency, cache hit rate |
| Secrets | AWS Secrets Manager | DB credentials, API keys |
| IaC | Terraform or CDK | Reproducible infra |

### 7.3 Dockerfile (Multi-stage Rust)

```dockerfile
FROM rust:1.77 AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY crates/ ./crates/
RUN cargo build --release --bin stormcore-api

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libssl3 ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/stormcore-api /usr/local/bin/stormcore
EXPOSE 3000
CMD ["stormcore"]
```

---

## 8. Tile Rendering Architecture

### 8.1 Rendering Pipeline

```
GET /tiles/radar/8/67/102?t=2024-06-15T19:30:00Z

1. Parse z/x/y → bounding box (WGS84)
2. Check Redis cache: key = "radar:8:67:102:2024-06-15T19:30:00Z"
   └─ HIT → return cached PNG (< 2ms)
   └─ MISS → continue

3. Query TimescaleDB for RadarFrame nearest to timestamp
   WHERE product = 'REFL_QC'
     AND ST_Intersects(bbox, tile_bbox)
     AND valid_time NEAR timestamp

4. Rasterise grid slice to 256x256 pixels
5. Apply NWS reflectivity colour scale (colorgrad)
6. Encode as PNG (image crate)
7. Store in Redis (TTL = permanent for historical, 5min for live)
8. Return PNG
```

### 8.2 NWS Reflectivity Colour Scale

| dBZ Range | Colour | Hex | Meaning |
|-----------|--------|-----|---------|
| < 20 | Transparent | 000000/0 | No precipitation |
| 20 - 30 | Light green | 00FF00 | Light rain |
| 30 - 40 | Yellow-green | AAFF00 | Moderate rain |
| 40 - 50 | Yellow | FFFF00 | Heavy rain |
| 50 - 55 | Orange | FF8800 | Very heavy / possible hail |
| 55 - 60 | Red | FF0000 | Severe — likely hail |
| 60 - 65 | Magenta | FF00FF | Extreme — large hail |
| > 65 | White | FFFFFF | Giant hail / anomalous |

---

## 9. Observability and Monitoring

```
// Key metrics emitted per ingest crate (Prometheus)
stormcore_ingest_lag_seconds{product="MRMS_REFL_QC"}  // Target < 300s
stormcore_ingest_lag_seconds{product="NEXRAD_KEWX"}
stormcore_ingest_lag_seconds{product="LIGHTNING"}
stormcore_ingest_frames_total{product, status}
stormcore_ingest_gap_detected{product}               // Alert if >1

// API metrics
stormcore_query_duration_seconds{endpoint, cache}
stormcore_tile_cache_hit_ratio
stormcore_claim_queries_total{query_type, confidence_band}

// Database metrics
stormcore_db_query_duration_seconds{query}
stormcore_db_chunk_size_bytes{hypertable}

// Alerts (Grafana)
- IngestLag > 10 minutes for any product → PagerDuty
- IngestGapDetected (missing expected file) → Slack
- API p95 latency > 1000ms → Slack
- TimescaleDB disk usage > 80% → PagerDuty
```

---

## 10. Security

| Concern | Mitigation |
|---------|-----------|
| API authentication | Internal mTLS between claims-monorepo and StormCore; API key for dev environments |
| Network exposure | StormCore not public-facing; VPC-internal only, accessed via private DNS |
| Data at rest | TimescaleDB encryption at rest (RDS default); S3 data public but not sensitive |
| Secrets management | AWS Secrets Manager for DB credentials; no secrets in environment variables |
| Dependency supply chain | cargo audit on CI; Dependabot for Cargo.lock updates |
| Data attribution | All query responses include DataProvenance with NOAA/NWS attribution as required by data use terms |

---

## 11. Build Order and Milestones

| # | Crate / Package | Deliverable | Week |
|---|----------------|-------------|------|
| 1 | weather-store | TimescaleDB schema, migrations, repository traits | 1 |
| 2 | mrms-ingest | MRMS S3 polling, GRIB2 decode, frame storage | 1-2 |
| 3 | stormcore-api | Axum skeleton, /v1/claims/query, /health endpoints | 2 |
| 4 | claims-engine | WeatherEvidence compilation, confidence scoring | 3 |
| 5 | stormcore-client | TypeScript SDK, claims-monorepo integration | 3-4 |
| 6 | alerts-ingest | NWS CAP polling, warning polygon storage | 4 |
| 7 | lightning-ingest | Blitzortung WebSocket, bulk strike storage | 5-6 |
| 8 | nexrad-decode | Level II parsing, station sweep processing | 6-7 |
| 9 | radar-composite | Grid compositing, tile render, Redis cache | 7-9 |
| 10 | tile-server | XYZ tile endpoints, Mapbox compatibility | 9-10 |
| 11 | observability | Prometheus metrics, Grafana dashboards, alerts | 10-11 |
| 12 | backfill | NEXRAD historical backfill scripts, gap detection | 11-12 |
