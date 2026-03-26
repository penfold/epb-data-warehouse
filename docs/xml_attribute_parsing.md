# XML Attribute Parsing and JSON Storage

## Overview

Several element types in the RdSAP XSD schemas use **XML attributes** to carry metadata (units, currency, language) alongside their textual content. This document explains how those attributes are parsed by the SAX parser and how they are represented in the stored JSON document.

---

## XSD Types That Use Attributes

### `Measurement`

```xml
<xs:complexType name="Measurement">
  <xs:simpleContent>
    <xs:extension base="xs:decimal">
      <xs:attribute name="quantity" type="xs:string" use="optional"/>
    </xs:extension>
  </xs:simpleContent>
</xs:complexType>
```

Used for physical measurements such as areas, lengths, and energy figures. The `quantity` attribute describes the unit (e.g. `metres`, `square metres`, `kWh/m²`).

**Examples in XSD:** `Heat-Loss-Perimeter`, `Total-Floor-Area`, `Room-Height`, `Party-Wall-Length`, `Energy-Consumption-Current`, `Energy-Consumption-Potential`, `CO2-Emissions-Current`, `CO2-Emissions-Potential`, `CO2-Emissions-Current-Per-Floor-Area`.

### `Money`

```xml
<xs:complexType name="Money">
  <xs:simpleContent>
    <xs:extension base="xs:decimal">
      <xs:attribute name="currency" type="xs:string" use="optional"/>
    </xs:extension>
  </xs:simpleContent>
</xs:complexType>
```

Used for cost figures. The `currency` attribute contains the ISO 4217 currency code (e.g. `GBP`).

**Examples in XSD:** `Lighting-Cost-Current`, `Lighting-Cost-Potential`, `Heating-Cost-Current`, `Heating-Cost-Potential`, `Hot-Water-Cost-Current`, `Hot-Water-Cost-Potential`, `Typical-Saving`.

### `Sentence` (language attribute)

Although not a named XSD type, many string elements (such as `Description` in `PropertySummaryType` and `Dwelling-Type`) carry an optional `language` attribute that indicates the language code of the text.

---

## How the SAX Parser Handles Attributes

The SAX parser (`lib/xml_presenter/parser.rb`) uses an `AssessmentDocument` handler:

1. **`start_element`** — when it encounters an opening XML tag, it captures any XML attributes into the instance variable `@attrs`.

2. **`characters`** — accumulates the text content of the current element.

3. **`flush_value_buffer`** — merges the text content with the captured attributes:
   - If `@attrs` is **non-empty**: the value is stored as a **hash** with the attribute names as keys plus a special `"value"` key for the text content.
   - If `@attrs` is **empty**: the value is stored as a **scalar** (string, integer, or decimal).

---

## JSON Output: Two Possible Formats per Field

Because the `quantity`, `currency`, and `language` attributes are all **optional** in the XSD, the JSON representation of a field depends entirely on **whether the originating XML included the attribute**.

### Scalar (attribute absent)

```xml
<Energy-Consumption-Current>230</Energy-Consumption-Current>
```

Stored as:

```json
"energy_consumption_current": 230
```

### Object (attribute present)

```xml
<Heat-Loss-Perimeter quantity="metres">27.09</Heat-Loss-Perimeter>
```

Stored as:

```json
"heat_loss_perimeter": { "quantity": "metres", "value": 27.09 }
```

```xml
<Lighting-Cost-Current currency="GBP">123</Lighting-Cost-Current>
```

Stored as:

```json
"lighting_cost_current": { "currency": "GBP", "value": 123 }
```

```xml
<Dwelling-Type language="1">Mid-terrace house</Dwelling-Type>
```

Stored as:

```json
"dwelling_type": { "language": "1", "value": "Mid-terrace house" }
```

---

## API Consumer Implications

Because both representations are valid, **API consumers must handle both forms** for any field of type `Measurement`, `Money`, or `Sentence`. In practice:

| Field category | Common form observed | Possible alternative |
|---|---|---|
| Floor dimensions (`heat_loss_perimeter`, `room_height`, `total_floor_area`, `party_wall_length`) | Object `{quantity, value}` | Scalar number |
| Cost fields (`lighting_cost_current`, etc.) | Varies by schema version; older XML often includes currency | Scalar decimal |
| Energy consumption (`energy_consumption_current/potential`) | Scalar integer (rarely attributed in XML) | Object `{quantity, value}` |
| CO2 emissions | Scalar decimal (rarely attributed in XML) | Object `{quantity, value}` |
| Property summary descriptions | Object `{language, value}` in older XML; scalar string in newer | Both forms seen |
| `dwelling_type` | Object `{language, value}` in older XML; scalar string in newer | Both forms seen |
| `Typical-Saving` in improvements | Varies — some XML includes currency, some does not | Both forms seen |

---

## Special Case: `nodes_ignoring_attributes` in RdSAP Schema 17

The export configuration for **RdSAP-Schema-17.x** (including NI-17.x) explicitly declares:

```ruby
nodes_ignoring_attributes(%w[Energy-Consumption-Current Energy-Consumption-Potential])
```

This instructs the parser to **discard all XML attributes** for these two elements, regardless of what the source XML contains. As a result:

- `energy_consumption_current` is **always** a plain integer in schema 17.x responses.
- `energy_consumption_potential` is **always** a plain integer in schema 17.x responses.

This was the original behaviour for these energy figures. From schema 18.x onwards the attributes are preserved and both scalar and object forms are possible.

---

## SAP-Special-Features and Air-Change-Rates

### Schema version availability

| Schema version range | `SAP-Special-Features` in XSD | Treated as list node |
|---|---|---|
| RdSAP-Schema-17.x, 18.x | Not present | N/A |
| RdSAP-Schema-19.x, 20.x, 21.x | Present (optional) | Yes — stored as array |

When present, `SAP-Special-Features` is stored as an array of objects. Each object contains a `description` (string) and either an `energy_feature` or `emissions_feature` sub-object. An optional `air_change_rates` array (12 monthly rate objects) may be nested inside an `energy_feature`.

---

## Summary

The dual-format behaviour is a consequence of the XML schema design (all attributes are optional) and the streaming SAX parser implementation. JSON schemas for the GET certificate API use `anyOf` to model these polymorphic fields, permitting either a scalar value or a typed object.
