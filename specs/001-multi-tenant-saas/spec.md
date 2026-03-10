# Feature Specification: Multi-Tenant SaaS Support

**Feature Branch**: `001-multi-tenant-saas`
**Created**: 2026-03-11
**Status**: Draft
**Input**: User description: "add multi tenant feature. the idea is, i want to host this in to a server and provide a SaaS service"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Tenant Onboarding (Priority: P1)

A platform administrator can create a new tenant (organization) with a unique identifier, configure branding (logo, colors, domain), and assign administrative users. The tenant becomes immediately available for provisioning resources and connecting agents.

**Why this priority**: This is the foundational capability for SaaS - without tenant creation, no other multi-tenant features can function. It represents the minimum viable product for multi-tenancy.

**Independent Test**: Can be fully tested by creating a tenant, verifying its unique identifier works, confirming branding appears in the web UI for that tenant, and validating that administrative users can log in as tenant admins without affecting other tenants.

**Acceptance Scenarios**:

1. **Given** the platform admin is logged in, **When** they create a new tenant with identifier "acme-corp" and upload a logo, **Then** the tenant is created with unique ID, resources are isolated from other tenants, and the logo appears in the web UI when users from that tenant log in.
2. **Given** a tenant "acme-corp" exists, **When** the platform admin assigns user "admin@acme.com" as tenant administrator, **Then** that user can log in and manage resources for "acme-corp" but cannot see or manage any other tenant's resources.
3. **Given** two tenants "acme-corp" and "beta-inc" exist, **When** an admin from "acme-corp" views resources, **Then** they see only "acme-corp" resources, and admin from "beta-inc" sees only "beta-inc" resources.
4. **Given** a platform admin attempts to create a tenant with an existing identifier, **When** they submit the form, **Then** they receive a clear error message indicating the identifier is already taken and can choose a different one.

---

### User Story 2 - Resource Isolation (Priority: P1)

Users and agents belonging to one tenant cannot access or discover resources (SSH servers, databases, Kubernetes clusters, web apps) belonging to another tenant. All operations are scoped to the tenant context derived from authentication.

**Why this priority**: Resource isolation is the core security requirement for multi-tenancy. Without it, tenants could accidentally or maliciously access each other's infrastructure, violating SaaS security guarantees and creating compliance risks.

**Independent Test**: Can be tested by creating two tenants, provisioning resources for each, and verifying that users from one tenant cannot see, access, or affect resources of the other tenant through CLI, web UI, or any API endpoint.

**Acceptance Scenarios**:

1. **Given** tenant "acme-corp" has SSH server "prod-server" registered and tenant "beta-inc" has SSH server "db-server" registered, **When** an "acme-corp" user runs `bamf ssh user@db-server`, **Then** the connection fails with a clear "resource not found" or "access denied" message.
2. **Given** both tenants have resources registered, **When** an "acme-corp" user browses resources in the web UI, **Then** only "acme-corp" resources are visible in all lists, dashboards, and dropdowns.
3. **Given** an "acme-corp" agent is connected, **When** it registers a new resource, **Then** that resource is automatically associated with "acme-corp" and is visible only to "acme-corp" users.
4. **Given** a user logs in with SSO that includes a tenant identifier claim, **When** they access any API or CLI command, **Then** all operations are scoped to that tenant without the user needing to specify tenant context explicitly.

---

### User Story 3 - Tenant-Specific Configuration (Priority: P2)

Each tenant can configure their own SSO providers, RBAC roles, certificate policies, and audit retention settings independent of other tenants. Platform-level settings (like overall system capacity limits) remain under platform admin control.

**Why this priority**: This enables tenants to have control over their security posture and compliance requirements while allowing the SaaS provider to maintain overall system governance. It's critical for enterprise customers who have specific security and compliance needs.

**Independent Test**: Can be tested by configuring different SSO providers and RBAC rules for two tenants, then verifying that each tenant's configuration only applies to their users and resources.

**Acceptance Scenarios**:

