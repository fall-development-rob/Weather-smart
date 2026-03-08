# StormCore Weather Engine — Documentation

> Xweather-Fidelity Weather Infrastructure
>
> **Version:** 1.0 | **Status:** DRAFT | **Author:** Rob-otix AI Ltd | **Date:** March 2026

---

## Documents

| Document | Description |
|----------|-------------|
| [Product Requirements (PRD)](./prd.md) | Goals, personas, use cases, functional & non-functional requirements, data sources, and phased delivery plan |
| [Architecture Requirements (ARD)](./ard.md) | Repository structure, component architecture, data schema, API design, TypeScript SDK, infrastructure, tile rendering, observability, and security |
| [Domain-Driven Design (DDD)](./ddd.md) | Ubiquitous language, bounded contexts, domain model (aggregates, value objects, events), repository interfaces, and domain services |

---

## Quick Links

- **Core Use Case:** Historical point-in-time weather verification and monitoring
- **Tech Stack:** Rust (ingest, decode, tile render) + TypeScript (SDK, API surface) + TimescaleDB + Redis
- **Data Sources:** NOAA MRMS, NEXRAD Level II, NWS CAP Alerts, Blitzortung, Iowa State ASOS — all free
- **Delivery:** 5 phases over 12 weeks (Core Ingest → Query API → Lightning → Radar Tiles → Hardening)
