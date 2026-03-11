# Quickstart Guide: Multi-Tenant SaaS Support

**Feature**: Multi-Tenant SaaS Support
**Date**: 2026-03-11
**Purpose**: Get started with multi-tenant BAMF deployment

## Overview

This guide walks you through setting up BAMF as a multi-tenant SaaS platform, including tenant onboarding, user management, and resource isolation.

## Prerequisites

- BAMF deployed and accessible (see [Deployment Guide](../../docs/admin/deployment.md))
- Platform admin credentials
- Access to API server at `https://bamf.example.com`
- CLI installed: `bamf` command available

---

## Step 1: Log in as Platform Admin

Login as platform admin to manage tenants.

```bash
bamf login --api https://bamf.example.com --admin
```

Follow the SSO login flow to authenticate.

**Verify Login**:
```bash
bamf --version
bamf whoami
```

Output:
```
Logged in as: platform-admin@example.com
Role: platform_admin
Tenant: N/A (system-wide access)
```

---

## Step 2: Create Your First Tenant

Create a tenant for an organization.

```bash
bamf tenant create \
  --identifier acme-corp \
  --name "Acme Corporation" \
  --logo-url https://acme.com/logo.png \
  --domain bamf.acme.com \
  --max-tunnels 1000 \
  --max-users 100
```

Output:
```
Created tenant: acme-corp
ID: 550e8400-e29b-41d4-a716-446655440000
Name: Acme Corporation
Status: active
Created: 2026-03-11 10:00:00 UTC
```

**View Tenant Details**:
```bash
bamf tenant get acme-corp
```

---

## Step 3: Configure Tenant SSO

Configure SSO for the tenant (optional, but recommended).

**Option A: OIDC (Okta, Google, Azure AD, etc.)**

Access the web UI at `https://bamf.acme.com/admin/settings/sso` and configure:
- Issuer URL (e.g., `https://acme.okta.com/oauth2/default`)
- Client ID and Client Secret
- Scopes and claims

**Option B: SAML**

Access the web UI at `https://bamf.acme.com/admin/settings/sso` and configure:
- IdP URL
- IdP certificate
- SAML assertion attributes

**Option C: Skip SSO**

Tenant can use local authentication with username/password (not recommended for production).

---

## Step 4: Create Tenant Admin User

Create the first user for the tenant (tenant admin).

```bash
bamf user create \
  --email admin@acme.com \
  --name "Acme Admin" \
  --roles admin
```

Output:
```
Created user: admin@acme.com
Name: Acme Admin
Roles: admin
Status: active
Created: 2026-03-11 10:05:00 UTC
```

**Important**: This user will be the tenant administrator with full access to tenant resources.

---

## Step 5: Log in as Tenant Admin

Logout as platform admin and log in as tenant admin.

```bash
# Logout
bamf logout

# Login as tenant admin
bamf login --api https://bamf.acme.com
```

Follow the SSO login flow to authenticate.

**Verify Tenant Context**:
```bash
bamf whoami
```

Output:
```
Logged in as: admin@acme.com
Role: tenant_admin
Tenant: acme-corp (Acme Corporation)
```

---

## Step 6: Deploy and Connect Agent

Deploy an agent to register resources for the tenant.

**Generate Join Token** (as tenant admin):

```bash
bamf tokens create --name acme-prod-agents --ttl 24h
```

Output:
```
Join Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Token Name: acme-prod-agents
TTL: 24h
Created: 2026-03-11 10:10:00 UTC
```

**Deploy Agent on Kubernetes**:

```bash
helm install bamf-agent oci://ghcr.io/mattrobinsonsre/bamf \
  --namespace acme-bamf \
  --create-namespace \
  --set agent.enabled=true \
  --set agent.platformUrl=https://bamf.acme.com \
  --set agent.joinToken=<JOIN_TOKEN>
```

**Deploy Agent on VM**:

```bash
curl -L https://github.com/mattrobinsonsre/bamf/releases/latest/download/bamf-agent-linux-amd64 \
  -o /usr/local/bin/bamf-agent && chmod +x /usr/local/bin/bamf-agent

bamf-agent \
  --platform-url https://bamf.acme.com \
  --join-token <JOIN_TOKEN>
```

