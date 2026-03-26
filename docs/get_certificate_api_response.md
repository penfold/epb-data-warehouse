# How the GET Certificate API Produces its JSON Response

Endpoint: `GET /api/certificate?certificate_number={id}`  
Controller: [`lib/controller/certificate_controller.rb`](../lib/controller/certificate_controller.rb)

---

## End-to-End Flow

```
HTTP GET /api/certificate?certificate_number=XXXX-XXXX-XXXX-XXXX
  └── CertificateController
        └── GetRedactedCertificate (use case)
              └── DocumentsGateway#fetch_by_id
                    └── PostgreSQL: fn_export_json_document(document, matched_uprn)
                          └── JSON.parse → Ruby hash
                                └── BaseController#convert_to_json (camelCase disabled)
                                      └── { data: <hash> }.to_json → HTTP response
```

---

## Step 1 — Controller

`CertificateController` receives the request and passes `certificate_number` to `GetRedactedCertificate`. Notably, `@camel_case_keys = false` is set explicitly for this endpoint — meaning the JSON keys remain **snake_case** (unlike most other endpoints which convert to camelCase).

---

## Step 2 — Use Case (`GetRedactedCertificate`)

Trivially delegates to `DocumentsGateway#fetch_by_id`. Raises `Errors::CertificateNotFound` (→ 404) if nothing is returned.

---

## Step 3 — Database query (`DocumentsGateway#fetch_by_id`)

```sql
SELECT fn_export_json_document(ad.document, ad.matched_uprn) AS document
FROM assessment_documents ad
WHERE ad.assessment_id = $1
AND EXISTS (SELECT * FROM assessment_search s WHERE s.assessment_id = ad.assessment_id)
```

Two things to note:
- **`ad.document`** is the raw JSONB column written at import time — the complete hash produced by the SAX parser, enriched with metadata fields.
- The `EXISTS` subquery ensures only certificates present in `assessment_search` are returned. Certificates that are opted-out, cancelled etc. are absent from `assessment_search` and therefore return 404.
- The result of `fn_export_json_document` is parsed with `JSON.parse(..., symbolize_names: true)` and returned as a Ruby hash.

---

## Step 4 — PostgreSQL function: `fn_export_json_document(document jsonb, matched_uprn bigint)`

This PL/pgSQL function is where the document is **transformed and redacted** before being returned. It does the following:

### 4a — Field removal (redaction)

Several fields are stripped from the document because they are sensitive or internal:

| Removed field | Reason |
|---------------|--------|
| `scheme_assessor_id` | Identifies the assessor — PII |
| `equipment_owner` | PII / commercial |
| `equipment_operator` | PII / commercial |
| `owner` | PII |
| `occupier` | PII |
| `assessment_address_id` | Internal address matching key |
| `calculation_software_name` | Internal |
| `opt_out` | Internal flag |
| `hashed_assessment_id` | Internal |
| `cancelled_at` | Internal |
| `related_rrn` | Replaced (see below) |
| `matched_uprn` | Replaced (see below) |

### 4b — Field rename: `related_rrn` → `related_certificate_number`

If the document contains a `related_rrn` field (e.g. a CEPC linked to a recommendation report), it is re-exposed under the public-facing name `related_certificate_number`.

### 4c — Energy band calculation

The function derives letter bands (A–G or A+–G for non-domestic) from numeric ratings using the `energy_band_calculator` PostgreSQL function:

| Assessment type | Source field | Band scale |
|----------------|-------------|-----------|
| SAP / RdSAP | `energy_rating_current` | SAP scale: ≤20=G … ≥92=A |
| SAP / RdSAP | `energy_rating_potential` | Same SAP scale |
| CEPC | `asset_rating` | Non-domestic scale: ≤-1=A+ … ≥151=G |
| DEC | `this_assessment.energy_rating` | Non-domestic scale |
| Recommendation reports (`*RR*`) | n/a | `null` |

The calculated bands are injected into the document as:
- `current_energy_efficiency_band`
- `potential_energy_efficiency_band` (domestic only)

#### SAP band thresholds

| Band | Range |
|------|-------|
| G | ≤ 20 |
| F | 21–38 |
| E | 39–54 |
| D | 55–68 |
| C | 69–80 |
| B | 81–91 |
| A | ≥ 92 |

#### Non-domestic band thresholds (CEPC/DEC)

| Band | Range |
|------|-------|
| A+ | ≤ -1 |
| A | 0–25 |
| B | 26–50 |
| C | 51–75 |
| D | 76–100 |
| E | 101–125 |
| F | 126–150 |
| G | ≥ 151 |

### 4d — UPRN resolution

The function resolves a public-facing `uprn` value from two possible sources, in priority order:

1. **Energy Assessor UPRN**: if `assessment_address_id` has the form `UPRN-<number>` (stripping the prefix), that numeric value is used and `uprn_source` is set to `"Energy Assessor"`.
2. **Address-matched UPRN**: if no assessor-supplied UPRN but `matched_uprn` (from the `assessment_documents.matched_uprn` column, populated by a separate address-matching process) is not null, that value is used and `uprn_source` is set to `"Address Matched"`.
3. **No UPRN**: if neither source exists, `uprn` is set to `null` and `uprn_source` to `""`.

The `assessment_address_id` field is removed from the output (it was an internal key).

---

## Step 5 — JSON serialisation

Back in the controller, `convert_to_json` serialises the hash. Because `@camel_case_keys = false`, keys stay as snake_case strings. The final HTTP response body is:

```json
{
  "data": {
    "address_line_1": "...",
    "postcode": "...",
    "assessment_type": "RdSAP",
    "energy_rating_current": 72,
    "current_energy_efficiency_band": "C",
    "potential_energy_efficiency_band": "B",
    "uprn": 123456789012,
    "uprn_source": "Energy Assessor",
    "walls": [...],
    "suggested_improvements": [...],
    ...
  }
}
```

---

## Summary

The document served by the API is fundamentally the **same JSONB blob written at import time** (the SAX-parsed XML enriched with metadata), but put through a PostgreSQL function that:

1. Redacts sensitive/internal fields
2. Renames `related_rrn` → `related_certificate_number`
3. Computes and injects energy band letters from numeric ratings
4. Resolves and injects a `uprn` + `uprn_source` from the assessor-supplied address ID or a separately matched UPRN