1. **Given** tenant "acme-corp" configures Okta SSO and tenant "beta-inc" configures Azure AD SSO, **When** users from each tenant log in, **Then** they are redirected to their tenant's configured IdP and receive appropriate permissions based on their identity provider.
2. **Given** "acme-corp" defines a role "developer" with access to SSH servers and "beta-inc" defines a role "auditor" with read-only access, **When** users are assigned these roles, **Then** they have exactly the permissions defined for their tenant's role, with no cross-tenant role leakage.
3. **Given** "acme-corp" sets a 90-day certificate TTL while "beta-inc" sets a 30-day TTL, **When** certificates are issued for each tenant, **Then** the respective TTL policies are enforced independently.
4. **Given** a tenant admin modifies their RBAC rules, **When** they save the configuration, **Then** the changes apply immediately to that tenant's users without affecting any other tenant.

---

### User Story 4 - Billing and Usage Metrics (Priority: P2)

Platform administrators can view usage metrics per tenant (number of resources, active tunnels, session hours, storage consumption) and configure billing plans or usage limits. Each tenant has their own usage report.

**Why this priority**: For a SaaS business, the ability to track usage per tenant is essential for billing, capacity planning, and identifying high-value customers. This enables different pricing tiers and helps prevent a single tenant from consuming disproportionate resources.

**Independent Test**: Can be tested by generating activity for two tenants, then verifying that usage metrics are accurately tracked per tenant and that platform admins can view comprehensive reports.

**Acceptance Scenarios**:

1. **Given** tenant "acme-corp" has 10 registered resources and 100 active tunnels, and tenant "beta-inc" has 5 resources and 20 tunnels, **When** the platform admin views usage metrics, **Then** each tenant's statistics are displayed separately with accurate counts.
2. **Given** "acme-corp" exceeds their configured concurrent tunnel limit, **When** a user attempts to establish a new tunnel, **Then** they receive a clear "limit exceeded" message with upgrade options, while "beta-inc" users are unaffected.
3. **Given** the billing period ends, **When** usage reports are generated, **Then** each tenant receives a report showing only their resource consumption (tunnel hours, storage, API calls) without seeing other tenants' data.
4. **Given** a platform admin configures different usage limits for "acme-corp" (1000 tunnels) and "beta-inc" (100 tunnels), **When** tenants reach their limits, **Then** enforcement happens per tenant without affecting the other.

---

### User Story 5 - Tenant Management Dashboard (Priority: P3)

Platform administrators have a dedicated dashboard to view all tenants, their status, resource counts, recent activity, and health metrics. They can perform bulk operations like suspending tenants, exporting configuration, or deleting tenants with data retention controls.

**Why this priority**: This improves operational efficiency for the SaaS provider by allowing them to manage many tenants from a single interface. It's not required for basic multi-tenancy but becomes important as the customer base grows.

**Independent Test**: Can be tested by creating multiple tenants, generating activity, and verifying the dashboard accurately reflects all tenants' status and metrics.

**Acceptance Scenarios**:

1. **Given** 10 tenants exist with varying levels of activity, **When** the platform admin opens the tenant management dashboard, **Then** they see a table listing all tenants with columns for name, status (active/suspended), resource count, last activity, and health indicators.
2. **Given** a tenant has not been used for 30 days, **When** the platform admin views the dashboard, **Then** the tenant is flagged as "inactive" for potential follow-up.
3. **Given** a tenant violates terms of service, **When** the platform admin clicks "suspend", **Then** the tenant immediately cannot establish new tunnels or access resources, existing sessions are terminated gracefully, and all users see a suspension notice.
4. **Given** a tenant needs to be deleted, **When** the platform admin initiates deletion, **Then** they can choose data retention period (e.g., 0, 30, or 90 days), receive confirmation of what will be deleted, and the deletion occurs in the background with progress tracking.

---

### Edge Cases