**Verify Agent Connection**:

```bash
bamf agents list
```

Output:
```
ID                  NAME            HOSTNAME        STATUS    LAST SEEN
agent-uuid         acme-prod-1     k8s-node-1     connected  2026-03-11 10:15:00
```

---

## Step 7: Register Resources

Register resources (SSH servers, databases, K8s clusters) for the tenant.

**Register SSH Server** (via CLI):

```bash
bamf resources create \
  --type ssh \
  --name prod-server \
  --host prod.acme.com \
  --port 22
```

**Register Database** (via CLI):

```bash
bamf resources create \
  --type database \
  --name postgres-db \
  --host db.acme.com \
  --port 5432
```

**View Registered Resources**:

```bash
bamf resources list
```

Output:
```
ID              TYPE       NAME           HOST            PORT  STATUS
resource-uuid   ssh        prod-server     prod.acme.com    22    active
resource-uuid   database   postgres-db     db.acme.com      5432  active
```

---

## Step 8: Test Tenant Isolation

Verify tenant isolation by creating a second tenant and confirming resources are isolated.

**As Platform Admin**:
```bash
bamf login --api https://bamf.example.com --admin

bamf tenant create \
  --identifier beta-inc \
  --name "Beta Inc" \
  --max-tunnels 500
```

**Create Beta Inc User**:
```bash
bamf user create \
  --email admin@beta.com \
  --name "Beta Admin" \
  --roles admin
```

**Log in as Beta Inc Admin**:
```bash
bamf logout
bamf login --api https://bamf.beta-inc.com
```

**Try to Access Acme Corp Resources**:

```bash
bamf resources list
```

Output:
```
No resources found for tenant 'beta-inc'
```

**Try to SSH to Acme Corp Server**:

```bash
bamf ssh user@prod.acme.com
```

Output:
```
Error: Resource not found for tenant 'beta-inc'
Resource ID: prod.acme.com
Tenant: beta-inc
```

**Result**: ✅ Tenant isolation confirmed!

---

## Step 9: View Tenant Usage

Platform admins can view usage metrics per tenant.

```bash
bamf login --api https://bamf.example.com --admin

bamf usage get acme-corp \
  --period daily \
  --start-date 2026-03-01 \
  --end-date 2026-03-31
```

Output:
```
Tenant: acme-corp
Period: daily
Date Range: 2026-03-01 to 2026-03-31

Date        Resources  Avg Tunnels  Session Hrs  Storage (GB)  API Requests
2026-03-01  2          0            0            0             10
2026-03-02  2          1            2            0             25
...
```

**Export Usage Report**:

```bash
bamf usage export acme-corp \
  --period daily \
  --start-date 2026-03-01 \
  --end-date 2026-03-31 \
  --format csv \
  --output acme-usage.csv
```

---

## Step 10: Customize Tenant Branding (Optional)

Customize the tenant's web UI appearance.

**As Platform Admin**:

```bash
bamf tenant update acme-corp \
  --logo-url https://acme.com/new-logo.png
```

**Or via Web UI**:

Access `https://bamf.acme.com/admin/settings/branding` and configure:
- Logo URL
- Primary color
- Secondary color
- Custom CSS

**Apply Custom Domain** (requires DNS configuration):

1. Configure DNS CNAME: `bamf.acme.com` → `bamf.example.com`
2. Configure TLS certificate for custom domain
3. Update tenant:

```bash
bamf tenant update acme-corp --domain bamf.acme.com
```

---

## Common Workflows

### Adding More Users to a Tenant

**As Tenant Admin**:

```bash
# Create developer user
bamf user create \
  --email developer@acme.com \
  --name "Acme Developer" \
  --roles developer

# Create auditor user
bamf user create \
  --email auditor@acme.com \
  --name "Acme Auditor" \
  --roles auditor
```

### Updating User Roles

**As Tenant Admin**:

```bash
bamf user update developer@acme.com --roles admin,developer
```

### Suspending a Tenant

**As Platform Admin**:

```bash
bamf tenant suspend acme-corp --reason "Payment overdue"
```

**Result**: Tenant immediately blocked from creating new tunnels; existing tunnels complete gracefully.

