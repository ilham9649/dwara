# API Contracts: Multi-Tenant SaaS Support

**Feature**: Multi-Tenant SaaS Support
**Date**: 2026-03-11
**Purpose**: Define REST API contracts for tenant management operations

## Overview

This document defines the REST API contracts for multi-tenant SaaS capabilities, including tenant CRUD operations, tenant management, and tenant-scoped resource operations.

## Base URL

```
https://bamf.example.com/api/v1
```

## Authentication

All API requests require authentication via JWT token. The token includes tenant context in claims:

```
{
  "sub": "user-id",
  "email": "user@example.com",
  "tenant_id": "tenant-uuid",
  "roles": ["admin", "developer"],
  "exp": 1234567890
}
```

Platform admin tokens have `tenant_id` set to `null` and include `is_platform_admin: true` claim.

## Common Response Codes

- `200 OK`: Successful GET request
- `201 Created`: Successful POST request
- `204 No Content`: Successful DELETE request
- `400 Bad Request`: Invalid request payload
- `401 Unauthorized`: Missing or invalid authentication
- `403 Forbidden`: Valid authentication but insufficient permissions
- `404 Not Found`: Resource not found for tenant
- `409 Conflict`: Resource already exists (e.g., duplicate tenant identifier)
- `422 Unprocessable Entity`: Validation error
- `500 Internal Server Error`: Server error

## Common Error Response Format

```json
{
  "error": {
    "code": "TENANT_NOT_FOUND",
    "message": "Tenant not found for current context",
    "details": {
      "tenant_id": "uuid-here"
    },
    "request_id": "correlation-id"
  }
}
```

---

## Tenant Management Endpoints (Platform Admin Only)

### Create Tenant

**Endpoint**: `POST /admin/tenants`

**Authentication**: Platform admin required

**Request Body**:
```json
{
  "identifier": "acme-corp",
  "name": "Acme Corporation",
  "logo_url": "https://example.com/logo.png",
  "branding": {
    "primary_color": "#3b82f6",
    "secondary_color": "#1e40af"
  },
  "domain": "bamf.acme.com",
  "sso_config": {
    "type": "oidc",
    "issuer": "https://acme.okta.com/oauth2/default",
    "client_id": "client-id",
    "client_secret": "client-secret"
  },
  "usage_limits": {
    "max_tunnels": 1000,
    "max_users": 100,
    "max_storage_gb": 1000
  }
}
```

**Response**: `201 Created`
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "name": "Acme Corporation",
  "status": "active",
  "created_at": "2026-03-11T10:00:00Z",
  "updated_at": "2026-03-11T10:00:00Z"
}
```

**Error Codes**:
- `TENANT_IDENTIFIER_EXISTS`: Tenant identifier already taken
- `DOMAIN_ALREADY_EXISTS`: Domain already assigned to another tenant

---

### List Tenants

**Endpoint**: `GET /admin/tenants`

**Authentication**: Platform admin required

**Query Parameters**:
- `status` (optional): Filter by status (`active`, `suspended`, `deleting`, `deleted`)
- `limit` (optional): Pagination limit (default 50, max 100)
- `offset` (optional): Pagination offset (default 0)

**Response**: `200 OK`
```json
{
  "tenants": [
    {
      "id": "tenant-uuid",
      "identifier": "acme-corp",
      "name": "Acme Corporation",
      "status": "active",
      "resource_count": 42,
      "active_tunnels": 15,
      "last_activity_at": "2026-03-11T09:30:00Z",
      "created_at": "2026-03-01T10:00:00Z"
    }
  ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

---

### Get Tenant

**Endpoint**: `GET /admin/tenants/{tenant_id}`

**Authentication**: Platform admin required

**Response**: `200 OK`
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "name": "Acme Corporation",
  "status": "active",
  "logo_url": "https://example.com/logo.png",
  "branding": {
    "primary_color": "#3b82f6"
  },
  "domain": "bamf.acme.com",
  "sso_config": {
    "type": "oidc",
    "issuer": "https://acme.okta.com/oauth2/default"
  },
  "usage_limits": {
    "max_tunnels": 1000,
    "max_users": 100
  },
  "usage": {
    "tunnels": 15,
    "users": 25,
    "storage_gb": 450
  },
  "created_at": "2026-03-01T10:00:00Z",
  "updated_at": "2026-03-11T10:00:00Z"
}
```

**Error Codes**:
- `TENANT_NOT_FOUND`: Tenant not found

---

### Update Tenant

**Endpoint**: `PATCH /admin/tenants/{tenant_id}`

**Authentication**: Platform admin required

**Request Body** (all fields optional):
```json
{
  "name": "Acme Corporation Updated",
  "logo_url": "https://example.com/new-logo.png",
  "branding": {
    "primary_color": "#ef4444"
  },
  "usage_limits": {
    "max_tunnels": 2000
  }
}
```

**Response**: `200 OK`
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "name": "Acme Corporation Updated",
  "status": "active",
  "updated_at": "2026-03-11T10:30:00Z"
}
```

**Error Codes**:
- `TENANT_NOT_FOUND`: Tenant not found
- `DOMAIN_ALREADY_EXISTS`: Domain already assigned to another tenant

---

### Suspend Tenant

**Endpoint**: `POST /admin/tenants/{tenant_id}/suspend`

**Authentication**: Platform admin required

**Request Body**:
```json
{
  "reason": "Terms of service violation"
}
```

**Response**: `200 OK`
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "status": "suspended",
  "suspended_at": "2026-03-11T10:30:00Z",
  "suspension_reason": "Terms of service violation"
}
```

**Error Codes**:
- `TENANT_NOT_FOUND`: Tenant not found
- `TENANT_ALREADY_SUSPENDED`: Tenant already suspended

---

### Resume Tenant

**Endpoint**: `POST /admin/tenants/{tenant_id}/resume`

**Authentication**: Platform admin required

**Response**: `200 OK`
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "status": "active",
  "resumed_at": "2026-03-11T11:00:00Z"
}
```

