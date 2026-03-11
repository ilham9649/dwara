# Implementation Plan: Multi-Tenant SaaS Support

**Branch**: `001-multi-tenant-saas` | **Date**: 2026-03-11 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-multi-tenant-saas/spec.md`

## Summary

Implement multi-tenant SaaS capabilities for BAMF to enable hosting multiple organizations on a single platform while maintaining complete resource isolation, tenant-specific configurations (SSO, RBAC, certificates), and per-tenant usage tracking. The feature requires database schema changes to add tenant context to all tenant-scoped entities, authentication flow updates to derive tenant context from SSO claims, and UI additions for tenant management.

## Technical Context

**Language/Version**:
- Go 1.21+ (CLI, bridge, agent)
- Python 3.11+ (API server, FastAPI 0.104+)
- TypeScript 5.0+ (Next.js 14+, React 18+)

**Primary Dependencies**:
- Python: FastAPI, SQLAlchemy, structlog, Pydantic, Alembic
- Go: slog, context, standard library (no CGo)
- TypeScript: Next.js, shadcn/ui, React Query

**Storage**: PostgreSQL 14+ with connection pooling, Redis for caching sessions and audit log pagination state

**Testing**:
- Python: pytest, pytest-asyncio, pytest-cov
- Go: built-in testing with table-driven tests
- TypeScript: Vitest, React Testing Library
- Integration: testcontainers for PostgreSQL, Redis

**Target Platform**: Kubernetes (local: Rancher Desktop, production: cloud K8s clusters)

**Project Type**: Distributed web service with CLI client (backend: Python API, frontend: Next.js, data plane: Go services)

**Performance Goals**:
- Tenant onboarding: <5 minutes including SSO configuration
- API latency: <200ms p95 (maintain existing performance)
- Support 100+ concurrent tenants without degradation
- Tenant configuration changes: <30 seconds to take effect
- Tenant deletion: <5 minutes for up to 10,000 resources

**Constraints**:
- Must maintain existing single-tenant functionality (migrate to "legacy" tenant)
- Zero cross-tenant data leakage under all conditions
- Existing BAMF infrastructure must continue functioning during implementation
- Must support existing authentication flows while adding tenant context
- Database migration must preserve all existing data

**Scale/Scope**:
- Support 100+ concurrent tenants
- Each tenant can have 1000+ resources
- Platform-level operations for managing tenant lifecycle
- Per-tenant SSO, RBAC, certificate policies, and branding
- Usage tracking and billing metrics per tenant

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Code Quality Excellence

- [x] Simplicity First: Multi-tenant isolation will use tenant IDs on database queries rather than complex query filters
- [x] No Comments Policy: Code will use descriptive names (e.g., `TenantScopedQuery`, `GetTenantContext`)
- [x] Error Handling: All multi-tenant operations will wrap errors with context (tenant ID, operation type)
- [x] Dependency Management: Only existing dependencies will be used; no new libraries without justification
- [x] Type Safety: Tenant IDs will be strongly typed (Python: `TenantId` type alias, Go: custom type, TypeScript: `TenantId` type)
- [x] Naming Conventions: All code will follow existing patterns (e.g., `tenant_id` in Python, `TenantID` in Go, `tenantId` in TypeScript)

### Testing Standards (NON-NEGOTIABLE)

- [x] Test Coverage: All multi-tenant code will have 80%+ coverage, critical paths 90%+ (tenant isolation, authentication)
- [x] Test Organization:
  - Unit tests for tenant scoping logic
  - Contract tests between Go, Python, TypeScript for tenant context propagation
  - Integration tests for end-to-end tenant flows (onboarding, isolation, deletion)
  - E2E tests for complete multi-tenant user journeys
- [x] Test-First Development: All multi-tenant features will follow TDD
- [x] Test Isolation: Tests will use tenant-specific fixtures to ensure independence
- [x] Security Testing: Security tests will validate tenant isolation (Semgrep rules for tenant ID checks, Nuclei templates for cross-tenant access attempts)
- [x] Performance Testing: Performance tests will validate multi-tenant performance targets (100+ concurrent tenants, <200ms API latency with tenant scoping)

### User Experience Consistency

- [x] CLI UX: CLI will maintain POSIX conventions; tenant context derived from SSO, no explicit `--tenant` flag needed
- [x] Web UI Consistency: Web UI will use shadcn/ui components for tenant management dashboard
- [x] Error Messages: All errors will include tenant context where applicable (e.g., "Resource not found for tenant 'acme-corp'")
- [x] Feedback Latency: Tenant operations will provide feedback within 100ms (loading indicators, progress bars)

### Security Requirements

- [x] Certificate-Based Trust: Per-tenant CAs will be used to prevent cross-tenant certificate misuse
- [x] No Secrets in Code: Tenant-specific secrets will be stored in environment variables or Kubernetes secrets
- [x] Audit Logging: All multi-tenant operations will be logged with tenant ID, user ID, action, result
- [x] Input Validation: Tenant IDs will be validated and sanitized at all API boundaries
- [x] Least Privilege: Tenant users will have access only to their tenant's data; platform admins have system-wide access
- [x] Session Recording: Recordings will be isolated per tenant with playback access restricted to tenant users

### Performance Requirements

- [x] Tunnel Performance: Multi-tenant overhead must be minimal; tunnel performance targets maintained (<500ms SSH, <300ms TCP)
- [x] API Performance: Multi-tenant queries must be efficient; <200ms p95 latency maintained
- [x] Resource Limits: API memory <512MB baseline, bridge memory <512MB per 100 tunnels (multi-tenant doesn't change per-tunnel cost)
- [x] Caching: Tenant-specific caching (role definitions, sessions, audit log state) will use Redis
- [x] Database Performance: All tenant-scoped queries will use indexed tenant_id columns to avoid full table scans

### Quality Gates

- [x] Testing: All tests must pass (unit, integration, contract, E2E)
- [x] Linting: All linters must pass (`golangci-lint`, `ruff`, `eslint`)
- [x] Type Checking: Strict type checking must pass (`mypy`, `tsc`)
- [x] Security Scanning: Must pass (Semgrep, Trivy)
- [x] Documentation: Multi-tenant docs must be updated
- [x] Code Review: At least one approval required

**Status**: All gates PASS ✓

## Project Structure

### Documentation (this feature)

```text
specs/001-multi-tenant-saas/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
│   ├── api-contracts.md # REST API contracts for tenant management
│   └── cli-contracts.md # CLI command contracts
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
# Multi-tenant additions to existing BAMF structure
services/bamf/api/
├── bamf/api/routes/tenants.py        # New: Tenant CRUD endpoints
├── bamf/api/routes/tenant_admin.py   # New: Tenant management operations
├── bamf/api/middleware/tenant.py     # New: Tenant context extraction middleware
├── bamf/auth/tenant_context.py        # New: Tenant context from SSO claims
├── bamf/db/models/tenant.py          # New: Tenant model
├── bamf/db/models/tenant_user.py      # New: Tenant user model
└── bamf/db/models/usage_metrics.py    # New: Usage tracking model

