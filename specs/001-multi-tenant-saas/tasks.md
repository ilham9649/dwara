---

description: "Task list for multi-tenant SaaS implementation"
---

# Tasks: Multi-Tenant SaaS Support

**Input**: Design documents from `/specs/001-multi-tenant-saas/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Test-First Development (TDD) is REQUIRED per Constitution. Tests are written first for all critical paths.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Python API**: `services/bamf/api/`
- **Go CLI/Data Plane**: `cmd/bamf/`, `pkg/`
- **TypeScript Web UI**: `web/src/`
- **Database Migrations**: `services/bamf/db/alembic/versions/`
- **Tests**: `tests/unit/`, `tests/integration/`, `tests/contract/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create tenant feature directories in services/bamf/api/bamf/api/routes/, bamf/auth/, bamf/db/models/, pkg/tenant/, cmd/bamf/tenant/, web/src/pages/admin/tenants/, web/src/components/tenant/, tests/unit/, tests/integration/, tests/contract/
- [ ] T002 Configure test database with PostgreSQL 14+ and Redis for testcontainers
- [ ] T003 [P] Configure linting and formatting tools for multi-tenant code (ruff, golangci-lint, eslint)
- [ ] T004 [P] Setup test fixtures for tenant entities (Tenant, TenantUser, PlatformAdmin, TenantResource, TenantAgent, TenantCA, TenantSession, UsageMetrics)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T005 Create "legacy" tenant model in services/bamf/db/models/tenant.py (for migration)
- [ ] T006 [P] Create database migration for new tenant tables (Tenant, TenantUser, PlatformAdmin, TenantResource, TenantAgent, TenantCA, TenantSession, UsageMetrics) in services/bamf/db/alembic/versions/
- [ ] T007 [P] Create database migration to add tenant_id column to existing tables (users, resources, roles, sessions, audit_logs, recordings, certificates) in services/bamf/db/alembic/versions/
- [ ] T008 [P] Create database migration to add is_platform_admin column to users table in services/bamf/db/alembic/versions/
- [ ] T009 Create database migration to backfill tenant_id with legacy tenant ID in services/bamf/db/alembic/versions/
- [ ] T010 [P] Create database migration to add indexes on tenant_id columns in services/bamf/db/alembic/versions/
- [ ] T011 Create database migration to create row-level security policies for tenant isolation in services/bamf/db/alembic/versions/
- [ ] T012 [P] Implement PostgreSQL tenant context session variable setter in services/bamf/db/session.py
- [ ] T013 [P] Implement Go TenantID type and validation in pkg/tenant/tenant.go
- [ ] T014 [P] Implement TypeScript TenantId type and utilities in web/src/lib/tenant.ts
- [ ] T015 [P] Configure Redis for tenant-specific caching (roles, sessions, audit log state) in services/bamf/db/redis.py
- [ ] T016 [P] Configure error handling with tenant context wrapping in services/bamf/api/errors.py
- [ ] T017 [P] Configure structured logging with tenant context in services/bamf/api/logging.py

**Checkpoint**: Foundation ready - database migrations, types, caching, and logging infrastructure complete

---

## Phase 3: User Story 1 - Tenant Onboarding (Priority: P1) 🎯 MVP

**Goal**: Platform administrators can create tenants, configure branding, assign tenant admin users, and tenants become immediately available for resource provisioning.

**Independent Test**: Create a tenant via platform admin CLI, verify unique identifier works, confirm branding appears in web UI for that tenant, and validate tenant admin can log in without affecting other tenants.

### Tests for User Story 1 (TDD - Write First!) ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [ ] T018 [P] [US1] Contract test for tenant creation API endpoint in tests/contract/test_tenant_api.py
- [ ] T019 [P] [US1] Contract test for tenant management API endpoints in tests/contract/test_tenant_management_api.py
- [ ] T020 [P] [US1] Integration test for tenant onboarding flow in tests/integration/test_tenant_onboarding.py
- [ ] T021 [P] [US1] Integration test for tenant branding application in tests/integration/test_tenant_branding.py

### Implementation for User Story 1

