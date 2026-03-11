# Data Model: Multi-Tenant SaaS Support

**Feature**: Multi-Tenant SaaS Support
**Date**: 2026-03-11
**Purpose**: Define data entities, relationships, and validation rules for multi-tenant architecture

## Overview

This document defines the data model for multi-tenant SaaS capabilities, including tenant entities, tenant-scoped resources, and usage tracking. All existing BAMF entities will be extended with `tenant_id` for isolation.

## Core Tenant Entities

### Tenant

Represents an organization/customer with unique identifier, branding, SSO settings, RBAC policies, and usage limits.

**Fields**:
- `id` (UUID, primary key, immutable): Unique tenant identifier
- `identifier` (string, unique, not null): Human-readable tenant identifier (e.g., "acme-corp")
- `name` (string, not null): Display name (e.g., "Acme Corporation")
- `status` (enum, not null): Lifecycle state (`active`, `suspended`, `deleting`, `deleted`)
- `logo_url` (string, nullable): URL to tenant logo image
- `branding` (JSONB, nullable): Branding configuration (colors, fonts, custom CSS)
- `domain` (string, nullable, unique): Custom domain for web UI (e.g., "bamf.acme.com")
- `sso_config` (JSONB, nullable): SSO provider configuration (OIDC/SAML settings)
- `rbac_config` (JSONB, nullable): RBAC policy configuration
- `cert_config` (JSONB, nullable): Certificate policy configuration (TTL, key types)
- `usage_limits` (JSONB, nullable): Resource usage limits (tunnels, storage, users)
- `created_at` (timestamp, not null): Creation timestamp
- `updated_at` (timestamp, not null): Last update timestamp
- `deleted_at` (timestamp, nullable): Soft delete timestamp

**Relationships**:
- One-to-many: `TenantUser` (users belonging to tenant)
- One-to-many: `TenantResource` (resources registered to tenant)
- One-to-many: `TenantAgent` (agents belonging to tenant)
- One-to-many: `TenantCA` (certificate authorities for tenant)
- One-to-many: `TenantSession` (active sessions for tenant)
- One-to-many: `UsageMetrics` (usage metrics for tenant)

**Validation Rules**:
- `identifier` must be globally unique, alphanumeric with hyphens, 3-50 characters
- `name` must be 1-255 characters
- `status` must transition logically: `active` ↔ `suspended` ↔ `deleting` → `deleted`
- `domain` must be valid FQDN if provided, globally unique
- `sso_config` must include required fields for configured SSO provider type
- `usage_limits` must validate limit values (positive integers)

**Indexes**:
- Unique index on `identifier`
- Unique index on `domain` (for custom domains)
- Index on `status` (for filtering active tenants)
- Index on `created_at` (for sorting)

**Constraints**:
- `identifier` unique across all tenants
- `domain` unique across all tenants (if not null)

---

### TenantUser

User account belonging to a specific tenant with roles scoped to that tenant.

**Fields**:
- `id` (UUID, primary key): User identifier
- `tenant_id` (UUID, foreign key to `Tenant.id`, not null): Tenant this user belongs to
- `email` (string, not null, unique per tenant): User email address
- `name` (string, nullable): User display name
- `roles` (JSONB, not null): Array of role names (e.g., `["admin", "developer"]`)
- `status` (enum, not null): User status (`active`, `disabled`)
- `sso_id` (string, nullable): SSO provider user identifier
- `created_at` (timestamp, not null): Creation timestamp
- `updated_at` (timestamp, not null): Last update timestamp

