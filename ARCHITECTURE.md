# epb-data-warehouse — Architecture Overview

## Purpose

This is the **Energy Performance of Buildings (EPB) Data Warehouse** — a Ruby/Sinatra application that ingests, stores, and serves Energy Performance Certificate (EPC) data for the UK. It acts as a secondary store downstream from the main EPB Register API.

---

## Architecture

The app is structured around a clean **layered architecture**:

| Layer | Location | Role |
|-------|----------|------|
| Controllers | `lib/controller/` | HTTP API endpoints (Sinatra/Rack) |
| Use Cases | `lib/use_case/` | Business logic |
| Gateways | `lib/gateway/` | DB / external service access |
| Boundaries | `lib/boundary/` | Input validation & error types |
| Domain | `lib/domain/` | Core domain models |

---

## Core Functionality

### 1. Data Ingestion (Queue Worker)

The `QueueWorker` in `app.rb` continuously polls AWS SQS queues and processes batches of assessment IDs. The `PullQueues` use case fans out to 6 sub-use-cases:

- **Import** new certificates (fetches XML from the Register API, parses it, stores it)
- **Cancel** certificates
- **Opt-out** certificates
- **Update addresses** (3 variants including backfill)

### 2. Data Storage Model

The database (PostgreSQL) uses two complementary storage approaches:

- **EAV (Entity-Attribute-Value)** model via `assessment_attributes`, `assessment_attribute_values`, and `assessment_lookups` tables — for flexible querying of individual data points
- **Document store** via `assessment_documents` (JSONB) — stores the full certificate as a JSON document for bulk export
- **`assessment_search`** — a denormalised search table for fast filtering by postcode, type, energy band, constituency, etc.

### 3. HTTP API (`DataWarehouseApiService`)

Defined in `lib/data_warehouse_api_service.rb`, it exposes endpoints grouped by controller:

| Controller | Endpoints |
|-----------|-----------|
| **Domestic** | Count and search SAP/RdSAP (residential) certificates |
| **Commercial** | Non-domestic EPC data |
| **Heat Pump** | Counts of heat pump installations by floor area |
| **Reporting** | Average CO₂ emissions |
| **Deltas** | Audit log queries for changed assessments within a date range |
| **File** | Pre-signed S3 URL generation and file info |
| **Codes** | Lookup/enum code data |
| **Display** | Certificate display data |
| **Certificate** | Individual certificate fetch and redacted versions |

### 4. Data Export

Rake tasks in `lib/tasks/` handle:

- Exporting assessment documents to S3 (multipart)
- Refreshing materialized views (e.g. open data exports)
- Emailing heat pump data reports
- Importing ONS postcode directory data
- Importing XSD enumerable values (e.g. mapping integer codes like `1` → `"dual"` for energy tariff)

### 5. ONS Geography

The `ons_postcode_directory` and `ons_postcode_directory_names` tables map postcodes to regions, local authorities, and Westminster parliamentary constituencies — used to enrich certificates with geographic context.

---

## External Integrations

| Service | Purpose |
|---------|---------|
| **AWS SQS** | Queue source for incoming assessment IDs |
| **AWS S3** | Storage for bulk exports and file downloads |
| **Redis** | Recovery list for failed queue messages |
| **EPB Register API** | Source of truth for certificate XML data |
| **GOV.UK Notify** | Email notifications (e.g. heat pump reports) |
| **Sentry** | Error tracking |