- [ ] T022 [P] [US1] Create Tenant model in services/bamf/db/models/tenant.py
- [ ] T023 [P] [US1] Create TenantUser model in services/bamf/db/models/tenant_user.py
- [ ] T024 [P] [US1] Create PlatformAdmin model in services/bamf/db/models/platform_admin.py
- [ ] T025 [US1] Implement Tenant service (CRUD operations) in services/bamf/api/services/tenant_service.py (depends on T022, T023, T024)
- [ ] T026 [US1] Implement tenant branding service in services/bamf/api/services/branding_service.py (depends on T022)
- [ ] T027 [US1] Implement tenant context extraction middleware in services/bamf/api/middleware/tenant.py (depends on T012)
- [ ] T028 [US1] Implement platform admin authentication middleware in services/bamf/api/middleware/admin.py
- [ ] T029 [US1] Implement tenant management API routes in services/bamf/api/routes/tenants.py (depends on T025, T027, T028)
- [ ] T030 [US1] Implement tenant CRUD API endpoints in services/bamf/api/routes/tenants.py (depends on T029)
- [ ] T031 [US1] Implement tenant user management API endpoints in services/bamf/api/routes/tenant_users.py (depends on T025)
- [ ] T032 [US1] Add validation and error handling for tenant operations in services/bamf/api/routes/tenants.py (depends on T030)
- [ ] T033 [US1] Add logging for tenant operations in services/bamf/api/routes/tenants.py (depends on T030)
- [ ] T034 [P] [US1] Implement tenant management CLI commands in cmd/bamf/tenant/create.go, list.go, get.go, update.go (depends on T013)
- [ ] T035 [US1] Implement tenant user management CLI commands in cmd/bamf/tenant/users.go (depends on T013)
- [ ] T036 [P] [US1] Create TenantContext React provider in web/src/components/tenant/TenantContext.tsx
- [ ] T037 [P] [US1] Create TenantBranding React component in web/src/components/tenant/TenantBranding.tsx (depends on T036)
- [ ] T038 [P] [US1] Implement tenant management dashboard pages in web/src/pages/admin/tenants/index.tsx, [id]/page.tsx, new.tsx (depends on T036, T037)
- [ ] T039 [US1] Implement tenant creation form in web/src/pages/admin/tenants/new.tsx (depends on T038)
- [ ] T040 [P] [US1] Implement tenant detail view in web/src/pages/admin/tenants/[id]/page.tsx (depends on T038)

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Resource Isolation (Priority: P1)

**Goal**: Users and agents belonging to one tenant cannot access or discover resources belonging to another tenant. All operations are scoped to tenant context derived from authentication.

**Independent Test**: Create two tenants with resources, verify users from one tenant cannot see/access resources of another tenant through CLI, web UI, or API endpoints.

### Tests for User Story 2 (TDD - Write First!) ⚠️

- [ ] T041 [P] [US2] Contract test for tenant context middleware in tests/contract/test_tenant_context_middleware.py
- [ ] T042 [P] [US2] Integration test for resource isolation in tests/integration/test_tenant_isolation.py
- [ ] T043 [P] [US2] Integration test for multi-tenant operations in tests/integration/test_multi_tenant_operations.py
- [ ] T044 [P] [US2] Integration test for agent tenant validation in tests/integration/test_agent_tenant_validation.py

### Implementation for User Story 2

- [ ] T045 [P] [US2] Create TenantResource model in services/bamf/db/models/tenant_resource.py
- [ ] T046 [P] [US2] Create TenantAgent model in services/bamf/db/models/tenant_agent.py
- [ ] T047 [P] [US2] Create TenantCA model in services/bamf/db/models/tenant_ca.py
- [ ] T048 [P] [US2] Create TenantSession model in services/bamf/db/models/tenant_session.py
- [ ] T049 [US2] Implement tenant context extraction service in services/bamf/auth/tenant_context.py (depends on T027)
- [ ] T050 [US2] Implement tenant scoping service (add tenant_id to all queries) in services/bamf/api/services/tenant_scoping_service.py (depends on T049)
- [ ] T051 [US2] Implement agent tenant validation service in services/bamf/api/services/agent_validation_service.py (depends on T046)
- [ ] T052 [US2] Apply tenant scoping middleware to all API routes in services/bamf/api/middleware/tenant.py (depends on T050)
- [ ] T053 [US2] Update resource registration to require tenant association in services/bamf/api/routes/resources.py (depends on T045, T051)
- [ ] T054 [US2] Update agent registration to require tenant association in services/bamf/api/routes/agents.py (depends on T046, T051)
- [ ] T055 [US2] Implement bridge/agent tenant validation in Go pkg/isolation/isolation.go (depends on T013)
- [ ] T056 [US2] Update CLI to derive tenant context from authentication in cmd/bamf/context.go (depends on T013)
- [ ] T057 [P] [US2] Implement SSH tunnel validation with tenant context in services/bamf/api/services/tunnel_validation_service.py (depends on T048)
- [ ] T058 [US2] Update resource list endpoints to filter by tenant_id in services/bamf/api/routes/resources.py (depends on T052)
- [ ] T059 [US2] Add validation and error handling for tenant-scoped operations in services/bamf/api/routes/resources.py (depends on T058)
- [ ] T060 [US2] Add logging for tenant-scoped operations in services/bamf/api/routes/resources.py (depends on T058)
- [ ] T061 [P] [US2] Update CLI resources command to use tenant context in cmd/bamf/resources/list.go (depends on T056)
- [ ] T062 [P] [US2] Update web UI resources page to filter by tenant in web/src/pages/resources/index.tsx (depends on T036)
- [ ] T063 [P] [US2] Update web UI to apply tenant branding in web/src/components/tenant/TenantBranding.tsx (depends on T037)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - Tenant-Specific Configuration (Priority: P2)

