# JSON Schema Platform – Sequence Diagrams

## Purpose
This document contains **sequence diagrams only** for key platform flows. Diagrams are expressed in Mermaid for easy embedding in engineering documentation.

---

## 1. Save Draft Schema (Authoring)

```mermaid
sequenceDiagram
  autonumber
  actor Author as Schema Author
  participant UI as Angular UI
  participant API as Schema Mgmt API
  participant DDB as DynamoDB

  Author->>UI: Edit schema (tree + wizard)
  UI->>API: SaveDraft(subject, draftSchema)
  API->>DDB: Put draft version + metadata (status=Draft)
  API-->>UI: Draft saved + validation feedback
```

---

## 2. Propose Schema for Review

```mermaid
sequenceDiagram
  autonumber
  actor Author as Schema Author
  participant UI as Angular UI
  participant API as Schema Mgmt API
  participant SFN as Step Functions
  participant DDB as DynamoDB

  Author->>UI: Click "Propose"
  UI->>API: ProposeVersion(versionId)
  API->>SFN: StartExecution(approvalWorkflow, versionId)
  SFN->>DDB: Update version status=Proposed
  API-->>UI: Proposed (awaiting review)
```

---

## 3. Review and Approve

```mermaid
sequenceDiagram
  autonumber
  actor Reviewer as Reviewer
  participant UI as Angular UI
  participant API as Schema Mgmt API
  participant SFN as Step Functions
  participant DDB as DynamoDB

  Reviewer->>UI: Open proposed version
  Reviewer->>UI: Approve (with optional comment)
  UI->>API: ApproveVersion(versionId, comment)
  API->>DDB: Write approval record
  API->>SFN: SendTaskSuccess/Continue(approved)
  SFN->>DDB: Update version status=Approved
  API-->>UI: Approved (ready to publish)
```

---

## 4. Publish Schema Version (Compatibility Enforcement)

```mermaid
sequenceDiagram
  autonumber
  actor Publisher as Publisher (Workflow/System)
  participant API as Schema Mgmt API
  participant Glue as Glue Schema Registry
  participant S3 as S3
  participant DDB as DynamoDB
  participant EB as EventBridge

  Publisher->>API: Publish(versionId)
  API->>Glue: RegisterSchemaVersion(subject, schema)
  alt Compatible
    Glue-->>API: Success (glueSchemaVersionId)
    API->>S3: Write artifacts (pretty schema, docs, diff)
    API->>DDB: Update status=Published + glueSchemaVersionId
    API->>EB: Emit SchemaPublished event
  else Incompatible
    Glue-->>API: Reject (compatibility violation)
    API->>DDB: Update status=Rejected + reason
    API-->>Publisher: Publish failed (reason)
  end
```

---

## 5. Preflight Compatibility Check (Staging Registry Pattern)

```mermaid
sequenceDiagram
  autonumber
  actor Author as Schema Author
  participant UI as Angular UI
  participant API as Schema Mgmt API
  participant GlueStg as Glue Registry (Staging)
  participant DDB as DynamoDB

  Author->>UI: Propose schema change
  UI->>API: PreflightCompatibility(versionId)
  API->>GlueStg: RegisterSchemaVersion(subject, candidateSchema)
  alt Compatible
    GlueStg-->>API: Success (stagingVersionId)
    API->>DDB: Record preflight=PASS + stagingVersionId
    API-->>UI: Compatibility PASS (safe to proceed)
  else Incompatible
    GlueStg-->>API: Reject (violation details)
    API->>DDB: Record preflight=FAIL + details
    API-->>UI: Compatibility FAIL (show issues)
  end
```

---

## 6. Deprecate Published Version

```mermaid
sequenceDiagram
  autonumber
  actor Admin as Admin/Owner
  participant UI as Angular UI
  participant API as Schema Mgmt API
  participant DDB as DynamoDB
  participant EB as EventBridge

  Admin->>UI: Deprecate version
  UI->>API: DeprecateVersion(versionId, reason)
  API->>DDB: Update status=Deprecated + reason
  API->>EB: Emit SchemaDeprecated event
  API-->>UI: Deprecated
```

---

## 7. Fetch Latest Published Version (Consumer)

```mermaid
sequenceDiagram
  autonumber
  actor Consumer as Consumer Service
  participant API as Schema Mgmt API
  participant DDB as DynamoDB

  Consumer->>API: GetLatestPublished(subject)
  API->>DDB: Query latest status=Published
  DDB-->>API: version metadata + content location
  API-->>Consumer: schema content + version info
```

---

## 8. Promote Inline Object → Reusable Type ($defs/$ref)

```mermaid
sequenceDiagram
  autonumber
  actor Author as Schema Author
  participant UI as Angular UI
  participant Store as AST Store

  Author->>UI: Select inline object node
  Author->>UI: Click "Promote to reusable type"
  UI->>Store: Create $defs[TypeName] from node
  UI->>Store: Replace node with $ref to $defs[TypeName]
  Store-->>UI: Tree updates (ref link + types library update)
```

---

## 9. Resolve $ref for Display (Cycle-safe Expansion)

```mermaid
sequenceDiagram
  autonumber
  participant Tree as Schema Tree Renderer
  participant Store as AST Store

  Tree->>Store: Request node children (nodeId)
  alt Node is RefNode
    Store-->>Tree: Return refTargetId
    Tree->>Store: Request definition children (refTargetId, visitedSet)
    alt Cycle detected
      Store-->>Tree: CycleDetected
      Tree-->>Tree: Render '…cycle detected' indicator
    else No cycle
      Store-->>Tree: Return definition subtree (read-only)
      Tree-->>Tree: Render expanded preview
    end
  else Regular node
    Store-->>Tree: Return children
    Tree-->>Tree: Render subtree
  end
```

---

## 10. Optional: Link Schema Change to MKS Change Request

```mermaid
sequenceDiagram
  autonumber
  actor Author as Schema Author
  participant UI as Angular UI
  participant API as Schema Mgmt API
  participant MKS as MKS / ALM Tool
  participant DDB as DynamoDB

  Author->>UI: Enter MKS CR ID
  UI->>API: LinkChangeRequest(versionId, mksCrId)
  API->>MKS: Validate CR exists (optional)
  MKS-->>API: CR status/metadata
  API->>DDB: Persist externalTraceId=mksCrId
  API-->>UI: Linked successfully
```

---

## Notes
- These flows assume the platform is the **governance layer** and AWS Glue is the **contract compatibility authority**.
- In staging preflight, registration is used as a compatibility check without affecting production.

