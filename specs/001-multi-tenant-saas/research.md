# Research: Multi-Tenant SaaS Support

**Feature**: Multi-Tenant SaaS Support
**Date**: 2026-03-11
**Purpose**: Resolve technical decisions and unknowns before design phase

## Overview

This document captures research findings for adding multi-tenant SaaS capabilities to BAMF, including database schema changes, authentication flow modifications, and tenant isolation strategies.

## Research Questions and Decisions

### 1. Tenant Context Extraction from SSO

**Question**: How should tenant context be extracted from SSO identity providers?

**Decision**: Extract tenant identifier from SSO claims, with fallback to email domain mapping

**Rationale**:
- SSO providers (Okta, Azure AD, Google) can include custom claims with tenant/organization IDs
- Email domain mapping provides fallback for IdPs that don't support custom claims
- Explicit tenant selection available as manual fallback for edge cases
- Matches industry best practices (Auth0, Salesforce, Atlassian use similar approaches)

**Alternatives Considered**:
- **Subdomain-based routing**: Rejected - requires complex DNS configuration per tenant
- **Header-based routing**: Rejected - less secure, requires complex ingress configuration
- **User selection after login**: Rejected - poor UX, adds friction to authentication flow

**Implementation Approach**:
1. Primary: Extract tenant ID from custom SSO claim (e.g., `https://bamf.io/tenant_id`)
2. Secondary: Map email domain to tenant (`user@acme.com` → tenant with domain `acme.com`)
3. Fallback: Present tenant selection UI for users associated with multiple tenants
4. Store tenant ID in session JWT for subsequent requests

**SSO Provider Support**:
- Okta: Custom claims via token customization
- Azure AD: Optional claims configuration
- Google Workspace: Custom attributes in Directory API
- Keycloak: Protocol mappers for custom claims
- SAML 2.0: Custom attributes in SAML assertion

---

### 2. Database Schema Multi-Tenancy Pattern

**Question**: What multi-tenancy pattern should be used for the database schema?

**Decision**: Row-level security with tenant_id columns (discriminator column pattern)

**Rationale**:
- BAMF already uses PostgreSQL which supports robust row-level security policies
- Single database instance reduces operational complexity vs. separate databases per tenant
- Easier to implement cross-tenant reporting for platform administrators
- Maintains existing database schema with minimal breaking changes
- Cost-effective for SaaS hosting (single instance vs. many instances)

**Alternatives Considered**:
- **Separate schemas per tenant**: Rejected - Complex migration path, harder cross-tenant queries
- **Separate databases per tenant**: Rejected - Higher cost, complex connection management, scalability issues
- **Application-level filtering only**: Rejected - Less secure, requires careful query construction across all code

**Implementation Approach**:
1. Add `tenant_id` column to all tenant-scoped tables (users, resources, roles, sessions, audit logs)
2. Create PostgreSQL row-level security policies to enforce tenant isolation at database level
3. Use database migrations (Alembic) to add columns and create indexes on `tenant_id`
4. Existing data migrated to "legacy" tenant with tenant_id set
5. Application queries automatically include tenant_id filter via ORM or middleware

**Tables Requiring tenant_id**:
- `users` (tenant users)
- `resources` (SSH servers, databases, K8s clusters, web apps)
- `roles` (RBAC roles)
- `sessions` (user sessions, tunnel connections)
- `audit_logs` (audit entries)
- `recordings` (session recordings)
- `certificates` (issued certificates)

**Tables NOT requiring tenant_id**:
- `tenants` (tenant definitions - global table)
- `platform_admins` (platform administrators - global table)
- `usage_metrics` (aggregated per-tenant metrics - has tenant_id but global access for billing)

---

### 3. Tenant-Specific Certificate Authorities

**Question**: How should certificate authorities be scoped per tenant?

**Decision**: Per-tenant CA instances with unique certificates per tenant

**Rationale**:
- BAMF already uses certificate-based trust model (x509 for API, SSH certificates for access)
- Per-tenant CA prevents cross-tenant certificate misuse
- Allows tenants to configure different certificate policies (TTL, key types, extensions)
- Aligns with principle of least privilege (each tenant can only use their own CA)
- Meets compliance requirements for audit trails and key rotation

**Alternatives Considered**:
- **Shared CA with tenant-specific certificates**: Rejected - Risk of certificate misuse, harder audit trails
- **CA per bridge instance**: Rejected - Doesn't align with tenant model, bridge failure disrupts all tenant certs

**Implementation Approach**:
1. Create `TenantCA` model with tenant_id, CA key/cert storage, policy configuration
2. Generate unique CA key pair for each tenant on tenant creation
3. Store CA certificates securely (database or Kubernetes secrets)
4. Certificate issuance validates tenant context and uses tenant-specific CA
5. Certificate revocation is scoped per tenant
6. Platform admin can rotate tenant CA keys if needed