**Goal**: Each tenant can configure their own SSO providers, RBAC roles, and certificate policies independent of other tenants.

**Independent Test**: Configure different SSO providers and RBAC rules for two tenants, verify each tenant's configuration only applies to their users and resources.

### Tests for User Story 3 (TDD - Write First!) ⚠️

- [ ] T064 [P] [US3] Integration test for tenant-specific SSO in tests/integration/test_tenant_sso.py
- [ ] T065 [P] [US3] Integration test for tenant-specific RBAC in tests/integration/test_tenant_rbac.py
- [ ] T066 [P] [US3] Integration test for tenant-specific certificate policies in tests/integration/test_tenant_cert_policies.py

### Implementation for User Story 3

- [ ] T067 [US3] Implement tenant SSO configuration service in services/bamf/api/services/sso_config_service.py (depends on T022)
- [ ] T068 [US3] Implement tenant RBAC configuration service in services/bamf/api/services/rbac_config_service.py (depends on T022)
- [ ] T069 [US3] Implement tenant certificate configuration service in services/bamf/api/services/cert_config_service.py (depends on T022)
- [ ] T070 [US3] Implement tenant-specific SSO authentication in services/bamf/auth/sso.py (depends on T067)
- [ ] T071 [US3] Implement tenant-specific RBAC authorization in services/bamf/auth/rbac.py (depends on T068)
- [ ] T072 [US3] Implement tenant-specific certificate issuance in services/bamf/auth/certificates.py (depends on T069)
- [ ] T073 [US3] Implement tenant configuration API endpoints in services/bamf/api/routes/tenant_config.py (depends on T067, T068, T069)
- [ ] T074 [US3] Update RBAC management to use tenant context in services/bamf/api/routes/tenant_users.py (depends on T071)
- [ ] T075 [US3] Add validation and error handling for tenant configuration in services/bamf/api/routes/tenant_config.py (depends on T073)
- [ ] T076 [US3] Add logging for tenant configuration changes in services/bamf/api/routes/tenant_config.py (depends on T073)
- [ ] T077 [P] [US3] Implement CLI configuration commands for tenants in cmd/bamf/tenant/config.go (depends on T013)
- [ ] T078 [P] [US3] Update CLI user role management for tenants in cmd/bamf/tenant/users.go (depends on T013)
- [ ] T079 [P] [US3] Implement tenant configuration web UI pages in web/src/pages/tenants/config/page.tsx (depends on T036)
- [ ] T080 [P] [US3] Implement tenant SSO configuration form in web/src/pages/tenants/config/sso/page.tsx (depends on T079)
- [ ] T081 [P] [US3] Implement tenant RBAC configuration form in web/src/pages/tenants/config/rbac/page.tsx (depends on T079)

**Checkpoint**: At this point, User Stories 1, 2, AND 3 should all work independently

---

## Phase 6: User Story 4 - Billing and Usage Metrics (Priority: P2)

**Goal**: Platform administrators can view usage metrics per tenant, configure usage limits, and each tenant has their own usage report.

**Independent Test**: Generate activity for two tenants, verify usage metrics are accurately tracked per tenant, and platform admins can view comprehensive reports.

### Tests for User Story 4 (TDD - Write First!) ⚠️

- [ ] T082 [P] [US4] Contract test for usage metrics API in tests/contract/test_usage_metrics_api.py
- [ ] T083 [P] [US4] Integration test for usage metrics aggregation in tests/integration/test_usage_metrics.py
- [ ] T084 [P] [US4] Integration test for usage limits enforcement in tests/integration/test_usage_limits.py

