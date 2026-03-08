# StormCore Weather Engine — Documentation

> Xweather-Fidelity Weather Infrastructure
>
> **Version:** 1.0 | **Status:** DRAFT | **Author:** Rob-otix AI Ltd | **Date:** March 2026

---

## Documents

| ID | Document | Description |
|----|----------|-------------|
| [ADR-001](./adr-001.md) | Product Requirements (PRD) | Goals, personas, use cases, functional & non-functional requirements, data sources, and phased delivery plan |
| [ADR-002](./adr-002.md) | Architecture Requirements (ARD) | Repository structure, component architecture, data schema, API design, TypeScript SDK, infrastructure, tile rendering, observability, and security |
| [ADR-003](./adr-003.md) | Domain-Driven Design (DDD) | Ubiquitous language, bounded contexts, domain model (aggregates, value objects, events), repository interfaces, and domain services |

---

## Quick Links

- **Core Use Case:** Real-time and historical monitoring for severe weather, storms, wildfires, and hurricanes
- **Tech Stack:** Rust (ingest, decode, tile render) + TypeScript (SDK, API surface) + TimescaleDB + Redis
- **Data Sources:** NOAA MRMS, NEXRAD Level II, NWS CAP Alerts, Blitzortung, Iowa State ASOS, NIFC/FIRMS fire data, NHC hurricane tracks — all free
- **Delivery:** 6 phases over 16 weeks (Core Ingest → Query API → Lightning → Radar Tiles → Fire & Hurricanes → Hardening)
