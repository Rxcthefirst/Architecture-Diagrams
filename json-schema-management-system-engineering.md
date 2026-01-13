# JSON Schema Management Platform
## Systems Engineering Documentation

---

## 1. Purpose

This document defines the systems engineering design for a platform that enables:

- Guided creation of JSON Schemas
- Schema persistence and versioning
- Controlled schema evolution
- Enforcement of producer/consumer contracts
- Governed change management with auditability

The system is designed for enterprise environments and emphasizes correctness, traceability, and controlled change while remaining accessible to non-expert users.

---

## 2. System Scope

### 2.1 In Scope
- JSON Schema authoring and management
- Schema lifecycle (draft → published → deprecated)
- Schema compatibility enforcement
- Approval workflows
- Audit and traceability
- AWS-based deployment

### 2.2 Out of Scope
- Runtime data ingestion or transformation
- Automatic client SDK generation
- Non-JSON schema formats (Avro, Protobuf)

---

## 3. Stakeholders

| Role | Responsibility |
|----|----|
| Schema Author | Create and update schemas |
| Reviewer | Approve or reject schema changes |
| Consumer | Use schemas as data contracts |
| Platform Admin | Configure policies and access |
| Audit / Compliance | Review change history |

---

## 4. C4 Context View

```mermaid
flowchart LR
  Author[Schema Author] --> UI[Schema Builder UI]
  Reviewer[Reviewer] --> UI
  Consumer[Schema Consumer] --> API[Schema Management API]

  UI --> API
  API --> Glue[AWS Glue Schema Registry]
  API --> DDB[(DynamoDB)]
  API --> S3[(S3)]
  API --> SFN[Step Functions]

  API -. optional .-> MKS[MKS / ALM Tool]
```

---

## 5. C4 Container View

```mermaid
flowchart TB
  subgraph Client
    UI[Angular Schema Builder UI]
  end

  subgraph AWS
    APIGW[API Gateway]
    API[Schema Management Service]
    DDB[DynamoDB]
    S3[S3]
    SFN[Step Functions]
    Glue[Glue Schema Registry]
    EB[EventBridge]
  end

  UI --> APIGW --> API
  API --> DDB
  API --> S3
  API --> SFN
  API --> Glue
  API --> EB
```

---

## 6. C4 Component View – Schema Management Service

```mermaid
flowchart LR
  API[API Layer]
  Validator[Schema Validator]
  Workflow[Workflow Orchestrator]
  Compat[Compatibility Gate]
  Audit[Audit Logger]
  Events[Event Publisher]

  API --> Validator
  API --> Workflow
  Workflow --> Compat
  Compat --> Glue[Glue Schema Registry]
  Workflow --> Audit
  Audit --> Events
```

---

## 7. Functional Requirements

### 7.1 Schema Authoring
- Guided creation of JSON Schemas
- Support for objects, arrays, scalars
- Reusable definitions via `$defs` / `$ref`

### 7.2 Versioning
- Immutable published versions
- Controlled creation of new versions
- Version metadata and changelogs

### 7.3 Compatibility
- Configurable compatibility modes
- Enforcement at publish time
- Rejection of incompatible changes

### 7.4 Change Management
- Draft → Review → Approval → Publish workflow
- Human approval gates
- Full audit trail

---

## 8. Schema Lifecycle

```mermaid
stateDiagram-v2
  Draft --> Proposed
  Proposed --> Approved
  Approved --> Published
  Proposed --> Rejected
  Approved --> Rejected
  Published --> Deprecated
```

---

## 9. Data Model (Logical)

```mermaid
erDiagram
  SCHEMA_SUBJECT ||--o{ SCHEMA_VERSION : contains
  SCHEMA_VERSION ||--o{ APPROVAL_RECORD : has
  SCHEMA_VERSION ||--o{ AUDIT_EVENT : generates

  SCHEMA_SUBJECT {
    string subjectId
    string domain
    string owner
    string compatibilityMode
  }

  SCHEMA_VERSION {
    string versionId
    int versionNumber
    string status
    string glueVersionId
  }
```

---

## 10. Validation Strategy

### 10.1 Client Side
- Continuous schema validation
- Example instance validation
- UX-friendly error feedback

### 10.2 Server Side
- Policy validation
- Lifecycle enforcement
- Compatibility enforcement via AWS Glue

---

## 11. Deployment Model

- Separate environments: Dev, Staging, Prod
- Independent Glue registries per environment
- Infrastructure defined via IaC

---

## 12. Security

- IAM-based access control
- Domain-scoped authorization
- Encryption at rest and in transit
- CloudTrail-based auditing

---

## 13. Risks and Mitigations

| Risk | Mitigation |
|----|----|
| Breaking changes | Compatibility enforcement |
| UX complexity | Guided authoring |
| Schema duplication | Reusable types |
| Approval bottlenecks | Policy-based routing |

---

## 14. Summary

This platform separates **schema governance** from **schema enforcement**:

- Governance, lifecycle, and UX are handled by the platform
- Compatibility and canonical versioning are enforced by AWS Glue

The result is a scalable, auditable, enterprise-ready schema management system.