### Implementation for User Story 4

- [ ] T085 [P] [US4] Create UsageMetrics model in services/bamf/db/models/usage_metrics.py
- [ ] T086 [US4] Implement usage metrics aggregation service in services/bamf/api/services/usage_aggregation_service.py (depends on T085)
- [ ] T087 [US4] Implement Redis-based usage metrics counters in services/bamf/db/redis_usage_counters.py (depends on T085)
- [ ] T088 [US4] Implement background job for usage metrics rollup (hourly/daily/monthly) in services/bamf/jobs/usage_rollup_job.py (depends on T086)
- [ ] T089 [US4] Implement usage metrics service in services/bamf/api/services/usage_metrics_service.py (depends on T085)
- [ ] T090 [US4] Implement usage limits enforcement service in services/bamf/api/services/usage_limits_service.py (depends on T089)
- [ ] T091 [US4] Implement usage metrics API endpoints in services/bamf/api/routes/usage_metrics.py (depends on T089, T090)
- [ ] T092 [US4] Implement usage metrics export endpoint (CSV/JSON) in services/bamf/api/routes/usage_metrics.py (depends on T091)
- [ ] T093 [US4] Add validation and error handling for usage metrics in services/bamf/api/routes/usage_metrics.py (depends on T091)
- [ ] T094 [US4] Add logging for usage metrics in services/bamf/api/routes/usage_metrics.py (depends on T091)
- [ ] T095 [P] [US4] Implement CLI usage metrics commands in cmd/bamf/tenant/usage.go (depends on T013)
- [ ] T096 [P] [US4] Implement tenant usage metrics dashboard in web/src/pages/admin/tenants/[id]/usage/page.tsx (depends on T036)
- [ ] T097 [P] [US4] Implement usage metrics export in web UI in web/src/pages/admin/tenants/[id]/usage/export/page.tsx (depends on T096)
- [ ] T098 [US4] Implement tenant usage metrics reporting for tenant users in web/src/pages/tenants/usage/page.tsx (depends on T036)

**Checkpoint**: At this point, User Stories 1, 2, 3, AND 4 should all work independently

---

## Phase 7: User Story 5 - Tenant Management Dashboard (Priority: P3)

**Goal**: Platform administrators have a dedicated dashboard to view all tenants, their status, resource counts, recent activity, and perform bulk operations.

**Independent Test**: Create multiple tenants with activity, verify dashboard accurately reflects all tenants' status and metrics.

### Tests for User Story 5 (TDD - Write First!) ⚠️

- [ ] T099 [P] [US5] Integration test for tenant management dashboard in tests/integration/test_tenant_dashboard.py

### Implementation for User Story 5

- [ ] T100 [P] [US5] Implement tenant suspension service in services/bamf/api/services/tenant_lifecycle_service.py (depends on T025)
- [ ] T101 [P] [US5] Implement tenant resumption service in services/bamf/api/services/tenant_lifecycle_service.py (depends on T025)
- [ ] T102 [P] [US5] Implement tenant deletion service in data retention in services/bamf/api/services/tenant_lifecycle_service.py (depends on T025)
- [ ] T103 [US5] Implement tenant lifecycle API endpoints (suspend/resume/delete) in services/bamf/api/routes/tenants.py (depends on T100, T101, T102)
- [ ] T104 [US5] Implement session termination for suspended/deleted tenants in services/bamf/api/services/session_termination_service.py (depends on T102)
- [ ] T105 [US5] Add validation and error handling for tenant lifecycle operations in services/bamf/api/routes/tenants.py (depends on T103)
- [ ] T106 [US5] Add logging for tenant lifecycle operations in services/bamf/api/routes/tenants.py (depends on T103)
- [ ] T107 [P] [US5] Implement CLI tenant lifecycle commands in cmd/bamf/tenant/suspend.go, resume.go, delete.go (depends on T013)
- [ ] T108 [P] [US5] Implement tenant management dashboard UI in web/src/pages/admin/tenants/dashboard/page.tsx (depends on T036)
- [ ] T109 [P] [US5] Implement tenant status indicators and activity tracking in web/src/pages/admin/tenants/dashboard/page.tsx (depends on T108)
- [ ] T110 [P] [US5] Implement bulk operations UI for tenants in web/src/pages/admin/tenants/dashboard/page.tsx (depends on T108)

