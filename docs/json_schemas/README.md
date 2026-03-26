# JSON Schema Reference Files

This directory contains JSON Schema (draft-07) files that describe the shape of the JSON document returned by the `GET /api/certificate` endpoint for each EPC schema version. One file exists per schema version.

---

## How the schemas were derived

The JSON shape of a stored certificate is **not** a direct serialisation of the original XSD — it is the result of a two-stage transformation applied during import.

### Stage 1 — XSD → flat Ruby hash (SAX export configuration)

When a certificate is imported from the EPB Register, the raw XML is parsed by `XmlPresenter::Parser`, a Nokogiri SAX parser, guided by a per-schema-version **export configuration** class (e.g. `XmlPresenter::Rdsap::Rdsap21Base`).

The export configuration intentionally changes the shape compared to the original XSD in the following ways:

| Transformation | Example |
|---|---|
| **`excludes`** — certain subtrees are **dropped entirely** | `Energy-Assessor`, `Insurance-Details`, `Green-Deal-Package`, `RRN` |
| **`includes`** — individual nodes inside an excluded subtree are **re-admitted** | `Certificate-Number` inside `Report-Header` |
| **`bases`** — container/wrapper node names are **stripped from the key path** | `Report-Header`, `SAP-Data`, `Address` — their children are promoted one level up |
| **`preferred_keys`** — nodes are **renamed** | `Certificate-Number` → `scheme_assessor_id` |
| **`list_nodes`** — a wrapper node's children are **collected into an array** | `Suggested-Improvements`, `SAP-Floor-Dimensions` |
| **`rootless_list_nodes`** — repeated sibling elements with no shared wrapper are **collected into a named array** | `<Wall>` elements → `walls: [...]` |

XML attribute metadata (`quantity`, `currency`, `language`) carried by `Measurement`, `Money`, and `Sentence` XSD types is preserved: if the attribute is present in the source XML, the field is stored as an object (`{ "value": 230, "quantity": "kWh/m²" }`); if absent, it is stored as a plain scalar (`230`). This dual representation is reflected in the `measurement_value`, `money_value`, and `sentence_value` `$defs` shared types used throughout the schemas.

---

### Stage 2 — Ruby hash → stored JSONB document (`fn_export_json_document`)

After parsing, the certificate is stored in the `assessment_documents` table as a JSONB column. When it is fetched via `GET /api/certificate`, the PostgreSQL function `fn_export_json_document` performs a further set of transforms before the document is returned:

- Adds/replaces `uprn` derived from the `assessment_address_id`
- Adds `current_energy_efficiency_band` (the letter band, e.g. `"C"`) computed from the numeric rating via `energy_band_calculator`
- Adds `potential_energy_efficiency_band` similarly
- Adds associated `related_rrn` for CEPC certificates

The JSON schemas in this directory reflect the output **after both stages** — i.e. the shape a consumer of the API will actually receive.

---

### Stage 3 — Metadata enrichment

Several top-level fields are not present in the original XML at all — they are injected from the EPB Register metadata API (`GET /api/assessments/{id}/meta-data`) during import:

| Field | Source |
|---|---|
| `assessment_type` | `typeOfAssessment` |
| `schema_type` | `schemaType` |
| `assessment_address_id` | `assessmentAddressId` |
| `hashed_assessment_id` | `hashedAssessmentId` |
| `cancelled_at` | `cancelledAt` |
| `created_at` | `createdAt` |

---

## Schema file naming

Each file is named `<schema-type>.json`, matching the `schemaType` value returned by the Register metadata API and stored in `assessment_documents.schema_type`. For example:

| File | Schema family | Region |
|---|---|---|
| `RdSAP-Schema-21.0.1.json` | RdSAP | England & Wales |
| `RdSAP-Schema-NI-21.0.1.json` | RdSAP | Northern Ireland |
| `SAP-Schema-19.2.0.json` | SAP | England & Wales |
| `SAP-Schema-NI-18.0.0.json` | SAP | Northern Ireland |

NI variants may have slightly different excluded/included fields due to their separate XSD lineage, reflected in their own export configuration classes (e.g. `Rdsap21NiExportConfiguration`).

---

## Further reading

- [docs/parse_xml_certificate.md](../parse_xml_certificate.md) — detailed walkthrough of the SAX parser and export configuration system
- [docs/xml_attribute_parsing.md](../xml_attribute_parsing.md) — how `Measurement`, `Money`, and `Sentence` XSD attributes become dual scalar/object fields in JSON
- [docs/certificate_import_pipeline.md](../certificate_import_pipeline.md) — full certificate import pipeline from queue to stored document
