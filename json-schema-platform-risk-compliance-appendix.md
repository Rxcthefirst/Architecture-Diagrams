# JSON Schema Platform – Risk & Compliance Appendix

## Purpose
This appendix provides risk, security, compliance, and audit considerations for the JSON Schema Management Platform. It is intended for architecture review boards, governance stakeholders, and security/compliance teams.

---

## 1. Risk Register

| ID | Risk | Description | Impact | Likelihood | Mitigation | Owner |
|---|---|---|---|---|---|---|
| R-001 | Breaking contract changes | Incompatible schema evolution breaks consumers. | High | Medium | Enforce compatibility at publish boundary via Glue; staged preflight checks; approval workflow; deprecation policies. | Platform |
| R-002 | Schema sprawl / duplication | Many similar schemas proliferate without reuse. | Medium | Medium | Domain ownership; tagging; templates; reuse via `$defs/$ref`; review gates. | Governance |
| R-003 | Unauthorized schema publication | Users publish changes without proper authority. | High | Low | RBAC/ABAC; domain-scoped permissions; publish requires workflow approval; least-privileged publish role. | Security |
| R-004 | Audit gaps | Incomplete traceability for approvals and changes. | High | Low | Append-only audit events; CloudTrail; immutable artifacts; approval records required to publish. | Compliance |
| R-005 | Data leakage in artifacts | Sensitive samples or payloads stored in logs/S3. | Medium | Medium | Redaction policies; forbid PII in examples; S3 bucket policy; encryption; scanning. | Security |
| R-006 | Workflow bottleneck | Review/approval slows delivery. | Medium | Medium | Policy-based routing; SLA alerts; reviewer delegation; emergency path with additional audit. | Governance |
| R-007 | Dependency on AWS service limits | Throttling/quotas impact publish flows. | Medium | Low | Backoff/retry; async publish jobs; request quota increases; monitoring. | Platform |
| R-008 | Ref cycles confuse users | `$ref` cycles cause confusing UI and infinite expansion. | Low | Medium | Cycle detection; read-only expansion previews; "go to definition" navigation. | UX |
| R-009 | Inconsistent schema standards | Teams use different JSON Schema drafts and conventions. | Medium | Medium | Normalize to Draft 2020-12; lint rules; org conventions; migration tooling. | Platform |
| R-010 | Vendor lock-in concerns | Glue chosen as contract authority. | Low | Medium | Abstract registry interface; export all schemas; maintain portability plan. | Architecture |

---

## 2. Security Requirements (Control Objectives)

### 2.1 Identity and Authentication
- Use enterprise SSO (OIDC/SAML) for UI access.
- Service-to-service calls use IAM roles and short-lived credentials.

### 2.2 Authorization
- Enforce role-based and attribute-based access:
  - `schema:read`, `schema:write`, `schema:approve`, `schema:publish`, `schema:admin`
- Scope permissions by domain/namespace (e.g., `loan/*`, `customer/*`).
- Require explicit approval roles for publication.

### 2.3 Data Protection
- Encrypt data at rest using KMS (S3, DynamoDB, Glue registry).
- Enforce TLS for all transit.
- Separate KMS keys per environment and per data classification if needed.

### 2.4 Secrets Management
- Store secrets (if any) in AWS Secrets Manager or SSM Parameter Store.
- Prefer IAM roles over static credentials.

### 2.5 Logging and Monitoring
- Centralize logs in CloudWatch (structured JSON).
- Apply redaction filters for PII and secrets.
- Alert on anomalous publish patterns.

---

## 3. Auditability & Evidence

### 3.1 Audit Event Coverage
The platform should produce immutable audit events for:
- Draft created/updated
- Proposal submitted
- Approval granted/denied
- Publish success/failure (including compatibility error reason)
- Deprecation actions
- Policy changes (compat mode, role changes)

### 3.2 Evidence Sources
- **CloudTrail**: API-level evidence for AWS actions (Glue, S3, DynamoDB, IAM).
- **Platform Audit Log**: append-only audit events (DynamoDB or S3 log stream).
- **Approval Records**: structured records including approver identity, timestamp, decision.
- **Artifacts**: immutable snapshots of schema versions and diffs stored in S3.

### 3.3 Retention
- Audit logs retained per organizational policy (e.g., 1–7 years).
- Artifacts retained at least as long as schemas are in use plus deprecation window.
- Enable DynamoDB PITR where appropriate.

---

## 4. Compliance Alignment (Typical Enterprise Expectations)

### 4.1 Least Privilege
- Publish operations require a narrowly scoped role.
- Separate execution roles for:
  - read-only operations
  - draft authoring
  - approval actions
  - publishing to Glue

### 4.2 Segregation of Duties
- Authors should not approve their own changes for regulated domains.
- Support multi-approver policies for high-impact subjects.

### 4.3 Change Management Integration
- Optionally require an external change request ID (e.g., MKS) before publish.
- Persist externalTraceId in version metadata and audit logs.

### 4.4 Environment Controls
- Production publication restricted to approved identities/roles.
- Separate registries per environment.
- Promotion between environments is controlled and audited.

---

## 5. Data Handling & Privacy

### 5.1 Sensitive Data Restrictions
- Do not store real production payloads as examples.
- Provide a safe example generator to avoid PII.
- Scan artifacts for banned patterns where feasible.

### 5.2 Logging Restrictions
- Avoid logging schema content by default.
- Store only hashes and identifiers in logs unless explicitly needed.

---

## 6. Availability, Reliability, and DR

### 6.1 Availability Objectives (Illustrative)
- API availability: 99.9% (multi-AZ services)
- Publish flow: designed to be retryable and idempotent

### 6.2 DR Strategy
- S3 versioning enabled; optional cross-region replication.
- DynamoDB PITR enabled.
- Periodic export snapshot of Glue registry to S3 (portability and DR).

---

## 7. Abuse Scenarios & Controls

| Scenario | Risk | Control |
|---|---|---|
| Mass schema creation | Resource abuse and sprawl | Rate limiting; quotas per domain/user; governance review |
| Malicious schema payload | SSRF / injection via refs | Restrict external `$ref`; sanitize inputs; allowlist libraries |
| Privilege escalation | Unauthorized publish | ABAC + approval workflows; CloudTrail alerts |
| Data exfiltration | Sensitive data in artifacts | Redaction; scanning; bucket policies; least privilege |

---

## 8. Open Questions for Review Boards
- Required retention period for artifacts and audit logs?
- Do regulated domains require dual-approval or SoD enforcement?
- Are external `$ref` sources allowed, and if so what allowlist is required?
- What are the required RTO/RPO targets?
- Are there mandatory integration points (MKS, Jira, ServiceNow)?

---

## 9. Summary
This appendix identifies key risks and documents the security/compliance controls expected for an enterprise-grade schema management platform. The central technical risk—breaking schema changes—is mitigated through compatibility enforcement and governance controls, with audit evidence available through both platform logs and AWS-native trails.