### Resuming a Suspended Tenant

**As Platform Admin**:

```bash
bamf tenant resume acme-corp
```

### Deleting a Tenant

**As Platform Admin**:

```bash
bamf tenant delete acme-corp --retention-days 30
```

**Result**: Tenant marked as `deleting`, data archived for 30 days, then permanently deleted.

---

## Troubleshooting

### Issue: "Tenant not found"

**Cause**: Tenant identifier incorrect or doesn't exist.

**Solution**:
```bash
# List all tenants to find correct identifier
bamf tenant list
```

### Issue: "User not found for tenant"

**Cause**: User email incorrect or not created in tenant.

**Solution**:
```bash
# List users in tenant
bamf user list
```

### Issue: "Resource not found for tenant"

**Cause**: Trying to access resource from another tenant.

**Solution**: Verify you're logged in to the correct tenant:
```bash
bamf whoami
```

### Issue: Agent not connecting

**Cause**: Agent join token invalid or expired.

**Solution**:
```bash
# Generate new join token
bamf tokens create --name new-token --ttl 24h
```

### Issue: SSO login failing

**Cause**: SSO configuration incorrect or IdP unreachable.

**Solution**:
1. Check SSO configuration in tenant settings
2. Verify IdP issuer URL is correct
3. Check IdP is reachable from BAMF server

---

## Web UI Access

Access the web UI for your tenant:

- **Tenant Admin**: `https://bamf.acme.com/admin`
- **Platform Admin**: `https://bamf.example.com/admin`
- **Regular Users**: `https://bamf.acme.com`

### Web UI Features

- Tenant dashboard with resource overview
- User management (create, update roles)
- Resource registration and management
- Session monitoring and recording playback
- Audit log viewer with tenant filtering
- Tenant configuration (SSO, RBAC, branding)
- Usage metrics dashboard

---

## Next Steps

1. **Configure RBAC**: Define custom roles and permissions for your tenant
2. **Set Up Monitoring**: Configure alerts for usage limits and anomalies
3. **Customize Branding**: Apply your organization's logo and colors
4. **Integrate SSO**: Connect your IdP for centralized authentication
5. **Deploy Agents**: Register more resources across your infrastructure
6. **Enable Session Recording**: Configure session recording for audit compliance

---

## Migration from Single-Tenant BAMF

If you're migrating an existing single-tenant BAMF deployment:

1. **Deploy Multi-Tenant BAMF**: Follow the deployment guide
2. **Migration Runs Automatically**: Existing data is migrated to a "legacy" tenant
3. **Verify Migration**: Check that all resources, users, and sessions are accessible
4. **Create New Tenants**: Onboard additional organizations as needed
5. **Reorganize if Desired**: Migrate data between tenants (manual process)

**Legacy Tenant Details**:
- Identifier: `legacy`
- Name: `Legacy Deployment`
- Status: `active`
- Contains all pre-migration data

---

## Documentation

For more detailed information:

- [Tenant Management Guide](../../docs/admin/tenants.md)
- [RBAC Guide](../../docs/admin/rbac.md)
- [SSO Configuration](../../docs/admin/sso.md)
- [API Reference](../../docs/reference/api.md)
- [CLI Reference](../../docs/reference/cli.md)

---

## Support

- **Documentation**: https://docs.bamf.io
- **Issues**: https://github.com/mattrobinsonsre/bamf/issues
- **Discussions**: https://github.com/mattrobinsonsre/bamf/discussions

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `bamf tenant create` | Create new tenant |
| `bamf tenant list` | List all tenants |
| `bamf tenant get <id>` | Get tenant details |
| `bamf tenant update <id>` | Update tenant config |
| `bamf tenant suspend <id>` | Suspend tenant |
| `bamf tenant resume <id>` | Resume suspended tenant |
| `bamf tenant delete <id>` | Delete tenant |
| `bamf user create` | Create tenant user |
| `bamf user list` | List tenant users |
| `bamf user update <email>` | Update user roles |
| `bamf resources list` | List tenant resources |
| `bamf usage get <id>` | Get tenant usage metrics |
| `bamf whoami` | Show current user and tenant |

---

You're now ready to use BAMF as a multi-tenant SaaS platform! 🎉