services/bamf/db/
└── alembic/versions/                 # New migrations: Add tenant_id to all tenant-scoped tables

pkg/tenant/
├── tenant.go                         # New: Go tenant context types
└── isolation.go                      # New: Tenant isolation validation

cmd/bamf/
└── tenant/                           # New: Tenant management CLI commands

web/src/
├── pages/admin/tenants/              # New: Tenant management dashboard
│   ├── index.tsx                     # Tenant list view
│   ├── [id]/                        # Tenant detail view
│   └── new.tsx                      # Tenant creation form
└── components/tenant/                # New: Tenant-specific UI components
    ├── TenantContext.tsx              # Tenant context provider
    └── TenantBranding.tsx            # Branding display component

tests/
├── unit/
│   ├── test_tenant_isolation.py       # New: Python unit tests for tenant scoping
│   ├── test_tenant_context.py         # New: Python unit tests for tenant context
│   └── tenant_test.go               # New: Go unit tests for tenant types
├── integration/
│   ├── test_multi_tenant_flow.py      # New: End-to-end multi-tenant flow tests
│   └── test_tenant_isolation.py     # New: Cross-tenant isolation tests
└── contract/
    └── test_tenant_context.go        # New: Contract tests for tenant context propagation
```

**Structure Decision**: Multi-tenant feature adds new models, routes, and UI components to existing BAMF structure. All tenant-scoped tables will have `tenant_id` added via migrations. Middleware will extract tenant context from SSO claims and inject it into request context for all subsequent operations. Go, Python, and TypeScript services will use contract tests to ensure tenant context is correctly propagated across service boundaries.

## Complexity Tracking

> **No violations requiring justification - all gates pass**
