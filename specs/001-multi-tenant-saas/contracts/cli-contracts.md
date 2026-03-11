# CLI Contracts: Multi-Tenant SaaS Support

**Feature**: Multi-Tenant SaaS Support
**Date**: 2026-03-11
**Purpose**: Define CLI command contracts for tenant management operations

## Overview

This document defines the CLI command contracts for multi-tenant SaaS capabilities, including tenant management commands (for platform admins) and tenant-scoped operations.

## Command Structure

All commands follow POSIX conventions:

```bash
bamf [global-flags] [command] [subcommand] [flags] [args]
```

**Global Flags**:
- `--help, -h`: Show help
- `--version, -v`: Show version
- `--verbose`: Enable verbose output
- `--quiet, -q`: Suppress output (except errors)
- `--json`: Output JSON instead of human-readable text
- `--api-url`: API server URL (default: from config)
- `--config`: Config file path

## Authentication

CLI commands require authentication via login command:

```bash
bamf login --api https://bamf.example.com
```

Login derives tenant context from SSO authentication. No explicit `--tenant` flag is needed.

## Output Formats

**Human-Readable** (default):
```
Tenant: acme-corp
Name: Acme Corporation
Status: active
Resources: 42
Active Tunnels: 15
Created: 2026-03-01 10:00:00 UTC
```

**JSON** (`--json`):
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "name": "Acme Corporation",
  "status": "active",
  "resource_count": 42,
  "active_tunnels": 15,
  "created_at": "2026-03-01T10:00:00Z"
}
```

---

## Platform Admin Commands

### Create Tenant

Create a new tenant.

```bash
bamf tenant create [flags]
```

**Flags**:
- `--identifier`: Tenant identifier (required, 3-50 chars, alphanumeric with hyphens)
- `--name`: Tenant display name (required)
- `--logo-url`: URL to tenant logo image
- `--domain`: Custom domain for web UI (optional)
- `--max-tunnels`: Maximum concurrent tunnels (default: 1000)
- `--max-users`: Maximum users (default: 100)
- `--max-storage-gb`: Maximum storage in GB (default: 1000)

**Example**:
```bash
bamf tenant create \
  --identifier acme-corp \
  --name "Acme Corporation" \
  --logo-url https://example.com/logo.png \
  --domain bamf.acme.com \
  --max-tunnels 1000
```

**Output** (human-readable):
```
Created tenant: acme-corp
ID: tenant-uuid
Name: Acme Corporation
Status: active
Created: 2026-03-11 10:00:00 UTC
```

**Output** (JSON):
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "name": "Acme Corporation",
  "status": "active",
  "created_at": "2026-03-11T10:00:00Z"
}
```

**Exit Codes**:
- `0`: Success
- `1`: Error (tenant identifier already exists, validation error)
- `2`: Authentication error

---

### List Tenants

List all tenants.

```bash
bamf tenant list [flags]
```

**Flags**:
- `--status`: Filter by status (`active`, `suspended`, `deleting`, `deleted`)
- `--limit`: Pagination limit (default: 50, max: 100)
- `--offset`: Pagination offset (default: 0)
- `--format`: Output format (`table`, `json`, default: `table`)

**Example**:
```bash
bamf tenant list --status active --limit 10
```

**Output** (table):
```
IDENTIFIER   NAME                 STATUS   RESOURCES  TUNNELS  LAST ACTIVITY
acme-corp    Acme Corporation     active    42         15        2026-03-11 09:30
beta-inc      Beta Inc             active    15         5         2026-03-11 08:45
```

**Output** (JSON):
```json
{
  "tenants": [
    {
      "identifier": "acme-corp",
      "name": "Acme Corporation",
      "status": "active",
      "resource_count": 42,
      "active_tunnels": 15,
      "last_activity_at": "2026-03-11T09:30:00Z"
    }
  ],
  "total": 1
}
```

---

### Get Tenant

Get details for a specific tenant.

```bash
bamf tenant get <identifier> [flags]
```

**Arguments**:
- `identifier`: Tenant identifier (required)

**Flags**:
- `--json`: Output JSON instead of human-readable text