**Error Codes**:
- `TENANT_NOT_FOUND`: Tenant not found
- `TENANT_NOT_SUSPENDED`: Tenant is not suspended

---

### Delete Tenant

**Endpoint**: `DELETE /admin/tenants/{tenant_id}`

**Authentication**: Platform admin required

**Query Parameters**:
- `retention_days` (required): Data retention period (0, 30, or 90)

**Response**: `202 Accepted`
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "status": "deleting",
  "deleted_at": "2026-03-11T11:00:00Z",
  "retention_days": 30,
  "data_permanent_delete_at": "2026-04-10T11:00:00Z"
}
```

**Error Codes**:
- `TENANT_NOT_FOUND`: Tenant not found
- `TENANT_ALREADY_DELETING`: Tenant already being deleted

---

## Tenant User Management Endpoints (Tenant Admin Only)

### Create Tenant User

**Endpoint**: `POST /tenants/users`

**Authentication**: Tenant admin required

**Request Body**:
```json
{
  "email": "newuser@acme.com",
  "name": "New User",
  "roles": ["developer"]
}
```

**Response**: `201 Created`
```json
{
  "id": "user-uuid",
  "email": "newuser@acme.com",
  "name": "New User",
  "roles": ["developer"],
  "status": "active",
  "created_at": "2026-03-11T10:00:00Z"
}
```

**Error Codes**:
- `USER_EMAIL_EXISTS`: User email already exists in tenant
- `INVALID_ROLE`: Invalid role specified

---

### List Tenant Users

**Endpoint**: `GET /tenants/users`

**Authentication**: Tenant admin required

**Query Parameters**:
- `status` (optional): Filter by status (`active`, `disabled`)
- `role` (optional): Filter by role
- `limit` (optional): Pagination limit (default 50, max 100)
- `offset` (optional): Pagination offset (default 0)

**Response**: `200 OK`
```json
{
  "users": [
    {
      "id": "user-uuid",
      "email": "user@acme.com",
      "name": "User",
      "roles": ["admin", "developer"],
      "status": "active",
      "created_at": "2026-03-01T10:00:00Z"
    }
  ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

---

### Update Tenant User Roles

**Endpoint**: `PATCH /tenants/users/{user_id}`

**Authentication**: Tenant admin required

**Request Body**:
```json
{
  "roles": ["admin", "developer", "auditor"]
}
```

**Response**: `200 OK`
```json
{
  "id": "user-uuid",
  "email": "user@acme.com",
  "roles": ["admin", "developer", "auditor"],
  "updated_at": "2026-03-11T10:30:00Z"
}
```

**Error Codes**:
- `USER_NOT_FOUND`: User not found in tenant
- `INVALID_ROLE`: Invalid role specified

---

## Tenant Configuration Endpoints (Tenant Admin Only)

### Get Tenant Configuration

**Endpoint**: `GET /tenants/config`

**Authentication**: Tenant admin required

**Response**: `200 OK`
```json
{
  "sso_config": {
    "type": "oidc",
    "issuer": "https://acme.okta.com/oauth2/default"
  },
  "rbac_config": {
    "roles": {
      "admin": {
        "permissions": ["read", "write", "delete", "admin"]
      },
      "developer": {
        "permissions": ["read", "write"]
      }
    }
  },
  "cert_config": {
    "default_ttl": 3600,
    "key_algorithm": "rsa_2048"
  }
}
```

---

### Update Tenant Configuration

**Endpoint**: `PATCH /tenants/config`

**Authentication**: Tenant admin required

**Request Body**:
```json
{
  "sso_config": {
    "type": "saml",
    "idp_url": "https://acme.okta.com/sso/saml",
    "idp_cert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  },
  "rbac_config": {
    "roles": {
      "auditor": {
        "permissions": ["read"]
      }
    }
  },
  "cert_config": {
    "default_ttl": 7200,
    "key_algorithm": "rsa_4096"
  }
}
```

**Response**: `200 OK`
```json
{
  "updated_at": "2026-03-11T10:30:00Z"
}
```

**Error Codes**:
- `INVALID_SSO_CONFIG`: Invalid SSO configuration
- `INVALID_RBAC_CONFIG`: Invalid RBAC configuration

---

## Tenant Usage Metrics Endpoints (Platform Admin Only)

### Get Tenant Usage Metrics

**Endpoint**: `GET /admin/tenants/{tenant_id}/usage`

**Authentication**: Platform admin required

**Query Parameters**:
- `period` (required): Aggregation period (`hourly`, `daily`, `monthly`)
- `start_date` (required): Start date (ISO 8601)
- `end_date` (required): End date (ISO 8601)

**Response**: `200 OK`
```json
{
  "tenant_id": "tenant-uuid",
  "period": "daily",
  "metrics": [
    {
      "period_start": "2026-03-01T00:00:00Z",
      "period_end": "2026-03-01T23:59:59Z",
      "resource_count": 42,
      "active_tunnels_avg": 15,
      "session_hours": 120,
      "storage_bytes": 450000000000,
      "api_requests": 10000
    }
  ]
}
```

**Error Codes**:
- `TENANT_NOT_FOUND`: Tenant not found
- `INVALID_DATE_RANGE`: Invalid date range

---

### Export Tenant Usage Report

**Endpoint**: `GET /admin/tenants/{tenant_id}/usage/export`

**Authentication**: Platform admin required

**Query Parameters**:
- `period` (required): Aggregation period (`hourly`, `daily`, `monthly`)
- `start_date` (required): Start date (ISO 8601)
- `end_date` (required): End date (ISO 8601)
- `format` (optional): Export format (`csv`, `json`, default `csv`)

**Response**: `200 OK` (with CSV/JSON attachment)

**Error Codes**:
- `TENANT_NOT_FOUND`: Tenant not found
- `INVALID_DATE_RANGE`: Invalid date range

---

## Tenant-Scoped Resource Endpoints

All existing resource endpoints (resources, sessions, recordings, etc.) remain unchanged but are automatically scoped to the authenticated user's tenant.

**Example**: Get Resources

**Endpoint**: `GET /resources`

**Authentication**: Tenant user required

**Behavior**: Returns only resources belonging to the authenticated user's tenant.

**Response**: `200 OK`
```json
{
  "resources": [
    {
      "id": "resource-uuid",
      "tenant_id": "tenant-uuid",
      "type": "ssh",
      "name": "prod-server",
      "host": "prod.example.com",
      "port": 22,
      "status": "active"
    }
  ]
}
```

**Cross-Tenant Access Attempt**: `404 Not Found`

**Response**:
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Resource not found for tenant 'acme-corp'",
    "details": {
      "resource_id": "other-tenant-resource-uuid",
      "tenant_id": "acme-corp"
    },
    "request_id": "correlation-id"
  }
}
```

---

## Tenant Branding Endpoints

### Get Tenant Branding

**Endpoint**: `GET /branding`

**Authentication**: Optional (public endpoint for web UI)

**Response**: `200 OK`
```json
{
  "identifier": "acme-corp",
  "name": "Acme Corporation",
  "logo_url": "https://example.com/logo.png",
  "branding": {
    "primary_color": "#3b82f6",
    "secondary_color": "#1e40af",
    "font_family": "Inter"
  }
}
```

---

## Webhook Contracts

### Tenant Suspension Webhook

Webhook sent to tenant-configured webhook URL when tenant is suspended.

**Method**: `POST`

**Request Body**:
```json
{
  "event": "tenant.suspended",
  "tenant_id": "tenant-uuid",
  "tenant_identifier": "acme-corp",
  "suspended_at": "2026-03-11T10:30:00Z",
  "reason": "Terms of service violation",
  "timestamp": "2026-03-11T10:30:00Z"
}
```

---

## Pagination

All list endpoints support pagination via `limit` and `offset` query parameters.

**Default**: `limit=50`, `offset=0`

**Maximum**: `limit=100`

**Response includes**:
- `total`: Total number of items
- `limit`: Current limit
- `offset`: Current offset

---

## Rate Limiting

Per-tenant rate limits apply to all API endpoints:

- Default: 1000 requests per minute per tenant
- Configurable via tenant `usage_limits`
- Exceeding limit returns `429 Too Many Requests`

**Rate Limit Response**:
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded for tenant 'acme-corp'",
    "details": {
      "limit": 1000,
      "window": "1m"
    },
    "request_id": "correlation-id"
  }
}
```