**Checkpoint**: All user stories should now be independently functional

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T111 [P] Update documentation for multi-tenant features in docs/admin/tenants.md, docs/admin/rbac.md, docs/admin/sso.md
- [ ] T112 [P] Update API documentation (OpenAPI/Swagger) for tenant endpoints in services/bamf/api/docs/
- [ ] T113 [P] Update CLI documentation for tenant commands in docs/reference/cli.md
- [ ] T114 [P] Update quickstart guide with multi-tenant onboarding in docs/getting-started.md
- [ ] T115 Code cleanup and refactoring for multi-tenant code
- [ ] T116 Performance optimization for multi-tenant queries (ensure indexed tenant_id usage, cache tuning)
- [ ] T117 [P] Additional unit tests for edge cases in tests/unit/test_tenant_edge_cases.py
- [ ] T118 Security hardening (Semgrep rules for tenant_id validation, Nuclei templates for cross-tenant access testing)
- [ ] T119 Run quickstart.md validation (create tenant, configure SSO, onboard users, verify isolation)
- [ ] T120 Run all tests (unit, integration, contract, E2E) and ensure 80%+ coverage
- [ ] T121 Run linters and fix all errors (ruff, golangci-lint, eslint)
- [ ] T122 Run type checkers and fix all errors (mypy, tsc)
- [ ] T123 Run security scans (Semgrep, Trivy) and fix all vulnerabilities

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-7)**: All depend on Foundational phase completion
  - User Story 1 (P1): Can start after Foundational - No dependencies on other stories
  - User Story 2 (P1): Can start after Foundational - No dependencies on other stories
  - User Story 3 (P2): Can start after Foundational - May integrate with US1/US2 but independently testable
  - User Story 4 (P2): Can start after Foundational - May integrate with US1/US2/US3 but independently testable
  - User Story 5 (P3): Can start after Foundational - May integrate with US1/US2/US3/US4 but independently testable
- **Polish (Phase 8)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P1)**: Can start after Foundational (Phase 2) - May integrate with US1 but should be independently testable
- **User Story 3 (P2)**: Can start after Foundational (Phase 2) - May integrate with US1/US2 but should be independently testable
- **User Story 4 (P2)**: Can start after Foundational (Phase 2) - May integrate with US1/US2/US3 but should be independently testable
- **User Story 5 (P3)**: Can start after Foundational (Phase 2) - May integrate with US1/US2/US3/US4 but should be independently testable

### Within Each User Story

- Tests (TDD) MUST be written and FAIL before implementation
- Models before services
- Services before endpoints
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- All Foundational tasks marked [P] can run in parallel (within Phase 2)
- Once Foundational phase completes, all user stories can start in parallel (if team capacity allows)
- All tests for a user story marked [P] can run in parallel
- Models within a story marked [P] can run in parallel
- Different user stories can be worked on in parallel by different team members

---

## Parallel Example: User Story 1

```bash
# Launch all tests for User Story 1 together:
Task: "Contract test for tenant creation API endpoint in tests/contract/test_tenant_api.py"
Task: "Contract test for tenant management API endpoints in tests/contract/test_tenant_management_api.py"
Task: "Integration test for tenant onboarding flow in tests/integration/test_tenant_onboarding.py"
Task: "Integration test for tenant branding application in tests/integration/test_tenant_branding.py"

# Launch all models for User Story 1 together:
Task: "Create Tenant model in services/bamf/db/models/tenant.py"
Task: "Create TenantUser model in services/bamf/db/models/tenant_user.py"
Task: "Create PlatformAdmin model in services/bamf/db/models/platform_admin.py"

# Launch all React components together:
Task: "Create TenantContext React provider in web/src/components/tenant/TenantContext.tsx"
Task: "Create TenantBranding React component in web/src/components/tenant/TenantBranding.tsx"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1 (Tenant Onboarding)
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo MVP (tenant creation, branding, user assignment)

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo (Isolation)
4. Add User Story 3 → Test independently → Deploy/Demo (Configuration)
5. Add User Story 4 → Test independently → Deploy/Demo (Metrics)
6. Add User Story 5 → Test independently → Deploy/Demo (Dashboard)
7. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1 (Tenant Onboarding)
   - Developer B: User Story 2 (Resource Isolation)
   - Developer C: User Story 3 (Configuration)
   - Developer D: User Story 4 (Metrics)
   - Developer E: User Story 5 (Dashboard)
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- **TDD REQUIRED**: Verify tests fail before implementing
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
- Constitution requires 80%+ test coverage, critical paths 90%+
- Security testing (Semgrep, Trivy, Nuclei) must pass for multi-tenant code
