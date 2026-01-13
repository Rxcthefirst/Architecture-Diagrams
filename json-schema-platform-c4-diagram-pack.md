# JSON Schema Platform – C4 Diagram Pack (Mermaid)

## Purpose
This document contains a **diagram-only pack** for architecture and design reviews. It includes C4-style Context, Container, and Component views expressed as Mermaid diagrams, plus supporting deployment and data-flow diagrams.

> Intended use: copy diagrams into docs, wikis, PRDs, ADRs, or slide decks that support Mermaid rendering.

---

## 1. C4 Context View

```mermaid
flowchart LR
  %% Actors
  Author[Schema Author]
  Reviewer[Reviewer]
  Admin[Platform Admin]
  Consumer[Schema Consumer\n(API/Event Producer/Consumer)]
  Audit[Audit/Compliance]

  %% System boundary (conceptual)
  UI[Schema Builder UI\n(Angular)]
  SMS[Schema Management Platform]

  %% External systems
  Glue[AWS Glue Schema Registry]
  MKS[MKS / ALM Tool\n(Optional)]

  %% Interactions
  Author -->|Create/modify schemas| UI
  Reviewer -->|Review/approve changes| UI
  Admin -->|Configure policies & access| UI

  UI -->|Use platform APIs| SMS
  Consumer -->|Resolve schema versions\n(fetch latest/published)| SMS
  Audit -->|Review evidence| SMS

  SMS -->|Publish canonical versions\n+ enforce compatibility| Glue
  SMS -. Link change request .-> MKS
```

---

## 2. C4 Container View

```mermaid
flowchart TB
  subgraph Client["Client"]
    UI["Angular Schema Builder UI\n(Tree + Wizard + Types Library)"]
  end

  subgraph AWS["AWS Account"]
    APIGW["API Gateway\n(REST)"]
    SVC["Schema Management Service\n(Lambda or ECS/Fargate)"]
    DDB["DynamoDB\n(Metadata + Workflow State)"]
    S3["S3\n(Artifacts, exports, diffs)"]
    SFN["Step Functions\n(Approval workflows)"]
    Glue["Glue Schema Registry\n(Canonical versions + compatibility)"]
    EB["EventBridge\n(Domain events)"]
    SNS["SNS/ChatOps\n(Notifications)"]
    CW["CloudWatch\n(Logs/Metrics)"]
    CT["CloudTrail\n(Audit trail)"]
    KMS["KMS\n(Encryption keys)"]
  end

  UI -->|HTTPS| APIGW --> SVC

  SVC --> DDB
  SVC --> S3
  SVC --> SFN
  SVC --> Glue
  SVC --> EB --> SNS

  APIGW --> CW
  SVC --> CW
  SFN --> CW

  APIGW --> CT
  SVC --> CT

  DDB --- KMS
  S3 --- KMS
  Glue --- KMS
```

---

## 3. C4 Component View – Schema Management Service

```mermaid
flowchart LR
  subgraph SVC["Schema Management Service"]
    AuthZ["AuthZ & Policy Gate\n(RBAC/ABAC + domain policies)"]
    Subjects["Subjects API\n(create/search/tag/owner)"]
    Versions["Versions API\n(draft/propose/approve/publish)"]
    Validate["Validator\n(syntax + org rules)"]
    Workflow["Workflow Adapter\n(Step Functions tasks)"]
    Compat["Compatibility Gate\n(Glue registration strategy)"]
    Artifacts["Artifact Generator\n(pretty schema, docs, diff)"]
    Audit["Audit Logger\n(append-only events)"]
    Events["Event Publisher\n(EventBridge)"]
  end

  AuthZ --> Subjects
  AuthZ --> Versions

  Subjects --> DDB[(DynamoDB)]
  Versions --> DDB
  Validate --> DDB
  Audit --> DDB

  Versions --> Validate
  Versions --> Workflow
  Workflow --> Compat
  Compat --> Glue[AWS Glue Schema Registry]

  Compat --> Artifacts
  Artifacts --> S3[(S3)]
  Artifacts --> Events
  Audit --> Events
  Events --> EB[EventBridge]
```

---

## 4. Deployment View (Environments and Registries)

```mermaid
flowchart LR
  Dev["Dev Environment"] --> Stg["Staging Environment"] --> Prod["Production Environment"]

  Dev --> DevReg["Glue Registry (Dev)"]
  Stg --> StgReg["Glue Registry (Staging)"]
  Prod --> ProdReg["Glue Registry (Prod)"]

  Dev --> DevMeta["DynamoDB (Dev)"]
  Stg --> StgMeta["DynamoDB (Staging)"]
  Prod --> ProdMeta["DynamoDB (Prod)"]

  Dev --> DevS3["S3 (Dev)"]
  Stg --> StgS3["S3 (Staging)"]
  Prod --> ProdS3["S3 (Prod)"]
```

---

## 5. Publish Data Flow (Governance → Enforcement)

```mermaid
flowchart TD
  Draft["Draft version created"] --> Proposed["Proposed for review"]
  Proposed --> Approved["Approved (workflow gate)"]
  Approved --> PublishAttempt["Attempt publish (register in Glue)"]

  PublishAttempt -->|Compatible| Published["Published (canonical in Glue)\nArtifacts generated\nEvent emitted"]
  PublishAttempt -->|Incompatible| Rejected["Rejected (compatibility error)\nReturned to author"]

  Published --> Deprecated["Deprecated (optional)"]
```

---

## 6. Data Model (Logical ER)

```mermaid
erDiagram
  SCHEMA_SUBJECT ||--o{ SCHEMA_VERSION : has
  SCHEMA_VERSION ||--o{ APPROVAL_RECORD : has
  SCHEMA_VERSION ||--o{ AUDIT_EVENT : generates

  SCHEMA_SUBJECT {
    string subjectId
    string name
    string domain
    string owner
    string compatibilityMode
  }

  SCHEMA_VERSION {
    string versionId
    string subjectId
    int versionNumber
    string status
    string glueVersionId
    string contentLocation
    string contentHash
  }

  APPROVAL_RECORD {
    string approvalId
    string versionId
    string approver
    string decision
    datetime decidedAt
  }

  AUDIT_EVENT {
    string eventId
    string versionId
    string type
    datetime timestamp
    string actor
  }
```

---

## 7. UI-to-$ref Modeling Diagram (Supporting View)

```mermaid
flowchart LR
  UI["Angular Schema Builder\n(Tree + Wizard)"] --> AST["Schema AST (Source of Truth)"]
  AST --> JSON["Generated JSON Schema\n(Draft 2020-12)"]

  AST --> Defs["Reusable Types Library\n($defs)"]
  AST --> Refs["Reference Nodes\n($ref)"]

  UI -->|Promote inline object| Defs
  UI -->|Replace inline with ref| Refs
```

---

## Notes
- These diagrams are C4-style but expressed using Mermaid flowcharts for portability.
- If your documentation tooling supports Mermaid, you can embed these diagrams directly.
