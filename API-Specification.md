# API Specification
## JSON Schema & Semantic Data Platform

---

## 1. Overview

This document provides the REST API specification for the Schema Management Platform. It defines all endpoints, request/response schemas, authentication, and error handling conventions.

**Base URL**: `https://api.schema-platform.example.com/v1`

**API Version**: `1.0.0`

**Specification Format**: This document follows OpenAPI 3.0 conventions and can be converted to a formal OpenAPI specification.

---

## 2. Authentication & Authorization

### 2.1 Authentication Methods

| Method | Use Case | Header |
|--------|----------|--------|
| Bearer Token (JWT) | User API access | `Authorization: Bearer <token>` |
| IAM Signature | Service-to-service | AWS SigV4 |

### 2.2 Token Structure

```json
{
  "sub": "user@example.com",
  "iss": "https://idp.example.com",
  "aud": "schema-platform",
  "exp": 1736784000,
  "iat": 1736780400,
  "scope": ["schema:read", "schema:write"],
  "domains": ["customer", "loan"]
}
```

### 2.3 Permission Scopes

| Scope | Description |
|-------|-------------|
| `schema:read` | Read schema content and metadata |
| `schema:write` | Create and edit draft schemas |
| `schema:propose` | Submit schemas for review |
| `schema:approve` | Approve proposed schemas |
| `schema:publish` | Publish approved schemas |
| `schema:admin` | Manage platform configuration |

---

## 3. Common Conventions

### 3.1 Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | Bearer token or IAM signature |
| `Content-Type` | Yes (POST/PUT) | `application/json` |
| `Accept` | No | `application/json` (default) |
| `X-Request-ID` | No | Client-provided correlation ID |

### 3.2 Response Headers

| Header | Description |
|--------|-------------|
| `X-Request-ID` | Echo of client ID or server-generated |
| `X-RateLimit-Limit` | Rate limit ceiling |
| `X-RateLimit-Remaining` | Remaining requests in window |
| `X-RateLimit-Reset` | Timestamp when limit resets |

### 3.3 Pagination

All list endpoints support pagination:

```
GET /schemas?limit=20&cursor=eyJsYXN0SWQiOiIxMjMifQ==
```

**Response**:
```json
{
  "items": [...],
  "nextCursor": "eyJsYXN0SWQiOiIxNDMifQ==",
  "hasMore": true
}
```

