# Data Lineage Specification
## Collibra → RDF Logical Model → REST JSON Schema Contracts (Attribute-Level)

---

## 1. Purpose

This document defines **attribute-level lineage** from Collibra governance metadata through:
1) RDF materialization (logical semantic model), and
2) JSON Schema contract generation (REST schema factory output).

It is intended to be included as **Section 6.x** (Information Model / Lineage) of the SDS, or published as an appendix.

---

## 2. Lineage Definitions

### 2.1 Lineage Objects
- **Source Attribute (Collibra):** a governed business definition of a data element.
- **Semantic Representation (RDF):** the logical model expression of entities, attributes, and relationships as triples.
- **Contract Representation (JSON Schema):** the consumer-facing schema describing entity objects and their properties.

### 2.2 Lineage Identifiers
To ensure traceability, every contract element SHOULD be traceable to stable identifiers:

- Collibra IDs (e.g., `collibra:assetId`)
- RDF IRIs (e.g., `ex:customerId`)
- Contract paths (e.g., `#/properties/customerId`)

---

## 3. Lineage View (High-Level)

```mermaid
flowchart LR
  C[Collibra Asset\n(Glossary Term / Attribute)] -->|Extract| X[Extracted Metadata Record]
  X -->|Transform/Map| R[RDF Triple Materialization]
  R -->|Load| DB[(RDF Store)]
  DB -->|Query| F[Schema Factory\n(SPARQL Abstraction)]
  F -->|Generate| J[JSON Schema Contract]
  J -->|Register| G[Glue Schema Registry\n(Published Versions)]
```

---

## 4. Provenance Model (Per-Run Traceability)

Each ETL execution (materialization run) SHOULD generate provenance artifacts that connect:
- the source snapshot (Collibra export),
- the transformation activity (ETL job run),
- the resulting RDF named graph,
- and the generated JSON Schemas.

### 4.1 Recommended PROV-O Pattern (Conceptual)

```mermaid
flowchart TB
  Src[Source Export\n(Collibra snapshot)] -->|prov:used| Act[Materialization Activity\n(ETL Run)]
  Act -->|prov:generated| Graph[Named Graph\n(run graph)]
  Graph -->|prov:wasDerivedFrom| Src

  Graph -->|queried by| Factory[Schema Factory]
  Factory -->|prov:generated| Schema[JSON Schema Artifact]
  Schema -->|published to| Glue[Glue Registry Version]
```

### 4.2 Named Graph Strategy
- Each run loads triples into a distinct named graph, e.g.:
  - `urn:graph:materialization:<runId>`
- Optionally, maintain a “latest” graph pointer for convenience:
  - `urn:graph:materialization:latest`

---

## 5. Attribute-Level Lineage Table (Template)

The following table is the minimum recommended lineage record per attribute.

| Domain | Entity (RDF Class) | Collibra Asset ID | Collibra Name | Collibra Description | RDF Property IRI | RDF Type | RDF Domain | RDF Range | JSON Schema Path | JSON Type | Required? | Constraints | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `<domain>` | `ex:Customer` | `<asset-id>` | `Customer ID` | `<desc>` | `ex:customerId` | DatatypeProperty | `ex:Customer` | `xsd:string` | `#/properties/customerId` | `string` | Yes/No | `minLength=1` | `<notes>` |

### 5.1 Required Columns (Minimum)
- Collibra Asset ID
- RDF Property IRI
- JSON Schema Path
- JSON Type
- Required?
- Constraints

---

## 6. Mapping Rules (Collibra → RDF → JSON Schema)

### 6.1 Collibra → RDF Mapping Rules
**Rule C2R-01 (Entities):**

If Collibra defines an entity term `T`, create an OWL class:

- `T` → `owl:Class`

- label/description copied into `rdfs:label`, `rdfs:comment`


**Rule C2R-02 (Attributes):**

If Collibra defines an attribute `A` belonging to entity `T`, create a datatype property:

- `A` → `owl:DatatypeProperty`

- `rdfs:domain` = class for `T`

- `rdfs:range` = mapped XSD type