**Certificate Policy Per-Tenant**:
- Certificate TTL (default: 1 hour, configurable per tenant)
- Key types (RSA 2048, RSA 4096, ECDSA P-256, etc.)
- Certificate extensions (custom extensions for tenant-specific claims)
- Revocation configuration (CRL, OCSP)

---

### 4. Resource Isolation Strategy

**Question**: How should resource isolation be enforced across all layers?

**Decision**: Defense in depth with isolation at API, application, and database layers

**Rationale**:
- No single layer of isolation is sufficient for security-critical systems
- Database-level isolation prevents application bugs from leaking data
- API-level validation provides user-friendly error messages
- Application-level filtering ensures efficient queries
- Matches security best practices for multi-tenant SaaS (zero-trust model)

**Layers of Isolation**:
1. **Authentication Layer**: Extract tenant context from JWT token, reject requests without valid tenant context
2. **API Middleware**: Validate tenant ID exists and is active before processing request
3. **Application Logic**: All queries include `tenant_id` filter via ORM or query builders
4. **Database Layer**: PostgreSQL row-level security policies prevent cross-tenant queries
5. **Service Layer**: Bridge and agent connections validate tenant membership before establishing tunnels

**Validation Points**:
- API endpoints: Validate tenant context on every request
- CLI commands: Derive tenant from authenticated session, reject explicit tenant switching
- Web UI: Hide/ disable resources from other tenants in all UI components
- Bridge connections: Validate agent's tenant_id matches resource's tenant_id
- Agent registration: Require tenant association token, reject mismatched registrations

**Failure Modes**:
- Missing tenant context: Return 401 Unauthorized with clear error message
- Invalid tenant ID: Return 404 Not Found (resource not found for tenant)
- Cross-tenant access attempt: Return 403 Forbidden with audit log entry

---

### 5. Usage Metrics and Billing

**Question**: How should usage metrics be tracked per tenant?

**Decision**: Event-driven aggregation with real-time counters and periodic rollups

**Rationale**:
- BAMF already emits structured audit logs for all operations
- Event-driven approach provides real-time metrics without expensive queries
- Periodic rollups enable efficient billing reports
- Supports both real-time monitoring (dashboard) and historical reporting (billing)

**Metrics Tracked Per-Tenant**:
- Resource counts (servers, databases, clusters, web apps)
- Active tunnels (count, duration, throughput)
- Session metrics (count, duration, users)
- Storage consumption (recordings, logs)
- API usage (requests, latency, errors)
- Certificate issuance (count, types)

**Implementation Approach**:
1. Emit structured audit events with tenant_id for all operations
2. Increment Redis counters in real-time for active metrics (tunnels, sessions)
3. Periodic background job aggregates counters into `usage_metrics` table
4. Rollups hourly, daily, monthly for different reporting needs
5. Platform admin dashboard queries aggregated metrics for all tenants
6. Tenant dashboard queries only their own metrics

**Storage**:
- Redis: Real-time counters (active tunnels, sessions)
- PostgreSQL: Aggregated usage metrics (hourly/daily/monthly rollups)
- Retention: Keep detailed data for 90 days, aggregated data for 1 year

---

### 6. Tenant Lifecycle Management

**Question**: How should tenant lifecycle (creation, suspension, deletion) be implemented?

**Decision**: Soft delete with configurable retention period and graceful session termination

**Rationale**:
- Soft delete allows data recovery and compliance with retention policies
- Graceful session termination prevents data loss for active users
- Configurable retention periods meet different compliance requirements (GDPR, HIPAA)
- Background deletion prevents API timeouts for large tenants

**Lifecycle States**:
- `active`: Tenant can onboard users, create resources, establish tunnels
- `suspended`: Existing sessions continue, new sessions blocked, resources read-only
- `deleting`: Data being archived, no new operations
- `deleted`: Tenant removed, data retained per retention policy

**Suspension Flow**:
1. Platform admin marks tenant as `suspended`
2. API rejects new tunnel establishment requests with clear error
3. Existing tunnels allowed to complete gracefully
4. Web UI shows suspension notice to all tenant users
5. Audit logs record suspension event

**Deletion Flow**:
1. Platform admin initiates deletion, selects retention period (0/30/90 days)
2. Tenant marked as `deleting`, no new operations allowed
3. Background job archives data to cold storage
4. Active sessions terminated gracefully
5. After retention period, data permanently deleted
6. Audit trail retained permanently (for compliance)

**Data Retention**:
- Audit logs: Permanent (required for compliance)
- Session recordings: Per-tenant retention policy (default: 90 days)
- Resource metadata: Per-tenant retention policy (default: 30 days)
- User accounts: Deleted immediately (except platform admins)

