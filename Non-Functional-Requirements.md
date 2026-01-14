# Non-Functional Requirements (NFRs)
## JSON Schema & Semantic Data Platform

---

## 1. Purpose

This document specifies the non-functional requirements (quality attributes) for the JSON Schema & Semantic Data Platform. These requirements define how the system should behave in terms of performance, scalability, security, and other operational characteristics.

---

## 2. Performance Requirements

### 2.1 Response Time

| Operation | Target (p50) | Target (p95) | Target (p99) | Max |
|-----------|--------------|--------------|--------------|-----|
| **API: Fetch schema** | 100ms | 300ms | 500ms | 1000ms |
| **API: Save draft** | 200ms | 500ms | 800ms | 1500ms |
| **API: Publish schema** | 500ms | 1500ms | 3000ms | 5000ms |
| **UI: Initial page load** | 1000ms | 2000ms | 3000ms | 5000ms |
| **UI: Schema tree render** | 100ms | 300ms | 500ms | 1000ms |
| **ETL: Full pipeline** | 30 min | 1 hour | 1.5 hours | 2 hours |
| **Schema Factory: Generate** | 500ms | 1500ms | 3000ms | 5000ms |
| **SPARQL: Single entity query** | 50ms | 200ms | 500ms | 1000ms |

### 2.2 Throughput

| Metric | Target | Peak |
|--------|--------|------|
| API requests per second | 100 | 500 |
| Concurrent UI users | 50 | 200 |
| Schema publications per hour | 50 | 200 |
| ETL entities per run | 10,000 | 50,000 |

### 2.3 Resource Utilization

| Resource | Normal | Warning | Critical |
|----------|--------|---------|----------|
| Lambda memory | < 60% | 60-80% | > 80% |
| DynamoDB consumed capacity | < 70% | 70-85% | > 85% |
| RDF store memory | < 70% | 70-85% | > 85% |
| S3 storage | N/A | N/A | Quota alert |

---

## 3. Scalability Requirements

### 3.1 Data Volume Projections

| Metric | Year 1 | Year 3 | Year 5 |
|--------|--------|--------|--------|
| Schema subjects | 500 | 2,000 | 5,000 |
| Schema versions (total) | 2,500 | 15,000 | 50,000 |
| RDF triples | 1M | 5M | 15M |
| API requests/day | 50K | 200K | 500K |
| Active users | 100 | 300 | 500 |

### 3.2 Scaling Strategy

| Component | Scaling Type | Trigger |
|-----------|--------------|---------|
| API Gateway | Automatic | N/A |
| Lambda functions | Automatic (concurrency) | Invocation rate |
| DynamoDB | Auto-scaling | Consumed capacity |
| RDF Store | Vertical → Horizontal | Query latency, data size |
| S3 | Unlimited | N/A |
| Glue Registry | Managed | N/A (monitor quotas) |

### 3.3 Elasticity Requirements

- **Scale out**: < 2 minutes to handle 5x traffic spike
- **Scale in**: Within 15 minutes after load decrease
- **Zero downtime**: Scaling operations must not impact availability

---

## 4. Availability Requirements

### 4.1 Uptime Targets

| Service | Availability | Downtime/Month | Downtime/Year |
|---------|--------------|----------------|---------------|
| Schema Management API | 99.9% | 43 min | 8.7 hours |
| Schema Factory API | 99.5% | 3.6 hours | 43 hours |
| Angular UI | 99.9% | 43 min | 8.7 hours |
| ETL Pipeline | 99% (per run) | N/A | N/A |

### 4.2 Redundancy Requirements

| Component | Redundancy Strategy |
|-----------|---------------------|
| API (Lambda) | Multi-AZ (automatic) |
| DynamoDB | Multi-AZ (automatic) |
| S3 | Multi-AZ (automatic), optional cross-region |
| RDF Store | Multi-AZ deployment, read replicas |
| UI (CloudFront) | Global edge distribution |

### 4.3 Failover Requirements

| Scenario | Detection | Failover Time |
|----------|-----------|---------------|
| Single Lambda failure | Automatic | Immediate (retry) |
| AZ failure | Automatic | < 1 minute |
| RDF store failure | Health check | < 5 minutes |
| Region failure | Manual trigger | < 4 hours (RTO) |

---

## 5. Reliability Requirements

