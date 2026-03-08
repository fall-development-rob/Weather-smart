# PRODUCT REQUIREMENTS DOCUMENT

## StormCore Weather Engine

> Xweather-Fidelity Weather Infrastructure
>
> **Version:** 1.0 | **Status:** DRAFT | **Author:** Rob-otix AI Ltd | **Date:** March 2026

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Problem Statement](#2-problem-statement)
- [3. Goals and Success Metrics](#3-goals-and-success-metrics)
- [4. User Personas and Use Cases](#4-user-personas-and-use-cases)
- [5. Functional Requirements](#5-functional-requirements)
- [6. Non-Functional Requirements](#6-non-functional-requirements)
- [7. Data Sources](#7-data-sources)
- [8. Out of Scope (v1)](#8-out-of-scope-v1)
- [9. Phased Delivery](#9-phased-delivery)

---

## 1. Executive Summary

StormCore is a Rust + TypeScript weather data engine built to match or exceed the fidelity of commercial providers such as Xweather and Weather Underground. It provides authoritative, high-resolution data for severe weather, storms, wildfires, and hurricanes through a clean, queryable API.

Rather than paying commercial API costs that scale with query volume, StormCore ingests freely available government data sources, processes them into structured, queryable formats, and exposes a clean API. Coverage spans the full spectrum of natural hazards — thunderstorms, tornadoes, hail, lightning, hurricanes, flooding, and wildfires. Historical data fidelity and data provenance are first-class concerns.

---

## 2. Problem Statement

### 2.1 Current Pain Points

- Commercial weather APIs (Xweather, Tomorrow.io) become cost-prohibitive at scale
- No single free API covers storms, fires, and hurricanes with the fidelity required for professional use
- Applications need point-in-time historical queries, not just current conditions
- Wildfire and hurricane tracking data is fragmented across multiple agencies (NIFC, FIRMS, NHC)
- Tile rendering for visual evidence is locked behind expensive commercial plans

### 2.2 Opportunity

The raw data required to match Xweather fidelity is freely available from NOAA, AWS Open Data, and NWS. The gap is the processing pipeline, not the data itself. A well-engineered ingest layer transforms free government data into a commercial-grade API at a fraction of the cost.

---

## 3. Goals and Success Metrics

### 3.1 Primary Goals

- Match Xweather data fidelity for severe weather, storms, wildfires, and hurricanes
- Eliminate per-request commercial API costs
- Provide point-in-time historical queries to 1991 for NEXRAD, 2001 for MRMS
- Unify storm, fire, and hurricane data from multiple agencies into a single API
- Support visual radar, fire perimeter, and hurricane track evidence generation

### 3.2 Success Metrics

| Metric | Target | Measurement Method |
|--------|--------|--------------------|
| Historical query latency | < 500ms p95 | API response time monitoring |
| Radar data freshness (live) | < 5 minutes | Ingest pipeline lag tracking |
| MRMS data coverage (US) | >= 95% CONUS | Grid coverage audit |
| Lightning strike latency | < 60 seconds | Strike timestamp vs ingest time |
| System uptime | >= 99.5% | Health check monitoring |
| Query accuracy vs NOAA | >= 99% | Spot-check validation suite |

---

## 4. User Personas and Use Cases

### 4.1 Primary Personas

**Application Developer**
A developer integrating StormCore into their application queries the API to verify weather conditions, track active wildfires, monitor hurricane paths, and produce structured event data for their users.

**Weather / Emergency Analyst (Human)**
A human analyst monitoring storms, wildfires, or hurricanes needs visual evidence — radar snapshots, fire perimeter maps, hurricane track overlays, lightning strike data — for situational awareness and reporting.

**Platform Engineer (Rob-otix)**
The internal engineering team needs to maintain the ingest pipelines, monitor data quality, replay historical data for backfilling, and add new data products as the platform evolves.

### 4.2 Core Use Cases

| ID | Use Case | Input | Output |
|----|----------|-------|--------|
| UC-01 | Historical point query | lat, lon, timestamp | Full weather conditions object |
| UC-02 | Hail event verification | lat, lon, date range | MESH hail size, probability, track |
| UC-03 | Lightning strike lookup | lat, lon, timestamp, radius | Strike count, distance, energy |
| UC-04 | Severe warning history | lat, lon, timestamp | Active warnings at time of event |
| UC-05 | Radar tile export | lat, lon, timestamp | PNG radar image for documentation |
| UC-06 | Wind event verification | lat, lon, timestamp | Observed/estimated wind speed, gust |
| UC-07 | Flood/precip verification | lat, lon, date range | QPE accumulation, return period |
| UC-08 | Live monitoring feed | region polygon | Real-time severe event stream |
| UC-09 | Wildfire detection | lat, lon, radius | Active fire hotspots, perimeters, spread direction |
| UC-10 | Hurricane track query | storm ID or lat/lon | Track history, forecast cone, wind radii, intensity |
| UC-11 | Fire weather conditions | lat, lon, timestamp | Red flag warnings, wind, humidity, fire weather index |

---

## 5. Functional Requirements

### 5.1 Data Ingestion

| ID | Requirement | Priority | Source |
|----|-------------|----------|--------|
| F-01 | Ingest MRMS GRIB2 files from AWS S3 on 2-minute cadence | MUST | NOAA MRMS |
| F-02 | Decode NEXRAD Level II binary into reflectivity/velocity sweeps | MUST | AWS NEXRAD |
| F-03 | Ingest NWS CAP alerts as GeoJSON polygons in real time | MUST | NWS API |
| F-04 | Ingest Blitzortung lightning strikes via WebSocket | MUST | Blitzortung |
| F-05 | Ingest ASOS/AWOS surface observations from Iowa State METAR feed | MUST | Iowa ASOS |
| F-06 | Backfill NEXRAD historical data from AWS S3 archive to 1991 | SHOULD | AWS NEXRAD |
| F-07 | Ingest MRMS MESH product for hail size estimation | MUST | NOAA MRMS |
| F-08 | Ingest MRMS QPE for quantitative precipitation estimates | MUST | NOAA MRMS |
| F-09 | Ingest MRMS rotation tracks for tornado detection | SHOULD | NOAA MRMS |
| F-10 | Ingest NCEI storm event database for verified reports | SHOULD | NOAA NCEI |
| F-25 | Ingest NASA FIRMS active fire hotspots (MODIS/VIIRS) | MUST | NASA FIRMS |
| F-26 | Ingest NIFC wildfire perimeters and incident data | MUST | NIFC GeoMAC |
| F-27 | Ingest NHC hurricane advisories, track forecasts, and wind radii | MUST | NHC/ATCF |
| F-28 | Ingest NWS Red Flag and Fire Weather Watch warnings | MUST | NWS API |

### 5.2 Query API

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| F-11 | Point-in-time weather query by lat/lon/timestamp | MUST | Core use case |
| F-12 | Radius query returning all events within N km of point | MUST | Variable radius support |
| F-13 | Time-range query returning aggregated conditions over period | MUST | Multi-day event support |
| F-14 | Hail query returning MESH value, probability, and size class | MUST | Critical weather product |
| F-15 | Lightning query with strike count and nearest strike distance | MUST | Fire/surge detection |
| F-16 | Polygon query for regional event detection | SHOULD | CAT event support |
| F-17 | Return structured confidence score per data product | MUST | Data quality scoring |
| F-18 | Return data provenance (source, resolution, age) | MUST | Data traceability |
| F-29 | Wildfire hotspot and perimeter query by lat/lon/radius | MUST | Fire monitoring |
| F-30 | Hurricane track and intensity query by storm ID or region | MUST | Hurricane tracking |
| F-31 | Fire weather index query (wind, humidity, red flag status) | MUST | Fire risk assessment |

### 5.3 Tile and Visualisation API

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| F-19 | Serve XYZ radar tiles for current and historical timestamps | MUST | Visual evidence |
| F-20 | Apply NWS reflectivity colour scale to radar tiles | MUST | Standard rendering |
| F-21 | Export point-in-time radar PNG for documentation | MUST | Document generation |
| F-22 | Serve lightning strike overlay tiles | SHOULD | Map visualisation |
| F-23 | Serve NWS alert polygon tiles | SHOULD | Warning display |
| F-32 | Serve wildfire perimeter and hotspot overlay tiles | MUST | Fire visualisation |
| F-33 | Serve hurricane track and forecast cone tiles | MUST | Hurricane visualisation |
| F-24 | Support Mapbox GL compatible tile endpoints | MUST | UI integration |

---

## 6. Non-Functional Requirements

| Category | Requirement | Target |
|----------|-------------|--------|
| Performance | Historical point query | < 500ms p95 |
| Performance | Tile serve latency | < 200ms p95 (cached) |
| Performance | Live data freshness | < 5 minutes lag |
| Availability | API uptime | >= 99.5% monthly |
| Storage | Hot data retention (TimescaleDB) | 12 months rolling |
| Storage | Cold archive (S3) | Indefinite, query on demand |
| Security | API authentication | Internal mTLS + API key |
| Compliance | Data attribution | NOAA/NWS attribution in responses |
| Scalability | Query concurrency | >= 100 concurrent requests |
| Observability | Ingest pipeline monitoring | Per-product lag + gap alerting |

---

## 7. Data Sources

| Source | Products | Update Rate | Access | Cost |
|--------|----------|-------------|--------|------|
| NOAA MRMS (AWS S3) | Radar mosaic, MESH hail, QPE, rotation | 2 min | S3 Open Data | Free |
| NEXRAD Level II (AWS) | Raw radar sweeps, all 160+ stations | 4-10 min | S3 Open Data | Free |
| NWS CAP Alerts | Tornado, severe wx, flash flood warnings | Real-time | REST API | Free |
| Blitzortung | Lightning strikes, global network | < 60 sec | WebSocket | Free |
| Iowa State ASOS | Surface obs (METAR), 900+ US stations | 5-60 min | HTTP feed | Free |
| NOAA NCEI | Verified storm events, historical | Daily | REST API | Free |
| NOAA SWDI | Hail, tornado, lightning reports | Daily | REST API | Free |
| NASA FIRMS | Active fire hotspots (MODIS/VIIRS), global | Near real-time | REST API | Free |
| NIFC (GeoMAC) | Wildfire perimeters, incident reports | Hourly | GeoJSON feed | Free |
| NHC / ATCF | Hurricane advisories, tracks, wind radii | 6 hours | REST / FTP | Free |
| Open-Meteo Archive | Model-based historical forecast data | Daily | REST API | Free |

---

## 8. Out of Scope (v1)

- International radar coverage (non-US)
- Marine / oceanic weather products
- Air quality data products
- Commercial PWS (personal weather station) network aggregation
- Mobile SDK or consumer-facing UI
- Multi-tenant / external API commercialisation

---

## 9. Phased Delivery

| Phase | Name | Deliverables | Timeline |
|-------|------|--------------|----------|
| 1 | Core Ingest | MRMS ingest, TimescaleDB schema, historical point query API | Weeks 1-3 |
| 2 | Query API | NWS alerts, TypeScript SDK, application integration | Weeks 4-5 |
| 3 | Lightning | Blitzortung ingest, strike query, tile overlay | Weeks 6-7 |
| 4 | Radar Tiles | NEXRAD decode, tile renderer, Mapbox-compatible endpoints | Weeks 8-10 |
| 5 | Fire & Hurricanes | FIRMS/NIFC fire ingest, NHC hurricane tracks, fire/storm tiles | Weeks 11-13 |
| 6 | Hardening | Historical backfill, monitoring, gap detection, load testing | Weeks 14-16 |
