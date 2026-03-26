# Certificate Import Pipeline

Entry point: [`lib/use_case/import_certificates.rb`](../lib/use_case/import_certificates.rb)  
Core logic: [`lib/use_case/import_xml_certificate.rb`](../lib/use_case/import_xml_certificate.rb)

---

## Step 1 — Queue consumption (`QueueWorker` → `ImportCertificates`)

The `QueueWorker` (in `app.rb`) polls continuously. Each loop calls `PullQueues`, which delegates to `ImportCertificates`. That use case pulls up to 50 **assessment IDs** at a time from a Redis list (the `:assessments` queue) via `QueuesGateway`. The IDs are also written to a **recovery list** in Redis so that if the process crashes mid-batch, they can be replayed.

---

## Step 2 — Fetch XML + Metadata in parallel (`ImportXmlCertificate`)

For each assessment ID, `ImportXmlCertificate` calls the **EPB Register API** via `RegisterApiGateway` — two async HTTP requests fired concurrently using the `Async` gem:

- `GET /api/assessments/{id}` → the raw **XML** of the certificate
- `GET /api/assessments/{id}/meta-data` → **metadata** (schema type, assessment type, country, address ID, cancelled/opted-out flags, etc.)

If either request returns 410 Gone, the certificate is marked as unimportable and skipped silently.

---

## Step 3 — Exclusion rules

Before parsing, a few conditions cause the certificate to be **skipped**:

- The metadata indicates it should be excluded (via `should_exclude?` — e.g. test certificates)
- The XML or metadata came back nil
- The certificate is flagged as a **Green Deal** assessment

If an existing EAV record is found for the assessment ID, it is **deleted** before re-importing (full replace, not update).

---

## Step 4 — XML Parsing (`ParseXmlCertificate`)

The schema type from metadata (e.g. `"RdSAP-Schema-21.0.1"`, `"SAP-Schema-19.2.0"`, `"CEPC-8.0.0"`) is used to select the correct **export configuration class** from a map of ~35 schema versions, covering:

- RdSAP (England/Wales and Northern Ireland variants)
- SAP (England/Wales and Northern Ireland variants)
- CEPC / CEPC-NI

The XML is parsed via `XmlPresenter::Parser` in a **subprocess** (via the `parallel` gem) to isolate memory usage. The result is a flat Ruby hash of attribute name → value.

---

## Step 5 — Enrich with metadata

The parsed hash is augmented with fields extracted from the metadata response:

| Field | Source |
|-------|--------|
| `assessment_address_id` | `meta_data[:assessmentAddressId]` |
| `created_at` | `meta_data[:createdAt]` |
| `schema_type` | `meta_data[:schemaType]` |
| `assessment_type` | `meta_data[:typeOfAssessment]` |
| `hashed_assessment_id` | `meta_data[:hashedAssessmentId]` |
| `cancelled_at` | `meta_data[:cancelledAt]` |
| `opt_out` | current timestamp if `meta_data[:optOut]` is true |

---

## Step 6 — Persist to three stores (`ImportCertificateData`)

The enriched certificate hash is saved in three ways:

| Store | Method | Purpose |
|-------|--------|---------|
| **EAV** | `assessment_attribute_gateway.add_attribute_values` | Flexible per-attribute querying and export |
| **Document (JSONB)** | `documents_gateway.add_assessment` | Full-document bulk export to S3 |
| **Search** | `assessment_search_gateway.insert_assessment` | Fast denormalised search by postcode, band, type, etc. |

For commercial certificates (CEPC/DEC), a record is also written to `commercial_reports` linking the assessment to its related RRN.

The country ID from metadata is written to `assessments_country_ids`.

---

## Step 7 — Recovery list cleanup

On success, the assessment ID is removed from the Redis recovery list.

On failure:

- The error is logged
- The attempt is registered in the recovery list (retry counter incremented)
- If the assessment is on its **last allowed attempt**, the error is reported to Sentry
- `Errors::ConnectionApiError` is not retried (not registered to recovery list)
- `UnimportableAssessment` errors clear the ID from the recovery list without retrying