### 5.1 Data Durability

| Data Type | Durability Target | Mechanism |
|-----------|-------------------|-----------|
| Schema content | 99.999999999% (11 9s) | S3 standard |
| Schema metadata | 99.999999999% (11 9s) | DynamoDB + PITR |
| RDF triples | 99.99% | Database backups |
| Audit logs | 99.999999999% (11 9s) | S3 + CloudTrail |

### 5.2 Data Integrity

- **Transactional consistency**: Schema publication is atomic
- **Idempotency**: All write operations are idempotent
- **Validation**: All schemas validated before storage
- **Checksums**: Artifact integrity verified on retrieval

### 5.3 Fault Tolerance

| Failure Mode | Expected Behavior |
|--------------|-------------------|
| Glue unavailable | Queue publications, retry with backoff |
| DynamoDB throttling | Automatic retry, backoff |
| SPARQL query timeout | Return cached result or error gracefully |
| Invalid input | Reject with clear error, no side effects |

---

## 6. Security Requirements

### 6.1 Authentication

| Interface | Mechanism | Session Duration |
|-----------|-----------|------------------|
| UI (user) | OIDC/SAML via enterprise IdP | 8 hours |
| API (user) | JWT bearer token | 1 hour |
| API (service) | IAM role | Temporary credentials |
| Internal services | IAM role | Temporary credentials |

### 6.2 Authorization

| Permission | Description | Enforced At |
|------------|-------------|-------------|
| `schema:read` | View schema content | API Gateway + Lambda |
| `schema:write` | Create/edit drafts | Lambda |
| `schema:propose` | Submit for review | Lambda |
| `schema:approve` | Approve changes | Lambda + Step Functions |
| `schema:publish` | Publish to registry | Lambda |
| `schema:admin` | Manage policies | Lambda |

**Scope Enforcement**:
- Permissions scoped by domain (e.g., `customer/*`, `loan/*`)
- ABAC rules for cross-domain access

### 6.3 Data Protection

| Data State | Protection |
|------------|------------|
| At rest (S3) | AES-256 (SSE-S3 or SSE-KMS) |
| At rest (DynamoDB) | AES-256 (KMS) |
| At rest (RDF) | Encryption at storage layer |
| In transit | TLS 1.2+ (enforced) |
| In memory | Short-lived, no persistence |

### 6.4 Secrets Management

- No hardcoded secrets in code or configuration
- Secrets stored in AWS Secrets Manager
- IAM roles preferred over static credentials
- Secret rotation: 90-day maximum

### 6.5 Security Compliance

| Standard | Requirement |
|----------|-------------|
| OWASP Top 10 | All vulnerabilities addressed |
| SOC 2 | Audit controls in place |
| GDPR | No PII in schema examples |
| Internal policy | Annual penetration test |

---

## 7. Maintainability Requirements

### 7.1 Code Quality

| Metric | Target |
|--------|--------|
| Unit test coverage | ≥ 80% |
| Cyclomatic complexity | ≤ 10 per function |
| Code duplication | < 5% |
| Documentation coverage | All public APIs |

### 7.2 Deployment

| Requirement | Target |
|-------------|--------|
| Deployment frequency | On-demand (CI/CD) |
| Deployment duration | < 15 minutes |
| Rollback time | < 5 minutes |
| Zero-downtime deployments | Required |

### 7.3 Dependency Management

- All dependencies version-locked
- Security scanning on every build
- Maximum dependency age: 1 year (exceptions documented)
- No dependencies with known critical CVEs

---

## 8. Usability Requirements

### 8.1 UI Performance Perception

| Interaction | Feedback Requirement |
|-------------|---------------------|
| Button click | Visual feedback < 100ms |
| Form submission | Loading indicator immediately |
| Long operation | Progress indicator + estimate |
| Error | Clear message + recovery action |

### 8.2 Accessibility

| Standard | Target |
|----------|--------|
| WCAG | 2.1 Level AA |
| Keyboard navigation | 100% functionality |
| Screen reader | Compatible with NVDA, JAWS |
| Color contrast | 4.5:1 minimum |

### 8.3 Browser Support

| Browser | Minimum Version |
|---------|-----------------|
| Chrome | Last 2 major versions |
| Firefox | Last 2 major versions |
| Edge | Last 2 major versions |
| Safari | Last 2 major versions |

