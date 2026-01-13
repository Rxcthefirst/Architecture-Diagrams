# ADR-001: Schema Registry and Compatibility Enforcement Strategy

## Status
Accepted

## Date
2026-01-12

---

## Context

The platform requires a reliable mechanism to:
- Store versioned JSON Schemas
- Enforce schema evolution rules
- Prevent breaking changes to producer/consumer contracts
- Support enterprise governance, auditability, and controlled change

Several approaches were considered, including:
- Custom-built compatibility validation
- EventBridge Schemas
- Third-party schema registries
- AWS Glue Schema Registry

The system already targets AWS as its primary deployment environment.

---

## Decision

**AWS Glue Schema Registry** is selected as the authoritative system for:

- Canonical storage of published schema versions
- Compatibility enforcement (BACKWARD, FORWARD, FULL modes)
- Final contract acceptance during schema publication

The platform’s internal **Schema Management Service** is responsible for:
- Authoring workflows
- Draft and approval lifecycle
- Metadata, ownership, and governance
- Audit and traceability
- Pre-publish validation and staging checks

Compatibility enforcement occurs at the **publish boundary** via Glue registration.

---

## Alternatives Considered

### 1. Custom Compatibility Engine
**Rejected**
- High implementation complexity
- Risk of incorrect interpretation of JSON Schema evolution rules
- Ongoing maintenance burden

### 2. EventBridge Schemas
**Rejected as primary registry**
- Optimized for event discovery, not governance
- Limited lifecycle and approval semantics
- Not suitable as a general schema contract authority

### 3. Third-Party Registries (e.g., Confluent)
**Rejected**
- Adds operational and cost overhead
- Primarily optimized for Kafka ecosystems
- Less integrated with AWS IAM and governance

---

## Rationale

AWS Glue Schema Registry provides:
- Native JSON Schema support
- Built-in compatibility enforcement
- Immutable, versioned schema storage
- Tight integration with AWS security and auditing
- Clear separation between governance (platform) and enforcement (Glue)

This avoids re-implementing complex compatibility logic while maintaining architectural clarity.

---

## Consequences

### Positive
- Reduced risk of breaking changes
- Simplified platform logic
- Enterprise-grade auditability
- Clear contract boundary for consumers

### Negative
- Compatibility checks are tied to registration operations
- Requires staging or multi-registry strategy for pre-approval checks
- AWS dependency (acceptable given system constraints)

---

## Implementation Notes

- Use separate Glue registries per environment (dev/staging/prod)
- Perform preflight compatibility checks in staging before approval
- Treat Glue registration success as the authoritative publish signal
- Persist Glue schema version IDs in platform metadata

---

## Related Documents

- Systems Engineering Documentation
- Angular Schema Builder Implementation Plan
- ADR-002: Schema Versioning Strategy (future)