- What happens when a tenant is deleted while active tunnels exist?
- How does the system handle a tenant's SSO provider becoming unavailable or misconfigured?
- What occurs when resource usage limits are reached during critical operations?
- How are tenant-specific audit logs handled during tenant suspension or deletion?
- What happens if two tenants attempt to register the same external resource (e.g., overlapping IP addresses)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST support creation of multiple tenants with globally unique identifiers
- **FR-002**: Each tenant MUST have isolated resource namespaces with no cross-tenant visibility
- **FR-003**: Authentication MUST include tenant context derived from SSO claims, email domain, or explicit selection
- **FR-004**: All API, CLI, and web UI operations MUST automatically scope to the authenticated user's tenant
- **FR-005**: Agents MUST be explicitly associated with a single tenant during registration
- **FR-006**: Certificate authorities (CA) MUST be scoped per tenant to prevent cross-tenant certificate misuse
- **FR-007**: Role-based access control (RBAC) policies MUST be configurable per tenant
- **FR-008**: Each tenant MUST be able to configure their own SSO providers (OIDC, SAML, local auth)
- **FR-009**: Audit logs MUST be isolated per tenant and accessible only to users with appropriate permissions
- **FR-010**: System MUST track and report usage metrics (resources, tunnels, sessions, storage) per tenant
- **FR-011**: Platform administrators MUST have ability to view all tenant data without tenant-specific restrictions
- **FR-012**: Tenant administrators MUST have full administrative rights within their tenant only
- **FR-013**: System MUST support tenant suspension (disable new sessions, terminate existing sessions gracefully)
- **FR-014**: System MUST support tenant deletion with configurable data retention periods
- **FR-015**: Platform administrators MUST be able to configure resource limits per tenant (tunnels, storage, users)
- **FR-016**: Each tenant MUST be able to configure branding elements (logo, colors, domain mapping)
- **FR-017**: Session recordings MUST be isolated per tenant with playback access restricted to tenant users
- **FR-018**: Bridge and agent connections MUST validate tenant membership before establishing tunnels
- **FR-019**: System MUST provide a tenant management dashboard for platform administrators
- **FR-020**: Tenant-specific settings (SSO, RBAC, certificates) MUST not affect other tenants' configurations

### Key Entities

- **Tenant**: Represents an organization/customer with unique identifier, branding configuration, SSO settings, RBAC policies, and usage limits
- **TenantUser**: User account belonging to a specific tenant with roles scoped to that tenant
- **PlatformAdmin**: User with system-wide administrative rights across all tenants
- **TenantResource**: Infrastructure resource (server, database, cluster) registered to a specific tenant
- **TenantAgent**: Agent process registered to and belonging to a single tenant
- **TenantCA**: Certificate authority instance scoped to a specific tenant for issuing certificates
- **TenantSession**: Active user session scoped to a specific tenant with associated tunnel connections
- **UsageMetrics**: Per-tenant usage tracking for billing and capacity management (tunnel hours, storage, resource counts)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Platform administrators can onboard new tenants in under 5 minutes including SSO configuration
- **SC-002**: System supports 100+ concurrent tenants without performance degradation
- **SC-003**: Users can access their tenant resources without explicitly specifying tenant context in 95% of scenarios
- **SC-004**: Resource isolation prevents any cross-tenant data leakage under all conditions
- **SC-005**: Usage metrics are accurate within 1% for all tracked dimensions (tunnels, sessions, storage)
- **SC-006**: Platform administrators can manage tenant lifecycle (create/suspend/delete) from a single dashboard
- **SC-007**: Each tenant's configuration changes (SSO, RBAC) take effect in under 30 seconds
- **SC-008**: System handles tenant deletion within 5 minutes for tenants with up to 10,000 resources
- **SC-009**: 99.9% of tenant operations succeed without cross-tenant interference
- **SC-010**: Platform administrators can view and export usage reports for any tenant within 10 seconds

## Assumptions

- Existing BAMF users will be migrated to a default "legacy" tenant to preserve their data and configuration
- Tenant identifiers must be globally unique and cannot be changed after creation
- Platform administrators are trusted and have full access to all tenant data for troubleshooting purposes
- SSO providers will include tenant identifier claims; otherwise, email domain mapping or manual selection will be used
- Resource limits will be enforced at tenant level to prevent any single tenant from consuming disproportionate resources
- Existing single-tenant architecture can be refactored to support multi-tenancy with minimal breaking changes
- Database schema will require migration to add tenant identifiers to all tenant-scoped tables
- Agent registration flow will require tenant association as a mandatory field

## Dependencies

- Existing BAMF infrastructure (API, bridge, agents, web UI) must continue to function during multi-tenant implementation
- Database migration strategy must preserve existing data and configuration
- SSO integration must support extracting tenant context from identity provider claims
- Certificate authority architecture must be refactored to support per-tenant CA instances
- Audit logging system must be enhanced to include tenant filtering and isolation
