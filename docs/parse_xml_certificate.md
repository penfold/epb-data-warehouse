# How `ParseXmlCertificate` Works

Source: [`lib/use_case/parse_xml_certificate.rb`](../lib/use_case/parse_xml_certificate.rb)

---

## Overview

`ParseXmlCertificate` takes raw XML and a schema type string, selects the correct parsing configuration for that schema version, and uses a Nokogiri SAX parser to produce a flat Ruby hash of attribute name → value. It runs in a subprocess to isolate memory usage.

---

## Step 1 — Schema type → Export configuration class

The `schema_type` string (e.g. `"RdSAP-Schema-21.0.1"`) is looked up in a static hash map of ~35 entries:

```
"RdSAP-Schema-21.0.1" => XmlPresenter::Rdsap::Rdsap21ExportConfiguration
"SAP-Schema-19.2.0"   => XmlPresenter::Sap::Sap1900ExportConfiguration
"CEPC-8.0.0"          => XmlPresenter::Cepc::Cepc800ExportConfiguration
...
```

If the schema type is unknown (not in the map), `nil` is returned and the certificate is treated as unimportable upstream.

Schema families covered:

| Family | Variants |
|--------|---------|
| RdSAP | England/Wales v17–21, Northern Ireland v17–21 |
| SAP | England/Wales v15–19, Northern Ireland v15–18 |
| CEPC | v7.0, v7.1, v8.0.0, NI v8.0.0 |

---

## Step 2 — Export configuration

Each configuration class inherits from `XmlPresenter::ToWarehouse::BaseConfiguration` and calls `setup` at class load time. `setup` declares rules that control how the XML is parsed:

| Rule | Purpose |
|------|---------|
| `excludes` | XML node names to skip entirely (e.g. `Energy-Assessor`, `Insurance-Details`, `RRN`) |
| `includes` | Nodes to include even if a parent is excluded (e.g. `Certificate-Number`) |
| `bases` | Container nodes whose name is stripped from the output key path (e.g. `Report-Header`, `SAP-Data`) |
| `preferred_keys` | Rename a node to a different output key (e.g. `Certificate-Number` → `scheme_assessor_id`) |
| `list_nodes` | Nodes whose children are collected into an array (e.g. `Suggested-Improvements`, `SAP-Floor-Dimensions`) |
| `rootless_list_nodes` | Repeated sibling nodes collected into a named array without a wrapping parent (e.g. `Wall` → `walls`) |

`to_args(sub_node_value: assessment_id)` converts the class-level config into a keyword argument hash for the parser. The `assessment_id` is used as a `sub_node_value` when the XML contains multiple reports (e.g. CEPC files containing both an EPC and a recommendation report) — it lets the parser locate the correct report by matching the assessment ID inside the XML.

---

## Step 3 — SAX parsing (`XmlPresenter::Parser`)

The parser wraps Nokogiri's SAX interface. SAX is used rather than DOM because it is streaming and memory-efficient — it does not load the entire XML tree into memory.

The `AssessmentDocument` SAX handler walks the XML node-by-node:

- **`start_element_namespace`**: pushes the node name onto an output position stack. If the node is a `base`, the name is not added to the key path. If the node is in `excludes`, reading is suppressed. If the node is a `list_node`, a new array is set up at the current position.
- **`characters`**: buffers the text content. Numeric strings are coerced to integers or floats. XML attributes on the node are merged into the value as a hash (e.g. `{ "value" => 42, "sap-code" => "1" }`). Values longer than 2,703 characters are truncated.
- **`end_element_namespace`**: flushes the buffered value into the output hash at the current key path, then pops the position stack.

The resulting `output` is a nested/flat Ruby hash, e.g.:

```ruby
{
  "address_line_1" => "10 Example Street",
  "postcode"       => "SW1A 1AA",
  "current_energy_rating" => "C",
  "walls"          => [{ "description" => "Cavity wall", "energy_efficiency_rating" => 4 }],
  "suggested_improvements" => [ ... ],
  ...
}
```

---

## Step 4 — Subprocess isolation (`Parallel.map`)

The entire parse lambda is run inside `Parallel.map([0])`, which forks a child process. This ensures any memory accumulated during XML parsing is released when the subprocess exits, preventing heap bloat in the long-running `QueueWorker` process.

When called with `use_subprocess: false` (as in some tests), parsing runs inline in the current process.

---

## Output

The method returns the hash produced by the SAX parser. If the schema type was unrecognised, it returns `nil`, which the caller (`ImportXmlCertificate`) treats as an unimportable certificate.