**Rule C2R-03 (Relationships):**

If Collibra defines a relationship from entity `T1` to entity `T2`, create an object property:

- relationship `R` → `owl:ObjectProperty`

- `rdfs:domain` = class for `T1`

- `rdfs:range` = class for `T2`


### 6.2 RDF → JSON Schema Mapping Rules
**Rule R2J-01 (Class → Object Schema):**

RDF class `C` maps to JSON Schema:

- `"type": "object"`

- `properties` derived from properties with `rdfs:domain = C`


**Rule R2J-02 (DatatypeProperty → Field):**

RDF datatype property `p` maps to:

- `properties[pLocalName] = { "type": <mapped json type> }`

- constraints derived from Collibra metadata and/or SHACL


**Rule R2J-03 (ObjectProperty → $ref):**

RDF object property `op` maps to:

- `properties[opLocalName] = { "$ref": "#/$defs/<TargetType>" }`

- cardinality determines array vs single


**Rule R2J-04 (Cardinality → required/array):**

- minCount >= 1 → property included in `required`

- maxCount > 1 or unspecified-many → property becomes array


---

## 7. Worked Example (End-to-End Lineage)

This example illustrates a single attribute moving through the system.

Replace identifiers with your real Collibra and namespace values.

### 7.1 Collibra Input (Conceptual)
- **Entity:** Customer

- **Attribute:** Customer ID

- **Type:** string

- **Required:** yes

- **Definition:** Unique identifier for customer


### 7.2 RDF Materialization Output (Conceptual)

```ttl
ex:Customer a owl:Class ;
  rdfs:label "Customer" ;
  rdfs:comment "A party that holds accounts." .

ex:customerId a owl:DatatypeProperty ;
  rdfs:label "Customer ID" ;
  rdfs:comment "Unique identifier for customer." ;
  rdfs:domain ex:Customer ;
  rdfs:range xsd:string .
```

### 7.3 JSON Schema Output (Conceptual)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "schema://domain/customer/Customer",
  "title": "Customer",
  "type": "object",
  "properties": {
    "customerId": {
      "type": "string",
      "description": "Unique identifier for customer."
    }
  },
  "required": ["customerId"]
}
```

### 7.4 Lineage Record (Worked)

| Domain | Entity (RDF Class) | Collibra Asset ID | Collibra Name | RDF Property IRI | RDF Range | JSON Schema Path | JSON Type | Required? |
|---|---|---|---|---|---|---|---|---|
| customer | `ex:Customer` | `collibra:12345` | Customer ID | `ex:customerId` | `xsd:string` | `#/properties/customerId` | string | Yes |

---

## 8. Operationalizing Lineage (Storage and Retrieval)

### 8.1 Where Lineage Lives
Recommended storage options:

- **DynamoDB**: fast lookup by (entity, attribute) and by Collibra asset ID

- **S3**: immutable lineage snapshots per run

- **RDF Store**: optionally store lineage triples (e.g., `prov:wasDerivedFrom` relationships)


### 8.2 Lineage Retrieval API (Optional)
Example endpoints:

- `GET /lineage/entities/{entityName}`

- `GET /lineage/attributes/{collibraAssetId}`

- `GET /lineage/runs/{runId}`


---

## 9. Acceptance Criteria

A lineage implementation is considered complete when:

- Every published JSON Schema field maps to at least one RDF property IRI

- Every RDF property maps to a Collibra asset ID (where applicable)

- A runId can trace from Collibra snapshot → named graph → schema artifact

- Auditors can retrieve evidence of approvals and publication version IDs


---

## 10. Appendix: Suggested DynamoDB Indexing (Optional)

### 10.1 Lineage Table Keys (Example)
Primary key options:

- **PK:** `ENTITY#<domain>#<entityName>`

- **SK:** `ATTR#<jsonFieldName>`


Secondary indexes:

- GSI1 by Collibra Asset ID

  - **GSI1PK:** `COLLIBRA#<assetId>`

  - **GSI1SK:** `ENTITY#<domain>#<entityName>#ATTR#<jsonFieldName>`


---