**Example**:
```bash
bamf tenant get acme-corp
```

**Output**:
```
Tenant: acme-corp
ID: tenant-uuid
Name: Acme Corporation
Status: active
Domain: bamf.acme.com
Logo URL: https://example.com/logo.png

Usage Limits:
  Max Tunnels: 1000
  Max Users: 100
  Max Storage: 1000 GB

Current Usage:
  Resources: 42
  Active Tunnels: 15
  Users: 25
  Storage: 450 GB

Created: 2026-03-01 10:00:00 UTC
Updated: 2026-03-11 10:30:00 UTC
```

**Exit Codes**:
- `0`: Success
- `1`: Error (tenant not found)
- `2`: Authentication error

---

### Update Tenant

Update tenant configuration.

```bash
bamf tenant update <identifier> [flags]
```

**Arguments**:
- `identifier`: Tenant identifier (required)

**Flags**:
- `--name`: New display name
- `--logo-url`: New logo URL
- `--max-tunnels`: New maximum concurrent tunnels
- `--max-users`: New maximum users
- `--max-storage-gb`: New maximum storage in GB

**Example**:
```bash
bamf tenant update acme-corp --name "Acme Corporation Updated" --max-tunnels 2000
```

**Output**:
```
Updated tenant: acme-corp
Name: Acme Corporation Updated
Max Tunnels: 2000
Updated: 2026-03-11 10:30:00 UTC
```

**Exit Codes**:
- `0`: Success
- `1`: Error (tenant not found, validation error)
- `2`: Authentication error

---

### Suspend Tenant

Suspend a tenant.

```bash
bamf tenant suspend <identifier> [flags]
```

**Arguments**:
- `identifier`: Tenant identifier (required)

**Flags**:
- `--reason`: Suspension reason (required)

**Example**:
```bash
bamf tenant suspend acme-corp --reason "Terms of service violation"
```

**Output**:
```
Suspended tenant: acme-corp
Reason: Terms of service violation
Suspended: 2026-03-11 10:30:00 UTC
```

**Exit Codes**:
- `0`: Success
- `1`: Error (tenant not found, already suspended)
- `2`: Authentication error

---

### Resume Tenant

Resume a suspended tenant.

```bash
bamf tenant resume <identifier>
```

**Arguments**:
- `identifier`: Tenant identifier (required)

**Example**:
```bash
bamf tenant resume acme-corp
```

**Output**:
```
Resumed tenant: acme-corp
Resumed: 2026-03-11 11:00:00 UTC
```

**Exit Codes**:
- `0`: Success
- `1`: Error (tenant not found, not suspended)
- `2`: Authentication error

---

### Delete Tenant

Delete a tenant.

```bash
bamf tenant delete <identifier> [flags]
```

**Arguments**:
- `identifier`: Tenant identifier (required)

**Flags**:
- `--retention-days`: Data retention period in days (required: 0, 30, or 90)
- `--force`: Skip confirmation prompt

**Example**:
```bash
bamf tenant delete acme-corp --retention-days 30
```

**Prompt** (without `--force`):
```
Are you sure you want to delete tenant 'acme-corp'?
This will:
- Suspend the tenant immediately
- Archive all data for 30 days
- Permanently delete data after 30 days

Type 'yes' to confirm: yes
```

**Output**:
```
Deleting tenant: acme-corp
Retention Period: 30 days
Data Permanent Delete: 2026-04-10 11:00:00 UTC
Status: deleting
```

**Exit Codes**:
- `0`: Success
- `1`: Error (tenant not found, validation error)
- `2`: Authentication error

---

## Tenant User Commands (Tenant Admin Only)

### Create Tenant User

Create a new user in the tenant.

```bash
bamf user create [flags]
```

**Flags**:
- `--email`: User email (required)
- `--name`: User display name
- `--roles`: Comma-separated list of roles (required, e.g., `admin,developer`)

**Example**:
```bash
bamf user create \
  --email newuser@acme.com \
  --name "New User" \
  --roles developer
```

**Output**:
```
Created user: newuser@acme.com
Name: New User
Roles: developer
Status: active
Created: 2026-03-11 10:00:00 UTC
```