**Relationships**:
- Many-to-one: `Tenant` (belongs to tenant)
- One-to-many: `TenantSession` (user's sessions)

**Validation Rules**:
- `email` must be valid email format
- `email` + `tenant_id` must be unique (same email can exist in different tenants)
- `roles` must reference valid roles defined in tenant's `rbac_config`
- `status` must be valid enum value

**Indexes**:
- Unique index on (`tenant_id`, `email`)
- Index on `tenant_id` (for tenant queries)
- Index on `sso_id` (for SSO lookups)

**Constraints**:
- `tenant_id` must reference existing `Tenant.id` where `status = 'active'`
- Unique constraint on (`tenant_id`, `email`)

---

### PlatformAdmin

User with system-wide administrative rights across all tenants.

**Fields**:
- `id` (UUID, primary key): Admin identifier
- `email` (string, not null, unique): Admin email address
- `name` (string, nullable): Admin display name
- `status` (enum, not null): Admin status (`active`, `disabled`)
- `created_at` (timestamp, not null): Creation timestamp
- `updated_at` (timestamp, not null): Last update timestamp

**Relationships**:
- No tenant relationships (system-wide access)

**Validation Rules**:
- `email` must be valid email format
- `email` must be unique across all platform admins

**Indexes**:
- Unique index on `email`

**Constraints**:
- `email` unique across all records

---

## Tenant-Scoped Entities

### TenantResource

Infrastructure resource (server, database, cluster, web app) registered to a specific tenant.

**Fields**:
- `id` (UUID, primary key): Resource identifier
- `tenant_id` (UUID, foreign key to `Tenant.id`, not null): Tenant owning this resource
- `type` (enum, not null): Resource type (`ssh`, `database`, `kubernetes`, `web_app`)
- `name` (string, not null): Resource display name
- `host` (string, not null): Hostname or IP address
- `port` (integer, nullable): Port number (if applicable)
- `config` (JSONB, nullable): Resource-specific configuration
- `status` (enum, not null): Resource status (`active`, `inactive`, `error`)
- `created_at` (timestamp, not null): Creation timestamp
- `updated_at` (timestamp, not null): Last update timestamp

**Relationships**:
- Many-to-one: `Tenant` (belongs to tenant)
- One-to-many: `TenantSession` (sessions accessing this resource)

**Validation Rules**:
- `name` must be unique within tenant (not enforced at DB level)
- `host` must be valid hostname or IP address
- `port` must be valid port number (1-65535) if provided
- `type` must be valid enum value

**Indexes**:
- Unique index on (`tenant_id`, `name`, `type`)
- Index on `tenant_id` (for tenant queries)
- Index on `status` (for filtering active resources)

**Constraints**:
- `tenant_id` must reference existing `Tenant.id`
- Unique constraint on (`tenant_id`, `name`, `type`)

---

### TenantAgent

Agent process registered to and belonging to a single tenant.

**Fields**:
- `id` (UUID, primary key): Agent identifier
- `tenant_id` (UUID, foreign key to `Tenant.id`, not null): Tenant this agent belongs to
- `name` (string, not null): Agent display name
- `hostname` (string, not null): Hostname where agent is running
- `version` (string, not null): Agent version
- `status` (enum, not null): Agent status (`connected`, `disconnected`, `error`)
- `last_seen_at` (timestamp, nullable): Last connection timestamp
- `created_at` (timestamp, not null): Creation timestamp

**Relationships**:
- Many-to-one: `Tenant` (belongs to tenant)
- One-to-many: `TenantResource` (resources registered by this agent)

**Validation Rules**:
- `name` must be unique within tenant
- `hostname` must be valid hostname
- `version` must follow semantic versioning

**Indexes**:
- Unique index on (`tenant_id`, `hostname`)
- Index on `tenant_id` (for tenant queries)
- Index on `status` (for filtering connected agents)

**Constraints**:
- `tenant_id` must reference existing `Tenant.id`
- Unique constraint on (`tenant_id`, `hostname`)

---

### TenantCA

Certificate authority instance scoped to a specific tenant for issuing certificates.

**Fields**:
- `id` (UUID, primary key): CA identifier
- `tenant_id` (UUID, foreign key to `Tenant.id`, not null): Tenant owning this CA
- `type` (enum, not null): CA type (`ssh`, `x509`)
- `key_algorithm` (enum, not null): Key algorithm (`rsa_2048`, `rsa_4096`, `ecdsa_p256`)
- `certificate` (text, not null): CA certificate in PEM format
- `private_key` (text, not null): CA private key in PEM format (encrypted)
- `certificate_ttl` (integer, not null): Default certificate TTL in seconds
- `created_at` (timestamp, not null): Creation timestamp
- `rotated_at` (timestamp, nullable): Last rotation timestamp

**Relationships**:
- Many-to-one: `Tenant` (belongs to tenant)

**Validation Rules**:
- `type` must be valid enum value
- `key_algorithm` must be valid enum value
- `certificate` must be valid PEM format
- `private_key` must be valid encrypted PEM format
- `certificate_ttl` must be positive integer (60-86400 seconds)

**Indexes**:
- Unique index on (`tenant_id`, `type`)
- Index on `tenant_id` (for tenant queries)

**Constraints**:
- `tenant_id` must reference existing `Tenant.id`
- Unique constraint on (`tenant_id`, `type`) (one SSH CA and one x509 CA per tenant)

---

### TenantSession

Active user session scoped to a specific tenant with associated tunnel connections.

**Fields**:
- `id` (UUID, primary key): Session identifier
- `tenant_id` (UUID, foreign key to `Tenant.id`, not null): Tenant this session belongs to
- `user_id` (UUID, foreign key to `TenantUser.id`, not null): User this session belongs to
- `resource_id` (UUID, foreign key to `TenantResource.id`, not null): Resource being accessed
- `type` (enum, not null): Session type (`ssh`, `database`, `kubernetes`, `web_app`)
- `status` (enum, not null): Session status (`active`, `terminated`, `error`)
- `started_at` (timestamp, not null): Session start timestamp
- `ended_at` (timestamp, nullable): Session end timestamp
- `metadata` (JSONB, nullable): Session metadata (client IP, user agent, etc.)

**Relationships**:
- Many-to-one: `Tenant` (belongs to tenant)
- Many-to-one: `TenantUser` (belongs to user)
- Many-to-one: `TenantResource` (accesses resource)

**Validation Rules**:
- `type` must be valid enum value
- `status` must transition logically: `active` → `terminated` / `error`
- `ended_at` must be >= `started_at`

**Indexes**:
- Index on `tenant_id` (for tenant queries)
- Index on `user_id` (for user session history)
- Index on `resource_id` (for resource access history)
- Index on `status` (for filtering active sessions)
- Index on `started_at` (for session history)

**Constraints**:
- `tenant_id` must reference existing `Tenant.id`
- `user_id` must reference existing `TenantUser.id`
- `resource_id` must reference existing `TenantResource.id`

---

## Usage Tracking Entities

### UsageMetrics

Per-tenant usage tracking for billing and capacity management (tunnel hours, storage, resource counts).

**Fields**:
- `id` (UUID, primary key): Metric identifier
- `tenant_id` (UUID, foreign key to `Tenant.id`, not null): Tenant these metrics belong to
- `period` (enum, not null): Aggregation period (`hourly`, `daily`, `monthly`)
- `period_start` (timestamp, not null): Start of aggregation period
- `period_end` (timestamp, not null): End of aggregation period
- `resource_count` (integer, not null, default 0): Number of resources
- `active_tunnels_avg` (integer, not null, default 0): Average concurrent tunnels
- `session_hours` (integer, not null, default 0): Total session hours in period
- `storage_bytes` (integer, not null, default 0): Storage consumption in bytes
- `api_requests` (integer, not null, default 0): API request count
- `created_at` (timestamp, not null): Creation timestamp

**Relationships**:
- Many-to-one: `Tenant` (belongs to tenant)

**Validation Rules**:
- `period` must be valid enum value
- `period_end` must be > `period_start`
- All metric values must be non-negative

**Indexes**:
- Unique index on (`tenant_id`, `period`, `period_start`)
- Index on `tenant_id` (for tenant queries)
- Index on `period_start` (for time-range queries)

**Constraints**:
- `tenant_id` must reference existing `Tenant.id`
- Unique constraint on (`tenant_id`, `period`, `period_start`)

---

## Existing BAMF Entities (Extended)

The following existing BAMF entities will be extended with `tenant_id` field:

### Users (existing)

**Changes**:
- Add `tenant_id` column (UUID, nullable for migration, then NOT NULL)
- Add `is_platform_admin` boolean column (default false)
- Migrate existing users to "legacy" tenant
- Platform admins: `tenant_id` is NULL, `is_platform_admin` is true

### Resources (existing)

**Changes**:
- Add `tenant_id` column (UUID, nullable for migration, then NOT NULL)
- Migrate existing resources to "legacy" tenant

### Roles (existing)

**Changes**:
- Add `tenant_id` column (UUID, nullable for migration, then NOT NULL)
- Migrate existing roles to "legacy" tenant

### Sessions (existing)

**Changes**:
- Add `tenant_id` column (UUID, nullable for migration, then NOT NULL)
- Migrate existing sessions to "legacy" tenant

### AuditLogs (existing)

**Changes**:
- Add `tenant_id` column (UUID, nullable for migration, then NOT NULL)
- Migrate existing audit logs to "legacy" tenant

### Recordings (existing)

**Changes**:
- Add `tenant_id` column (UUID, nullable for migration, then NOT NULL)
- Migrate existing recordings to "legacy" tenant

### Certificates (existing)

**Changes**:
- Add `tenant_id` column (UUID, nullable for migration, then NOT NULL)
- Add `ca_id` column (UUID, foreign key to `TenantCA.id`, nullable)
- Migrate existing certificates to "legacy" tenant's CA

---

## Row-Level Security Policies

PostgreSQL row-level security policies will enforce tenant isolation at database level:

### Policy: tenant_isolation

**Applies to**: All tenant-scoped tables (users, resources, roles, sessions, audit_logs, recordings, certificates)

**Rule**: Allow SELECT/INSERT/UPDATE/DELETE only on rows matching user's tenant_id

**SQL Example** (for `users` table):
```sql
CREATE POLICY tenant_isolation ON users
FOR ALL
USING (
  tenant_id = current_setting('app.current_tenant_id')::UUID
)
WITH CHECK (
  tenant_id = current_setting('app.current_tenant_id')::UUID
);
```

**Platform Admin Exception**:
```sql
CREATE POLICY platform_admin_access ON users
FOR ALL
TO platform_admin_role
USING (true);
```

---

## Database Migration Plan

### Migration Steps

1. Create new tenant tables (`Tenant`, `TenantUser`, `PlatformAdmin`, `TenantResource`, `TenantAgent`, `TenantCA`, `TenantSession`, `UsageMetrics`)
2. Create "legacy" tenant record
3. Add `tenant_id` column to existing tables (nullable)
4. Backfill `tenant_id` with "legacy" tenant ID for all existing records
5. Add `is_platform_admin` column to `users` table
6. Create indexes on `tenant_id` columns
7. Create row-level security policies
8. Enable row-level security on all tables
9. Update application code to set `app.current_tenant_id` session variable
10. Test multi-tenant isolation
11. (After verification) Set `tenant_id` NOT NULL

### Rollback Plan

1. Disable row-level security on all tables
2. Drop row-level security policies
3. Drop `tenant_id` columns from existing tables
4. Drop new tenant tables
5. Restore pre-migration database backup

---

## Data Relationships Diagram

```
Tenant (1) ──────── (N) TenantUser
  │
  ├── (1) ──────── (N) TenantResource
  │                        │
  │                        └── (N) TenantSession ──── (N) TenantUser
  │
  ├── (1) ──────── (N) TenantAgent
  │
  ├── (1) ──────── (N) TenantCA
  │
  └── (1) ──────── (N) UsageMetrics

PlatformAdmin (no tenant relationship)
```

---

## Validation Summary

All entities include:
- Primary key constraints
- Foreign key constraints (where applicable)
- Unique constraints (where applicable)
- Check constraints (for enums and ranges)
- Indexes for common query patterns
- Timestamps for auditing (created_at, updated_at, deleted_at)

Tenant isolation enforced at:
- Application layer (tenant context in request)
- API layer (middleware validation)
- Database layer (row-level security policies)