### 3.4 Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Schema validation failed",
    "details": [
      {
        "field": "properties.name",
        "issue": "Missing required property 'type'"
      }
    ],
    "requestId": "req-123-456"
  }
}
```

### 3.5 Standard Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `VALIDATION_ERROR` | Invalid request payload |
| 400 | `INVALID_SCHEMA` | Schema syntax invalid |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `CONFLICT` | Resource state conflict |
| 409 | `COMPATIBILITY_ERROR` | Schema compatibility violation |
| 422 | `BUSINESS_RULE_VIOLATION` | Business rule not satisfied |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server error |
| 503 | `SERVICE_UNAVAILABLE` | Downstream service unavailable |

---

## 4. Schema Subjects API

### 4.1 List Subjects

**GET** `/schemas`

List all schema subjects with optional filtering.

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `domain` | string | Filter by domain |
| `source` | string | Filter by source (`collibra_etl`, `user_authored`) |
| `owner` | string | Filter by owner |
| `limit` | integer | Page size (default: 20, max: 100) |
| `cursor` | string | Pagination cursor |

**Response** `200 OK`:
```json
{
  "items": [
    {
      "subjectId": "gov.customer.Customer",
      "domain": "customer",
      "owner": "data-governance@example.com",
      "source": "collibra_etl",
      "compatibilityMode": "BACKWARD",
      "latestVersion": "1.2.0",
      "latestStatus": "PUBLISHED",
      "createdAt": "2025-06-15T10:30:00Z",
      "updatedAt": "2026-01-10T14:20:00Z"
    }
  ],
  "nextCursor": "eyJsYXN0SWQiOiJnb3YuY3VzdG9tZXIuQ3VzdG9tZXIifQ==",
  "hasMore": true
}
```

---

### 4.2 Create Subject

**POST** `/schemas`

Create a new schema subject.

**Request Body**:
```json
{
  "subjectId": "app.orderservice.OrderCreated",
  "domain": "orders",
  "owner": "order-team@example.com",
  "compatibilityMode": "BACKWARD",
  "description": "Event schema for order creation",
  "tags": ["event", "orders"]
}
```

**Response** `201 Created`:
```json
{
  "subjectId": "app.orderservice.OrderCreated",
  "domain": "orders",
  "owner": "order-team@example.com",
  "source": "user_authored",
  "compatibilityMode": "BACKWARD",
  "createdAt": "2026-01-13T10:30:00Z",
  "createdBy": "user@example.com"
}
```

**Errors**:
- `409 CONFLICT`: Subject already exists

---

### 4.3 Get Subject

**GET** `/schemas/{subjectId}`

Retrieve subject metadata.

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `subjectId` | string | Subject identifier (URL-encoded) |

**Response** `200 OK`:
```json
{
  "subjectId": "gov.customer.Customer",
  "domain": "customer",
  "owner": "data-governance@example.com",
  "source": "collibra_etl",
  "compatibilityMode": "BACKWARD",
  "description": "Customer entity from governance",
  "tags": ["entity", "customer", "governed"],
  "latestVersion": "1.2.0",
  "latestStatus": "PUBLISHED",
  "versionCount": 5,
  "collibraAssetId": "collibra:CUST_ENTITY_001",
  "rdfClassIRI": "http://example.com/model/Customer",
  "createdAt": "2025-06-15T10:30:00Z",
  "updatedAt": "2026-01-10T14:20:00Z"
}
```

---

### 4.4 Update Subject

**PATCH** `/schemas/{subjectId}`

Update subject metadata (not schema content).

**Request Body**:
```json
{
  "owner": "new-owner@example.com",
  "compatibilityMode": "FULL",
  "tags": ["entity", "customer", "governed", "critical"]
}
```

**Response** `200 OK`: Updated subject object

---

### 4.5 Delete Subject

**DELETE** `/schemas/{subjectId}`

Delete a subject (only if no published versions).

**Response** `204 No Content`

**Errors**:
- `409 CONFLICT`: Cannot delete subject with published versions

---

## 5. Schema Versions API

### 5.1 List Versions

**GET** `/schemas/{subjectId}/versions`

List all versions of a subject.

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status |
| `limit` | integer | Page size |
| `cursor` | string | Pagination cursor |

**Response** `200 OK`:
```json
{
  "items": [
    {
      "versionId": "v-123-456",
      "versionNumber": "1.2.0",
      "status": "PUBLISHED",
      "source": "collibra_etl",
      "glueVersionId": "glue-789",
      "createdAt": "2026-01-10T14:20:00Z",
      "publishedAt": "2026-01-10T15:00:00Z"
    },
    {
      "versionId": "v-123-455",
      "versionNumber": "1.1.0",
      "status": "PUBLISHED",
      "source": "collibra_etl",
      "glueVersionId": "glue-788",
      "createdAt": "2025-12-01T10:00:00Z",
      "publishedAt": "2025-12-01T11:00:00Z"
    }
  ],
  "hasMore": false
}
```

---

### 5.2 Create Draft Version

**POST** `/schemas/{subjectId}/versions`

Create a new draft version.

**Request Body**:
```json
{
  "schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
      "customerId": {
        "type": "string",
        "description": "Unique customer identifier"
      },
      "fullName": {
        "type": "string",
        "maxLength": 200
      }
    },
    "required": ["customerId"]
  },
  "changelog": "Added fullName field"
}
```

**Response** `201 Created`:
```json
{
  "versionId": "v-123-457",
  "versionNumber": "1.3.0-draft",
  "status": "DRAFT",
  "createdAt": "2026-01-13T10:30:00Z",
  "createdBy": "user@example.com"
}
```

---

### 5.3 Get Version

**GET** `/schemas/{subjectId}/versions/{versionId}`

Retrieve a specific version.

**Response** `200 OK`:
```json
{
  "versionId": "v-123-456",
  "subjectId": "gov.customer.Customer",
  "versionNumber": "1.2.0",
  "status": "PUBLISHED",
  "source": "collibra_etl",
  "glueVersionId": "glue-789",
  "schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": { ... }
  },
  "metadata": {
    "collibraAssetId": "collibra:CUST_ENTITY_001",
    "rdfClassIRI": "http://example.com/model/Customer",
    "etlRunId": "etl-run-2026-01-10"
  },
  "changelog": "Updated from Collibra governance",
  "createdAt": "2026-01-10T14:20:00Z",
  "createdBy": "etl-service",
  "publishedAt": "2026-01-10T15:00:00Z",
  "publishedBy": "workflow-system"
}
```

---

### 5.4 Get Latest Published Version

**GET** `/schemas/{subjectId}/versions/latest`

Retrieve the latest published version (consumer shortcut).

**Response** `200 OK`: Same as Get Version

**Errors**:
- `404 NOT_FOUND`: No published version exists

---

### 5.5 Update Draft Version

**PUT** `/schemas/{subjectId}/versions/{versionId}`

Update a draft version (only DRAFT status).

**Request Body**:
```json
{
  "schema": { ... },
  "changelog": "Updated description"
}
```

**Response** `200 OK`: Updated version object

**Errors**:
- `409 CONFLICT`: Cannot update non-draft version

---

### 5.6 Delete Draft Version

**DELETE** `/schemas/{subjectId}/versions/{versionId}`

Delete a draft version.

**Response** `204 No Content`

**Errors**:
- `409 CONFLICT`: Cannot delete non-draft version

---

## 6. Workflow API

### 6.1 Propose Version

**POST** `/schemas/{subjectId}/versions/{versionId}/propose`

Submit a draft for review.

**Request Body**:
```json
{
  "message": "Ready for review - added new customer fields",
  "externalTraceId": "JIRA-1234"
}
```

**Response** `200 OK`:
```json
{
  "versionId": "v-123-457",
  "status": "PROPOSED",
  "workflowId": "wf-789",
  "proposedAt": "2026-01-13T10:35:00Z",
  "proposedBy": "user@example.com"
}
```

---

### 6.2 Approve Version

**POST** `/schemas/{subjectId}/versions/{versionId}/approve`

Approve a proposed version.

**Request Body**:
```json
{
  "comment": "Looks good, approved for publication"
}
```

**Response** `200 OK`:
```json
{
  "versionId": "v-123-457",
  "status": "APPROVED",
  "approvedAt": "2026-01-13T11:00:00Z",
  "approvedBy": "reviewer@example.com"
}
```

**Errors**:
- `403 FORBIDDEN`: User cannot approve own submission
- `409 CONFLICT`: Version not in PROPOSED status

---

### 6.3 Reject Version

**POST** `/schemas/{subjectId}/versions/{versionId}/reject`

Reject a proposed version.

**Request Body**:
```json
{
  "reason": "Missing required description for new field"
}
```

**Response** `200 OK`:
```json
{
  "versionId": "v-123-457",
  "status": "REJECTED",
  "rejectedAt": "2026-01-13T11:00:00Z",
  "rejectedBy": "reviewer@example.com",
  "rejectionReason": "Missing required description for new field"
}
```

---

### 6.4 Publish Version

**POST** `/schemas/{subjectId}/versions/{versionId}/publish`

Publish an approved version to Glue registry.

**Response** `200 OK`:
```json
{
  "versionId": "v-123-457",
  "status": "PUBLISHED",
  "glueVersionId": "glue-790",
  "versionNumber": "1.3.0",
  "publishedAt": "2026-01-13T11:05:00Z",
  "publishedBy": "workflow-system"
}
```

**Errors**:
- `409 COMPATIBILITY_ERROR`: Schema incompatible with previous version

---

### 6.5 Deprecate Version

**POST** `/schemas/{subjectId}/versions/{versionId}/deprecate`

Mark a published version as deprecated.

**Request Body**:
```json
{
  "reason": "Superseded by v2.0.0",
  "sunsetDate": "2026-06-01"
}
```

**Response** `200 OK`:
```json
{
  "versionId": "v-123-456",
  "status": "DEPRECATED",
  "deprecatedAt": "2026-01-13T12:00:00Z",
  "deprecationReason": "Superseded by v2.0.0",
  "sunsetDate": "2026-06-01"
}
```

---

## 7. Compatibility API

### 7.1 Check Compatibility

**POST** `/schemas/{subjectId}/compatibility`

Check if a schema is compatible with the latest published version.

**Request Body**:
```json
{
  "schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": { ... }
  }
}
```

**Response** `200 OK`:
```json
{
  "compatible": true,
  "compatibilityMode": "BACKWARD",
  "checkedAgainst": "1.2.0"
}
```

**Response** `200 OK` (incompatible):
```json
{
  "compatible": false,
  "compatibilityMode": "BACKWARD",
  "checkedAgainst": "1.2.0",
  "violations": [
    {
      "type": "FIELD_REMOVED",
      "path": "#/properties/legacyField",
      "message": "Removing a field breaks backward compatibility"
    }
  ]
}
```

---

## 8. Validation API

### 8.1 Validate Schema

**POST** `/validate/schema`

Validate a JSON Schema without creating a version.

**Request Body**:
```json
{
  "schema": { ... }
}
```

**Response** `200 OK`:
```json
{
  "valid": true,
  "schemaVersion": "draft/2020-12"
}
```

**Response** `200 OK` (invalid):
```json
{
  "valid": false,
  "errors": [
    {
      "path": "#/properties/name/type",
      "message": "Invalid type: 'strin' is not a valid JSON Schema type"
    }
  ]
}
```

---

### 8.2 Validate Instance

**POST** `/validate/instance`

Validate a JSON instance against a published schema.

**Request Body**:
```json
{
  "subjectId": "gov.customer.Customer",
  "version": "latest",
  "instance": {
    "customerId": "C-12345",
    "fullName": "John Doe"
  }
}
```

**Response** `200 OK`:
```json
{
  "valid": true,
  "validatedAgainst": "gov.customer.Customer@1.2.0"
}
```

---

## 9. Search API

### 9.1 Search Schemas

**GET** `/search`

Full-text search across schemas.

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `q` | string | Search query |
| `domain` | string | Filter by domain |
| `tags` | string | Comma-separated tags |

**Response** `200 OK`:
```json
{
  "items": [
    {
      "subjectId": "gov.customer.Customer",
      "domain": "customer",
      "relevanceScore": 0.95,
      "matchedFields": ["description", "properties.customerId.description"]
    }
  ],
  "totalCount": 15
}
```

---

## 10. Audit API

### 10.1 Get Audit Log

**GET** `/schemas/{subjectId}/audit`

Retrieve audit history for a subject.

**Response** `200 OK`:
```json
{
  "items": [
    {
      "eventId": "evt-123",
      "eventType": "VERSION_PUBLISHED",
      "timestamp": "2026-01-10T15:00:00Z",
      "actor": "workflow-system",
      "details": {
        "versionId": "v-123-456",
        "versionNumber": "1.2.0"
      }
    },
    {
      "eventId": "evt-122",
      "eventType": "VERSION_APPROVED",
      "timestamp": "2026-01-10T14:45:00Z",
      "actor": "reviewer@example.com",
      "details": {
        "versionId": "v-123-456",
        "comment": "Approved"
      }
    }
  ]
}
```

---

## 11. System A Integration API

### 11.1 Submit Governed Schema

**POST** `/internal/governed-schemas`

Internal endpoint for System A (Schema Factory) to submit governed schemas.

**Authentication**: IAM service role only

**Request Body**:
```json
{
  "subjectId": "gov.customer.Customer",
  "schema": { ... },
  "metadata": {
    "collibraAssetId": "collibra:CUST_ENTITY_001",
    "rdfClassIRI": "http://example.com/model/Customer",
    "etlRunId": "etl-run-2026-01-13",
    "materializationGraph": "urn:graph:materialization:2026-01-13"
  },
  "autoPublish": true
}
```

**Response** `201 Created` or `200 OK`:
```json
{
  "subjectId": "gov.customer.Customer",
  "versionId": "v-123-458",
  "status": "PUBLISHED",
  "action": "CREATED_AND_PUBLISHED"
}
```

---

## 12. Rate Limits

| Endpoint Category | Limit | Window |
|-------------------|-------|--------|
| Read operations | 1000 | 1 minute |
| Write operations | 100 | 1 minute |
| Publish operations | 20 | 1 minute |
| Search | 50 | 1 minute |

---

## 13. Versioning Strategy

- API version in URL path: `/v1/`, `/v2/`
- Breaking changes require new major version
- Deprecated endpoints marked with `Deprecation` header
- Minimum 6-month deprecation notice

---

## Related Documents

- [System Integration Architecture](System-Integration-Architecture.md)
- [Non-Functional Requirements](Non-Functional-Requirements.md)
- [Testing Strategy](Testing-Strategy.md)