**Exit Codes**:
- `0`: Success
- `1`: Error (email already exists, invalid role)
- `2`: Authentication error

---

### List Tenant Users

List users in the tenant.

```bash
bamf user list [flags]
```

**Flags**:
- `--status`: Filter by status (`active`, `disabled`)
- `--role`: Filter by role
- `--limit`: Pagination limit (default: 50, max: 100)
- `--offset`: Pagination offset (default: 0)
- `--format`: Output format (`table`, `json`)

**Example**:
```bash
bamf user list --status active --format table
```

**Output** (table):
```
EMAIL                NAME            ROLES            STATUS    CREATED
user@acme.com        User            admin,developer  active     2026-03-01 10:00
newuser@acme.com      New User        developer        active     2026-03-11 10:00
```

---

### Update User Roles

Update user roles.

```bash
bamf user update <email> [flags]
```

**Arguments**:
- `email`: User email (required)

**Flags**:
- `--roles`: Comma-separated list of roles (required)

**Example**:
```bash
bamf user update user@acme.com --roles admin,developer,auditor
```

**Output**:
```
Updated user: user@acme.com
Roles: admin,developer,auditor
Updated: 2026-03-11 10:30:00 UTC
```

---

## Tenant Configuration Commands (Tenant Admin Only)

### Get Tenant Configuration

Get tenant configuration.

```bash
bamf config get
```

**Output**:
```
SSO Configuration:
  Type: oidc
  Issuer: https://acme.okta.com/oauth2/default

RBAC Configuration:
  Roles:
    - admin: read, write, delete, admin
    - developer: read, write

Certificate Configuration:
  Default TTL: 3600 seconds
  Key Algorithm: rsa_2048
```

---

### Update Tenant Configuration

Update tenant configuration.

```bash
bamf config update [flags]
```

**Flags**:
- `--sso-type`: SSO type (`oidc`, `saml`)
- `--sso-issuer`: SSO issuer URL
- `--cert-ttl`: Default certificate TTL in seconds
- `--cert-key-algo`: Certificate key algorithm (`rsa_2048`, `rsa_4096`, `ecdsa_p256`)

**Example**:
```bash
bamf config update --cert-ttl 7200 --cert-key-algo rsa_4096
```

**Output**:
```
Updated tenant configuration:
  Certificate TTL: 7200 seconds
  Key Algorithm: rsa_4096
Updated: 2026-03-11 10:30:00 UTC
```

---

## Usage Metrics Commands (Platform Admin Only)

### Get Tenant Usage Metrics

Get usage metrics for a tenant.

```bash
bamf usage get <identifier> [flags]
```

**Arguments**:
- `identifier`: Tenant identifier (required)

**Flags**:
- `--period`: Aggregation period (`hourly`, `daily`, `monthly`, required)
- `--start-date`: Start date (ISO 8601, required)
- `--end-date`: End date (ISO 8601, required)

**Example**:
```bash
bamf usage get acme-corp \
  --period daily \
  --start-date 2026-03-01 \
  --end-date 2026-03-31
```

**Output** (human-readable):
```
Tenant: acme-corp
Period: daily
Date Range: 2026-03-01 to 2026-03-31

Date        Resources  Avg Tunnels  Session Hrs  Storage (GB)  API Requests
2026-03-01  42         15           120          450           10000
2026-03-02  43         17           125          460           10500
...
```

**Exit Codes**:
- `0`: Success
- `1`: Error (tenant not found, invalid date range)
- `2`: Authentication error

---

### Export Usage Report

Export usage report for a tenant.

```bash
bamf usage export <identifier> [flags]
```

**Arguments**:
- `identifier`: Tenant identifier (required)

**Flags**:
- `--period`: Aggregation period (`hourly`, `daily`, `monthly`, required)
- `--start-date`: Start date (ISO 8601, required)
- `--end-date`: End date (ISO 8601, required)
- `--format`: Export format (`csv`, `json`, default: `csv`)
- `--output`: Output file path (default: stdout)

**Example**:
```bash
bamf usage export acme-corp \
  --period daily \
  --start-date 2026-03-01 \
  --end-date 2026-03-31 \
  --format csv \
  --output usage-report.csv
```