---

### 7. Migration Strategy for Existing Data

**Question**: How should existing single-tenant BAMF data be migrated to multi-tenant?

**Decision**: Create "legacy" tenant and migrate all existing data to it automatically

**Rationale**:
- Preserves all existing data and configuration without manual intervention
- Allows existing users to continue using BAMF without changes
- Platform admin can reorganize tenants later if needed
- Aligns with backward compatibility requirements

**Migration Steps**:
1. Create "legacy" tenant with default branding and SSO configuration
2. Add `tenant_id` column to all tenant-scoped tables (nullable initially)
3. Set `tenant_id = legacy_tenant.id` for all existing records
4. Make `tenant_id` NOT NULL after migration completes
5. Create indexes on `tenant_id` columns
6. Add row-level security policies
7. Deploy code changes to use tenant context
8. Monitor for errors, rollback if needed

**Migration Safety**:
- Run migration in transaction to ensure atomicity
- Backup database before migration
- Test migration on staging environment first
- Provide rollback plan in case of failure
- Monitor application logs for errors after migration

---

## Technology Stack Validation

### PostgreSQL Features Used

- **Row-Level Security (RLS)**: Enforce tenant isolation at database level
- **Indexes on tenant_id**: Ensure efficient queries
- **Foreign Keys**: Maintain referential integrity with tenant_id
- **JSONB Columns**: Store tenant-specific configuration (branding, SSO settings)
- **Triggers**: Optional - automatic timestamp updates for usage metrics

### Redis Usage

- **Counters**: Real-time usage metrics (active tunnels, sessions)
- **Caching**: Tenant configuration caching (5-minute TTL)
- **Session Storage**: JWT tokens with tenant context
- **Rate Limiting**: Per-tenant API rate limits

### FastAPI Middleware

- **Tenant Context Middleware**: Extract tenant_id from JWT, inject into request state
- **Access Control Middleware**: Validate tenant permissions before processing
- **Audit Middleware**: Log all operations with tenant context

### Go Tenant Types

- **TenantID**: Strongly typed identifier (not string)
- **TenantContext**: Struct passed to all operations requiring tenant scoping
- **TenantScope**: Interface for tenant-scoped operations

### Next.js Tenant Context

- **TenantProvider**: React context provider for tenant branding
- **useTenant**: Hook to access tenant context in components
- **TenantBranding**: Component for applying tenant-specific styling

---

## Performance Considerations

### Database Performance

- All tenant-scoped queries must use indexed `tenant_id` columns
- Row-level security policies must not add significant overhead
- Connection pooling must handle multi-tenant queries efficiently
- Query performance targets maintained: <500ms p95 for audit log queries

### Caching Strategy

- Tenant configuration cached in Redis (5-minute TTL)
- Role definitions cached per tenant (5-minute TTL)
- User sessions cached in Redis with tenant context
- Audit log pagination state cached per tenant

### Scale Targets

- 100+ concurrent tenants
- 1000+ resources per tenant
- 1000+ concurrent tunnels per tenant
- <200ms API latency p95 (including tenant scoping overhead)

---

## Security Considerations

### Cross-Tenant Data Leakage Prevention

- Database row-level security as final defense
- Application-level filtering as primary defense
- Audit all cross-tenant access attempts
- Regular security scans to validate isolation

### Tenant Context Validation

- Validate tenant_id exists and is active on every request
- Reject requests with invalid tenant context immediately
- Log all validation failures for security monitoring
- Platform admin context bypasses tenant validation

### Certificate Security

- Per-tenant CA keys stored securely (database or Kubernetes secrets)
- Certificate issuance validates tenant context
- Certificate revocation scoped per tenant
- Audit all certificate operations

---

## Open Questions and Risks

### Open Questions

None - all technical decisions resolved.

### Risks

1. **Migration Complexity**: Database migration for existing data could fail
   - Mitigation: Test on staging, backup database, have rollback plan

2. **Performance Impact**: Adding tenant filtering to all queries could degrade performance
   - Mitigation: Use indexed tenant_id columns, cache aggressively, monitor performance

3. **SSO Provider Variability**: Different IdPs may not support custom claims
   - Mitigation: Fallback to email domain mapping, manual selection option

4. **Large Tenant Deletion**: Deleting tenant with 10,000 resources could timeout API
   - Mitigation: Background deletion with progress tracking

---

## Conclusion

All technical research complete. Multi-tenant SaaS support is feasible using PostgreSQL row-level security, per-tenant CAs, and defense-in-depth isolation strategy. Existing BAMF architecture can be extended with minimal breaking changes by migrating existing data to a "legacy" tenant.

Next steps: Proceed to Phase 1 design (data model, contracts, quickstart).