---

## 9. Interoperability Requirements

### 9.1 Standards Compliance

| Standard | Compliance |
|----------|------------|
| JSON Schema | Draft 2020-12 |
| RDF | W3C RDF 1.1 |
| OWL | W3C OWL 2 |
| SPARQL | W3C SPARQL 1.1 |
| REST API | OpenAPI 3.0 |

### 9.2 Integration Interfaces

| Interface | Format | Protocol |
|-----------|--------|----------|
| Schema API | JSON | REST/HTTPS |
| Events | CloudEvents JSON | EventBridge |
| Collibra export | JSON | File/HTTPS |
| Glue Registry | AWS API | AWS SDK |

### 9.3 Data Portability

- All schemas exportable as JSON Schema files
- RDF exportable in Turtle, JSON-LD, N-Triples
- No proprietary lock-in for schema content

---

## 10. Compliance & Audit Requirements

### 10.1 Audit Trail

| Event | Captured Data | Retention |
|-------|---------------|-----------|
| Schema created | Who, when, what | 7 years |
| Schema modified | Who, when, diff | 7 years |
| Schema published | Who, when, version | 7 years |
| Access attempt | Who, what, result | 1 year |
| Configuration change | Who, when, diff | 7 years |

### 10.2 Compliance Controls

| Control | Implementation |
|---------|----------------|
| Segregation of duties | Authors cannot approve own changes |
| Least privilege | Role-based, scoped permissions |
| Change management | Approval workflow required |
| Data retention | Configurable per policy |

---

## 11. Operational Requirements

### 11.1 Monitoring

- All components emit standard metrics
- Centralized logging with correlation IDs
- Distributed tracing for request flows
- Dashboards for each persona (ops, dev, business)

### 11.2 Alerting

- Alerts for SLO violations
- Alerts for security anomalies
- Alerts for capacity thresholds
- Escalation paths defined

### 11.3 Backup & Recovery

| Data | Backup Frequency | Recovery Point Objective |
|------|------------------|--------------------------|
| Schemas | Continuous (PITR) | 5 minutes |
| Metadata | Continuous (PITR) | 5 minutes |
| RDF store | Daily | 24 hours |
| Logs | Continuous | 0 (streaming) |

---

## 12. Environmental Requirements

### 12.1 Deployment Environments

| Environment | Purpose | Parity with Prod |
|-------------|---------|------------------|
| Local | Development | Functional |
| Dev | Integration testing | High |
| Staging | Pre-production | Near-identical |
| Production | Live system | N/A |

### 12.2 Infrastructure as Code

- All infrastructure defined in Terraform/CDK
- Version controlled with code
- Changes reviewed via PR process
- Drift detection enabled

---

## 13. Constraints

### 13.1 Technology Constraints

| Constraint | Rationale |
|------------|-----------|
| AWS only | Organizational standard |
| JSON Schema (not Avro/Protobuf) | Consumer compatibility |
| Angular (UI) | Team expertise |
| Python or Node.js (backend) | Team expertise |

### 13.2 Resource Constraints

| Constraint | Limit |
|------------|-------|
| AWS account limits | Per-service quotas apply |
| Glue registry limits | 1000 schemas per registry |
| Budget | Defined per environment |

---

## 14. Acceptance Criteria

All non-functional requirements are verified through:

1. **Performance testing**: Load tests validate response time targets
2. **Security assessment**: Penetration test validates security requirements
3. **Availability testing**: Chaos engineering validates redundancy
4. **Audit**: Compliance review validates audit requirements
5. **Accessibility audit**: Third-party WCAG assessment

---

## 15. Summary

This NFR specification ensures the platform delivers:
- ✅ Responsive user experience (sub-second API responses)
- ✅ Enterprise-grade availability (99.9%)
- ✅ Secure by design (encryption, RBAC, audit)
- ✅ Scalable for growth (5-year projections)
- ✅ Compliant with enterprise standards
- ✅ Maintainable and operable

---

## Related Documents

- [System Integration Architecture](System-Integration-Architecture.md)
- [Operations & Observability Guide](Operations-Observability-Guide.md)
- [Risk & Compliance Appendix](json-schema-platform-risk-compliance-appendix.md)
- [Testing Strategy](Testing-Strategy.md)