**Output**:
```
Exported usage report to: usage-report.csv
Tenant: acme-corp
Period: daily
Date Range: 2026-03-01 to 2026-03-31
Records: 31
```

---

## Tenant-Scoped Resource Commands

All existing resource commands (`bamf ssh`, `bamf resources`, `bamf sessions`, etc.) remain unchanged but automatically use the authenticated user's tenant context.

**Example**: List Resources

```bash
bamf resources list
```

**Behavior**: Lists only resources belonging to the authenticated user's tenant.

**Output**:
```
ID              TYPE   NAME           HOST              PORT  STATUS
resource-uuid   ssh    prod-server     prod.example.com   22    active
resource-uuid   db     postgres-db     db.example.com     5432  active
```

**Cross-Tenant Access Attempt**:

If a user tries to access a resource from another tenant (e.g., via direct ID):

```bash
bamf ssh user@other-tenant-resource-uuid
```

**Output**:
```
Error: Resource not found for tenant 'acme-corp'
Resource ID: other-tenant-resource-uuid
Tenant: acme-corp
```

**Exit Code**: `1`

---

## Error Messages

All CLI errors follow this format:

```
Error: <error message>
Code: <error-code>
Details: <additional details>
Request ID: <correlation-id>
```

**Example**:
```
Error: Tenant not found
Code: TENANT_NOT_FOUND
Details: No tenant found with identifier 'unknown-corp'
Request ID: abc-123-def-456
```

---

## Shell Completion

CLI provides shell completion for bash, zsh, and fish.

**Install Completion**:
```bash
# Bash
bamf completion bash > /etc/bash_completion.d/bamf

# Zsh
bamf completion zsh > ~/.zsh/completion/_bamf

# Fish
bamf completion fish > ~/.config/fish/completions/bamf.fish
```

**Completion Includes**:
- Command names
- Subcommand names
- Flag names
- Flag values (for enumerated types like `--status`, `--role`)

---

## Configuration

CLI configuration stored in `~/.config/bamf/config.yaml`.

**Example Config**:
```yaml
api_url: https://bamf.example.com
default_format: table
verbose: false
```

Tenant context is derived from authentication token, not stored in config.

---

## Exit Codes

Standard exit codes for all commands:

- `0`: Success
- `1`: Error (validation, not found, permission denied)
- `2`: Authentication error
- `3`: Network error
- `4`: Configuration error
- `127`: Command not found

---

## Platform Admin Context

Platform admin commands (`bamf tenant *`, `bamf usage *`) require platform admin authentication.

**Platform Admin Login**:
```bash
bamf login --api https://bamf.example.com --admin
```

Platform admin token has `is_platform_admin: true` claim and `tenant_id: null`.

**Tenant Context for Regular Users**:
Regular users have `tenant_id` set in their JWT token, so all commands automatically use that tenant context.

**Multi-Tenant Users**:
If a user belongs to multiple tenants, the CLI presents a selection prompt:

```
You belong to multiple tenants. Select one:
  [1] acme-corp (Acme Corporation)
  [2] beta-inc (Beta Inc)

Enter selection [1-2]: 1

Selected tenant: acme-corp
```

---

## Verbose Mode

Verbose mode (`--verbose`) provides detailed logging:

```
[DEBUG] Sending request to: POST https://bamf.example.com/api/v1/admin/tenants
[DEBUG] Request body: {"identifier":"acme-corp","name":"Acme Corporation"}
[DEBUG] Response status: 201 Created
[DEBUG] Response time: 245ms
Created tenant: acme-corp
```

---

## Quiet Mode

Quiet mode (`--quiet`) suppresses all output except errors.

```bash
bamf tenant list --quiet
```

**Output**: (no output on success)
**On Error**: Error message only

---

## JSON Output

JSON mode (`--json`) outputs machine-readable JSON:

```bash
bamf tenant get acme-corp --json
```

**Output**:
```json
{
  "id": "tenant-uuid",
  "identifier": "acme-corp",
  "name": "Acme Corporation",
  "status": "active"
}
```

Useful for scripting and automation.
