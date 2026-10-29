# VaultKit CLI (`vkit`)

[![Gem Version](https://img.shields.io/badge/gem-v1.2.0-blue.svg)](https://rubygems.org/gems/vaultkit)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Ruby Version](https://img.shields.io/badge/ruby-3.1+-CC342D.svg)](https://www.ruby-lang.org)

> Command-line interface for VaultKit — secure, policy-driven data access control plane

The VaultKit CLI (`vkit`) is the primary interface for interacting with the VaultKit control plane. It enables users to request governed data access, track approvals, manage datasources, and deploy policy bundles—all while maintaining complete audit trails.

---

## 📋 Table of Contents

- [What is VaultKit CLI?](#what-is-vaultkit-cli)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Authentication](#authentication)
- [Core Workflows](#core-workflows)
- [Command Reference](#command-reference)
- [Configuration](#configuration)
- [Advanced Usage](#advanced-usage)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

---

## 🎯 What is VaultKit CLI?

The VaultKit CLI provides command-line access to VaultKit for:

- **Users** - Data analysts, engineers, and administrators
- **AI Agents** - LLMs and autonomous systems requiring governed data access
- **Tools & Services** - CI/CD pipelines, data pipelines, and automated workflows
- **Applications** - Programmatic integration via scripting

| Capability | Description |
|------------|-------------|
| **Authentication** | Login, session management, identity verification |
| **Data Requests** | Submit AQL queries with automatic policy evaluation |
| **Approval Workflows** | Approve or deny access requests requiring authorization |
| **Grant Management** | Fetch data using approved, time-limited grants |
| **Datasource Admin** | Register and manage database connections (admin only) |
| **Schema Discovery** | Scan datasources and detect schema drift |
| **Policy Deployment** | Build, validate, and deploy policy bundles |
| **Audit Queries** | Search and export audit logs |

### Mental Model

VaultKit uses a **request → policy → grant → fetch** workflow:

```
┌──────────┐    ┌──────────┐    ┌───────┐    ┌───────┐    ┌───────────┐
│ Request  │ -> │ Policy   │ -> │ Grant │ -> │ Fetch │ -> │ Audit Log │
│ (AQL)    │    │ Eval     │    │ (JWT) │    │ Data  │    │ (Record)  │
└──────────┘    └──────────┘    └───────┘    └───────┘    └───────────┘
```

**Key Principles:**
- Users **request** data, not queries
- Policies **decide** what happens (allow, mask, deny, require approval)
- Grants **authorize** execution with TTLs
- FUNL **executes** queries safely
- Everything is **audited**

---

## 📦 Installation

### Prerequisites

- **Ruby** 3.1 or higher
- **VaultKit Control Plane** running and accessible

### Option 1: Install from RubyGems (Recommended)

```bash
gem install vaultkit
```

### Verify Installation

```bash
vkit --version
# => VaultKit CLI v0.1.0

vkit --help
```

---

## ⚡ Quick Start

### 1. Set Environment Variables

```bash
export VKIT_API_URL=http://localhost:3000
```

For production:
```bash
export VKIT_API_URL=https://vaultkit.company.com
```

### 2. Authenticate

```bash
vkit login --endpoint http://localhost:3000 --email analyst@company.com
```

You'll receive a prompt for your password or be redirected to SSO.

### 3. Submit Your First Request

```bash
vkit request --aql '{
  "source_table": "customers",
  "columns": ["id", "email", "total_spend"],
  "limit": 10
}'
```

### 4. Fetch Results

If granted immediately:
```bash
vkit fetch --grant gr_abc123xyz
```

If approval required:
```bash
# Check status
vkit requests list --state pending

# After approval
vkit fetch --grant gr_abc123xyz
```

---

## 🔐 Authentication

### `vkit login`

Authenticate with the VaultKit control plane.

```bash
vkit login --endpoint http://localhost:3000 --email analyst@acme.com
```

**Options:**
- `--endpoint` — VaultKit API endpoint (can also use `VKIT_API_URL`)
- `--email` — Email address for authentication

**SSO Integration:**

If your organization uses SSO, you'll be redirected to your identity provider:

```bash
vkit login --endpoint https://vaultkit.company.com --email you@company.com

# Output:
# Opening browser for SSO authentication...
# ✓ Authentication successful
# Session expires: 2024-01-15 18:00:00 UTC
```

### `vkit whoami`

Display the currently authenticated user and session information.

```bash
vkit whoami
```

**Output:**
```
👤 analyst@acme.com (role: analyst, org: acme)
```

**JSON Format:**
```bash
vkit whoami --format json
```

**Output:**
```json
{
  "email": "analyst@acme.com",
  "role": "analyst",
  "org": "acme"
}
```

### `vkit logout`

Clear stored credentials and end session.

```bash
vkit logout
```

### Credentials from the environment

`vkit` normally uses the login saved by `vkit login`. Scripts, CI jobs and demos can
supply credentials for a single command or shell instead, without replacing it:

```bash
export VKIT_ENDPOINT=https://vaultkit.example.com
export VKIT_TOKEN=<a VaultKit user token>
vkit request < query.json
```

- **Both variables are needed.** Together they replace the saved login for every
  command except `vkit login` and `vkit logout`.
- **One alone is an error**, naming the missing variable.
- **A notice goes to stderr** while the override is active. The token is never printed.
- **Unset both** to go back to your saved login.

To act as someone else for one command, set the token on that command only:

```bash
VKIT_TOKEN="$APPROVER_TOKEN" vkit approval:approve 42
```

Requires vkit 1.1.6 or later.

---

## 🔄 Core Workflows

### Workflow 1: Immediate Access (Auto-Approved)

```bash
# 1. Submit request
vkit request --aql '{
  "source_table": "public_reports",
  "columns": ["report_id", "title", "published_date"],
  "limit": 10
}'

# Output:
# ✓ Request granted immediately
# Grant ID: gr_abc123xyz
# TTL: 4 hours
# Use: vkit fetch --grant gr_abc123xyz

# 2. Fetch data
vkit fetch --grant gr_abc123xyz --format table
```

### Workflow 2: Approval Required

```bash
# 1. Submit request
vkit request --aql '{
  "source_table": "financial_transactions",
  "columns": ["transaction_id", "amount", "customer_id"],
  "filters": [{"field": "amount", "operator": "gt", "value": 10000}]
}'

# Output:
# ⏳ Approval required
# Request ID: req_20240115_xyz789
# Approver: finance_manager
# Reason: Financial data requires manager approval
# Status: PENDING

# 2. Check status
vkit requests list --state pending

# 3. After approval, fetch data
vkit fetch --grant gr_approved_abc123
```

### Workflow 3: Denied Access

```bash
vkit request --aql '{
  "source_table": "payment_cards",
  "columns": ["card_number", "cvv"],
  "limit": 10
}'

# Output:
# ❌ Request denied
# Reason: Direct payment card data access forbidden
# Policy: pci_dss_restrictions
# Alternative: Use tokenized_card_id field instead
# Audit ID: audit_20240115_denied_123
```

---

## Command Reference

### Authentication Commands

#### `vkit login`

Authenticate with VaultKit.

```bash
vkit login --endpoint <URL> --email <EMAIL>
```

**Options:**
- `--endpoint, -e` — VaultKit API endpoint (default: `$VKIT_API_URL`)
- `--email` — User email address
- `--password` — Password (prompted if not provided)

**Examples:**
```bash
# Interactive login
vkit login --endpoint http://localhost:3000 --email analyst@company.com

# Non-interactive (CI/CD)
vkit login --endpoint https://vaultkit.company.com --email bot@company.com --password $BOT_PASSWORD
```

#### `vkit whoami`

Display current user information.

```bash
vkit whoami [--format json|table]
```

**Examples:**
```bash
vkit whoami
vkit whoami --format json
```

#### `vkit logout`

End session and clear credentials.

```bash
vkit logout
```

---

### Data Request Commands

#### `vkit request`

Submit a governed data request using AQL.

```bash
vkit request --aql <AQL_JSON> [OPTIONS]
```

**Options:**
- `--aql` — AQL JSON payload (inline or via STDIN)
- `--format` — Output format: `json`, `table` (default: `table`)

Policies see the requester's role, clearance and region from their account, and the
environment and dataset region from the datasource. A request can't set them: the
older `--env`, `--requester_region`, `--dataset_region` and `--requester_clearance`
options are accepted but ignored. Admins set clearance and region on users (console,
Team) and agent tokens (see [Agent Tokens](#agent-tokens)).

**Inline AQL:**
```bash
vkit request \
  --aql '{"source_table":"customers","columns":["email","total_spend"],"limit":100}'
```

**AQL via STDIN:**
```bash
cat query.json | vkit request

# Or
echo '{"source_table":"orders","columns":["order_id","total"]}' | vkit request
```

**AQL from File:**
```bash
vkit request --aql "$(cat queries/customer_analysis.json)"
```

**Possible Outcomes:**

| Outcome | Description | Next Step |
|---------|-------------|-----------|
| ✅ **Granted** | Immediate access | `vkit fetch --grant <GRANT_ID>` |
| ⏳ **Queued** | Requires approval | Wait for approval, check with `vkit requests list` |
| ❌ **Denied** | Policy disallows | Review policy restrictions |

#### `vkit requests list`

View your request history.

```bash
vkit requests list [--state STATE] [--format FORMAT]
```

**Options:**
- `--state` — Filter by state: `all`, `pending`, `approved`, `denied` (default: `all`)
- `--format` — Output format: `json`, `table` (default: `table`)
- `--limit` — Number of results (default: 50)

**Examples:**
```bash
# All requests
vkit requests list

# Only pending requests
vkit requests list --state pending

# Approved requests in JSON
vkit requests list --state approved --format json

# Last 100 requests
vkit requests list --limit 100
```

#### `vkit requests show`

Show details for a specific request.

```bash
vkit requests show <REQUEST_ID>
```

**Example:**
```bash
vkit requests show req_20240115_xyz789
```

---

### Approval Commands

#### `vkit approval:list`

List approval requests you are authorized to act on.

```bash
vkit approval:list [--state STATE] [--format FORMAT]
```

**Options:**
- `--state` — Filter by state: `all`, `pending`, `approved`, `denied` (default: `pending`)
- `--format` — Output format: `json`, `table` (default: `table`)

**Examples:**
```bash
# Pending approvals
vkit approval:list

# All approvals you've handled
vkit approval:list --state all

# JSON output for scripting
vkit approval:list --format json
```

#### `vkit approval:approve`

Approve a pending request.

```bash
vkit approval:approve <REQUEST_ID> [--ttl SECONDS] [--notes "..."]
```

**Options:**
- `--ttl` — Grant lifetime in seconds (default: 3600)
- `--notes` — Approval notes (optional)

**Examples:**
```bash
# Basic approval
vkit approval:approve 42

# With custom TTL and notes
vkit approval:approve 42 \
  --ttl 7200 \
  --notes "Approved for Q4 analysis. Limited to aggregate data only."
```

#### `vkit approval:deny`

Deny a pending request.

```bash
vkit approval:deny <REQUEST_ID> --reason "..."
```

**Options:**
- `--reason` — Required explanation for denial

**Examples:**
```bash
vkit approval:deny 42 --reason "Insufficient business justification"

vkit approval:deny 42 --reason "Request violates data minimization policy"
```

---

### Data Fetch Commands

#### `vkit fetch`

Fetch data using a previously issued grant.

```bash
vkit fetch --grant <GRANT_REF> [--format FORMAT]
```

**Options:**
- `--grant, -g` — Grant reference or ID (required)
- `--format` — Output format: `json`, `table`, `csv` (default: `table`)
- `--output, -o` — Write to file instead of STDOUT

**Examples:**
```bash
# Fetch and display as table
vkit fetch --grant gr_abc123xyz

# Fetch as JSON
vkit fetch --grant gr_abc123xyz --format json

# Fetch and save to CSV
vkit fetch --grant gr_abc123xyz --format csv --output results.csv

# Pipe to jq for processing
vkit fetch --grant gr_abc123xyz --format json | jq '.data[] | select(.revenue > 1000)'
```

---

### Agent Tokens

Agents authenticate with agent tokens instead of a user login. Issuing and revoking
them is for admins.

#### `vkit agents tokens create`

```bash
vkit agents tokens create --name <NAME> [OPTIONS]
```

**Options:**
- `--name` — Human-readable name (required), e.g. `support-assistant`
- `--expires-in` — Lifetime, e.g. `1h`, `24h`, `30d`
- `--tenant` — The tenant this token acts for on tenant-scoped datasets (vkit 1.2.0)
- `--clearance` — Clearance policies see: `low`, `medium` or `high` (vkit 1.2.0)
- `--region` — Region policies see: `EU`, `US`, `CA` or `APAC` (vkit 1.2.0)

The secret is shown once. Tenant, clearance and region are fixed when the token is
issued; the agent can't change them. For an assistant that serves several customers,
issue a short-lived token per customer session:

```bash
vkit agents tokens create --name support-assistant --tenant org_42 --expires-in 1h
```

Every query that token makes on a dataset marked with `tenant_column` is confined to
`org_42`'s rows. See [Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets).

#### `vkit agents tokens list` and `revoke`

```bash
vkit agents tokens list
vkit agents tokens revoke --token <ID_OR_PREFIX>
```

---

### Datasource Management (Admin Only)

#### `vkit datasource add`

Register a new datasource.

```bash
vkit datasource add --id <ID> --engine <ENGINE> [OPTIONS]
```

**Options:**
- `--id` — Unique datasource identifier (required)
- `--engine` — Database engine: `postgres`, `mysql`, `snowflake`, `bigquery` (required)
- `--username` — Database username
- `--password` — Database password
- `--config` — JSON configuration (host, port, database, etc.)
- `--credential-backend` — Secret backend: `vaultkit`, `vault`, `aws-secrets`

**Examples:**

**PostgreSQL (Built-in credential storage):**
```bash
vkit datasource add \
  --id production_pg \
  --engine postgres \
  --username vaultkit_ro \
  --password $DB_PASSWORD \
  --config '{
    "host": "db.production.internal",
    "port": 5432,
    "database": "analytics",
    "ssl_mode": "require",
    "connect_timeout": 10
  }'
```

**MySQL:**
```bash
vkit datasource add \
  --id production_mysql \
  --engine mysql \
  --username app_reader \
  --password $MYSQL_PASSWORD \
  --config '{
    "host": "mysql.production.internal",
    "port": 3306,
    "database": "ecommerce",
    "ssl_mode": "REQUIRED"
  }'
```

#### `vkit datasource list`

List all registered datasources.

```bash
vkit datasource list [--format FORMAT]
```

**Examples:**
```bash
vkit datasource list
vkit datasource list --format json
```

#### `vkit datasource get`

Get details for a specific datasource.

```bash
vkit datasource get <DATASOURCE_ID>
```

**Example:**
```bash
vkit datasource get production_pg
```

#### `vkit datasource test`

Test connectivity to a datasource.

```bash
vkit datasource test <DATASOURCE_ID>
```

**Example:**
```bash
vkit datasource test production_pg

# Output:
# Testing connection to production_pg...
# ✓ Connection successful
# Engine: PostgreSQL 15.3
# Latency: 45ms
```

#### `vkit datasource remove`

Remove a datasource (admin only).

```bash
vkit datasource remove <DATASOURCE_ID> [--force]
```

**Options:**
- `--force` — Skip confirmation prompt

**Example:**
```bash
vkit datasource remove old_staging_db --force
```

---

### Schema Discovery Commands

#### `vkit scan`

Scan a datasource and detect schema drift.

```bash
vkit scan <DATASOURCE_ID> [--mode MODE]
```

**Options:**
- `--mode` — Scan mode: `diff_only`, `apply` (default: `diff_only`)

**Modes:**

| Mode | Description |
|------|-------------|
| `diff_only` | Show changes without applying (safe, read-only) |
| `apply` | Update baseline registry with discovered schema |

**Examples:**

**Discover Changes:**
```bash
vkit scan production_pg

# Output:
# Scanning datasource: production_pg
# 
# Schema Drift Detected:
# 
# + Dataset: customers
#     + Field: phone_number (varchar) [PII] ⚠️  NEW - requires policy
#     ~ Field: email (text → varchar) [PII] ✓ Already masked
# 
# + Dataset: user_sessions
#     + Field: ip_address (inet) [PII] ⚠️  NEW - requires policy
#     + Field: user_agent (text) [INTERNAL] ⚠️  NEW - requires policy
# 
# Summary: 2 new datasets, 3 new fields, 1 modified field
```

**Apply Changes:**
```bash
vkit scan production_pg --mode apply

# Output:
# ✓ Baseline registry updated
# Next steps:
#   1. Review new fields: customers.phone_number, user_sessions.ip_address
#   2. Update policies in Git: config/policies/
#   3. Build new bundle: vkit policy bundle
#   4. Deploy bundle: vkit policy deploy
```

**Labels and suggestions:**

The scan labels each column with a sensitivity (`high`, `medium`, `low` or
`unknown`) and a category (`pii`, `financial`, `credential`, `internal` or
`unknown`), from its name, its type and its table. `unknown` means it couldn't place
a column that could hold anything, such as a `payload` jsonb or `notes` text column:
review it and give it a real label. The scan reads names and types only, never data.

A rescan keeps labels you've reviewed (from a deployed bundle or a manual edit). Where
the classifier now disagrees, `vkit scan` (vkit 1.2.0) lists a suggestion instead of
changing the label:

```
  api_keys                 ✓
    ? key_value  low/internal → high/credential (suggested)

  ? = classifier suggestion, not applied. To accept one, edit
    datasets/registry.yaml and redeploy the bundle.
```

`unknown` labels, and labels that only came from an earlier scan, are replaced by
the current classification.

**Scan All Datasources:**
```bash
for ds in $(vkit datasource list --format json | jq -r '.[].id'); do
  echo "Scanning $ds..."
  vkit scan $ds
done
```

---

### Policy Management Commands

#### `vkit policy bundle`

Compile policies and registry into a deployable bundle.

```bash
vkit policy bundle [OPTIONS]
```

**Options:**
- `--policies_dir` — Policies directory (default: `config/policies`)
- `--registry_dir` — Directory holding `registry.yaml` (default: `datasets`)
- `--out` — Output bundle file (default: `dist/policy_bundle.json`)
- `--org` — Organization (defaults to the logged-in user's)
- `--version` — Bundle version (defaults to the git commit, `git rev-parse HEAD`)

**Example:**
```bash
vkit policy bundle
vkit policy bundle --out dist/policy_bundle.json --org acme
```

**Checks:** the bundle is refused, with every problem listed, when a policy can never
match, or a dataset would leak across tenants (vkit 1.2.0):

- a key the engine ignores, such as `match.fields.category` (use `contains`);
- a tag or sensitivity no field has and the classifier can't produce;
- a dataset that isn't in `datasets/registry.yaml`;
- an invalid environment, region or clearance value, or `time` under top-level
  `context` (it works under `match.context`);
- a mask policy with no fields to mask;
- a `tenant_column` that isn't one of its dataset's fields, or a dataset that has
  another dataset's tenant column but isn't marked (`tenant_column: false` declares a
  table every tenant may read).

If the problems are in policies from a pack, `vkit policy pack upgrade` replaces them.

#### `vkit policy validate`

Validate a policy bundle without deploying.

```bash
vkit policy validate --bundle <BUNDLE_FILE>
```

**Example:**
```bash
vkit policy validate --bundle dist/policy_bundle.json

# Output:
# Validating policy bundle...
# ✓ Schema validation passed
# ✓ Policy syntax valid
# ✓ No circular dependencies
# ✓ All datasources referenced
# Bundle is valid and ready to deploy
```

#### `vkit policy deploy`

Deploy a policy bundle to the control plane.

```bash
vkit policy deploy --bundle <BUNDLE_FILE> [OPTIONS]
```

**Options:**
- `--bundle` — Path to bundle file (required)
- `--org` — Organization identifier (optional, defaults to logged-in user's org)
- `--activate` — Immediately activate bundle
- `--dry-run` — Validate deployment without activating

**Examples:**
```bash
# Deploy and activate (using logged-in user's org)
vkit policy deploy \
  --bundle dist/policy_bundle.json \
  --activate

# Deploy with explicit org
vkit policy deploy \
  --bundle dist/policy_bundle.json \
  --org acme \
  --activate

# Dry run (test deployment)
vkit policy deploy \
  --bundle dist/policy_bundle.json \
  --dry-run
```

#### `vkit policy list`

List active policy bundles.

```bash
vkit policy list
```

#### `vkit policy show`

Show details of a specific policy.

```bash
vkit policy show <POLICY_ID>
```

---

## Policy Pack Commands

### Initialize With Packs
```bash
vkit init --with starter
```

If no packs specified:
```
ℹ️  No packs specified. Installing 'starter' for secure defaults.
```

### Pack Commands

**List Packs**
```bash
vkit policy pack list
```

Output:
```
✓ starter v2.0.0
⚠ ai_safety (installed v1.1.0, available v2.0.0)
  financial_compliance (not installed)
```

**Install Pack**
```bash
vkit policy pack add starter
```

**Remove Pack**
```bash
vkit policy pack remove starter
```

**Upgrade Pack**
```bash
vkit policy pack upgrade starter
```

Upgrade all:
```bash
vkit policy pack upgrade
```

### Upgrade Safety

**`--dry-run`**

Preview upgrade:
```bash
vkit policy pack upgrade starter --dry-run
```

Shows what would change without modifying files.

**`--force`**

Overwrite existing pack-installed files:
```bash
vkit policy pack upgrade starter --force
```

⚠ Will overwrite pack-managed policy files.

### How Packs Install Files

Installed files are written to:
```
config/policies/
```

Using deterministic names:
```
starter__01__deny_credentials.yaml
starter__02__mask_pii_by_default.yaml
```

This ensures:

* File ownership tracking
* Safe removal
* Deterministic upgrades

### Pack State File

VaultKit tracks installed packs in:
```
.vkit/packs.yaml
```

Example:
```yaml
format_version: v1
installed_packs:
  starter:
    version: "2.0.0"
    layer: 10
    installed_at: "2026-02-15T15:00:00Z"
    pack_checksum: "abc123..."
    files:
      - path: config/policies/starter__02__mask_pii_by_default.yaml
        policy_id: mask_pii_by_default
```

This enables:

* Drift detection
* Safe removal
* Controlled upgrade
* Bundle reproducibility

### Recommended Workflow
```bash
vkit init --with starter
vkit policy pack add ai_safety
vkit scan production_db --mode apply
vkit policy bundle
vkit policy deploy
```

---

### Policy Lifecycle Commands

#### `vkit policy deploy`

Upload a policy bundle to the control plane.
```bash
vkit policy deploy --bundle dist/policy_bundle.json
```

This creates a new version in `uploaded` state.

**Options:**
- `--bundle` - Path to bundle file (required)
- `--org` - Organization identifier (optional)

**Example:**
```bash
vkit policy deploy --bundle dist/policy_bundle.json --org acme
```

---

#### `vkit policy activate`

Activate a specific bundle version.
```bash
vkit policy activate --bundle-version v12
```

**Only one bundle may be active at a time.**

Activation automatically marks the previously active bundle as `superseded`.

**Options:**
- `--bundle-version` - Version to activate (required)
- `--org` - Organization identifier (optional)
- `--force` - Skip confirmation prompt

**Example:**
```bash
vkit policy activate --bundle-version v12 --force
```

---

#### `vkit policy rollback`

Activate a previous version.
```bash
vkit policy rollback --to v11
```

Rollback is equivalent to activating an earlier version.

**Options:**
- `--to` - Target version (required)
- `--reason` - Rollback reason (required)
- `--org` - Organization identifier (optional)

**Example:**
```bash
vkit policy rollback --to v11 --reason "v12 had masking issues"
```

---

#### `vkit policy revoke`

Revoke a deployed policy bundle.
```bash
vkit policy revoke \
  --bundle-version v12 \
  --reason "Incorrect masking rule exposed sensitive data"
```

**Revocation:**

- Permanently blocks the bundle
- Invalidates associated grants
- Emits an audit event

**Revoked bundles cannot be reactivated.**

**Options:**
- `--bundle-version` - Version to revoke (required)
- `--reason` - Revocation reason (required)
- `--org` - Organization identifier (optional)
- `--force` - Skip confirmation prompt

**Example:**
```bash
vkit policy revoke \
  --bundle-version v12 \
  --reason "Critical security flaw in masking logic" \
  --force
```

**⚠️ Warning:** This action is irreversible. Use with caution.
---

### Audit Commands

#### `vkit audit query`

Search audit logs.

```bash
vkit audit query [OPTIONS]
```

**Options:**
- `--user` — Filter by user email
- `--action` — Filter by action: `granted`, `denied`, `pending`
- `--dataset` — Filter by dataset name
- `--days` — Look back N days (default: 7)
- `--limit` — Number of results (default: 100)
- `--format` — Output format: `json`, `table`, `csv`

**Examples:**
```bash
# All queries by a user
vkit audit query --user analyst@company.com --days 30

# Denied access attempts
vkit audit query --action denied --days 7

# Access to specific dataset
vkit audit query --dataset customers --action granted

# Export to CSV
vkit audit query --user analyst@company.com --format csv --output audit_report.csv
```

#### `vkit audit export`

Export audit logs for compliance reporting.

```bash
vkit audit export --start-date <DATE> --end-date <DATE> [OPTIONS]
```

**Options:**
- `--start-date` — Start date (YYYY-MM-DD)
- `--end-date` — End date (YYYY-MM-DD)
- `--format` — Output format: `json`, `csv` (default: `csv`)
- `--output` — Output file path

**Example:**
```bash
vkit audit export \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --format csv \
  --output compliance_reports/january_2024.csv
```

---

## Advanced Usage

### Scripting with vkit

**Batch Request Processing:**

```bash
#!/bin/bash
# process_requests.sh

REQUESTS=(
  '{"source_table":"customers","columns":["id","email"],"limit":100}'
  '{"source_table":"orders","columns":["order_id","total"],"limit":100}'
  '{"source_table":"products","columns":["product_id","name"],"limit":100}'
)

for aql in "${REQUESTS[@]}"; do
  echo "Processing: $aql"
  vkit request --aql "$aql" --format json | jq -r '.grant_id' >> grants.txt
done

echo "Generated grants:"
cat grants.txt
```

**Automated Approvals (for testing):**

```bash
#!/bin/bash
# auto_approve_pending.sh

PENDING=$(vkit approval:list --state pending --format json)

echo "$PENDING" | jq -r '.[].id' | while read -r request_id; do
  echo "Approving request: $request_id"
  vkit approval:approve "$request_id" --ttl 3600 --notes "Auto-approved for testing"
done
```

### Integration with CI/CD

**GitHub Actions Example:**

```yaml
name: Deploy VaultKit Policies

on:
  push:
    branches: [main]
    paths:
      - 'config/policies/**'
      - 'config/registry.yaml'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install vkit CLI
        run: gem install vaultkitcli
      
      - name: Authenticate
        run: |
          vkit login \
            --endpoint ${{ secrets.VKIT_API_URL }} \
            --email ${{ secrets.VKIT_BOT_EMAIL }} \
            --password ${{ secrets.VKIT_BOT_PASSWORD }}
      
      - name: Build Policy Bundle
        run: |
          vkit policy bundle \
            --policies_dir config/policies \
            --registry_dir config \
            --out policy_bundle.json \
            --org ${{ secrets.ORG_ID }}
      
      - name: Validate Bundle
        run: vkit policy validate --bundle policy_bundle.json
      
      - name: Deploy Bundle
        run: |
          vkit policy deploy \
            --bundle policy_bundle.json \
            --org ${{ secrets.ORG_ID }} \
            --activate
```

### Using jq for Advanced Filtering

**Extract specific fields:**
```bash
vkit requests list --format json | jq '.[] | select(.state == "approved") | {id, dataset, requester}'
```

**Count requests by state:**
```bash
vkit requests list --format json | jq 'group_by(.state) | map({state: .[0].state, count: length})'
```

**Find high-value transactions:**
```bash
vkit fetch --grant gr_abc123 --format json | jq '.data[] | select(.amount > 10000)'
```

---

## Troubleshooting

### Common Issues

#### Authentication Failed

**Problem:**
```
Error: Authentication failed (401 Unauthorized)
```

**Solutions:**
```bash
# 1. Check endpoint
echo $VKIT_API_URL

# 2. Re-authenticate
vkit logout
vkit login --endpoint http://localhost:3000 --email you@company.com

# 3. Verify credentials
vkit whoami
```

#### Connection Refused

**Problem:**
```
Error: Connection refused - connect(2) for "localhost" port 3000
```

**Solutions:**
```bash
# 1. Check if control plane is running
curl http://localhost:3000/health

# 2. Verify Docker containers
cd infra
docker compose ps

# 3. Start services
docker compose up
```

#### Grant Expired

**Problem:**
```
Error: Grant has expired
Grant ID: gr_abc123xyz
Expired at: 2024-01-15 12:00:00 UTC
```

**Solutions:**
```bash
# 1. Request new grant
vkit request --aql '{"source_table":"customers",...}'

# 2. Request longer TTL (if approval required)
vkit request --aql '{...}' --ttl 7200
```

#### Policy Denial

**Problem:**
```
❌ Request denied
Reason: Cross-region PII access forbidden
Policy: gdpr_protection
```

**Solutions:**
1. Review policy restrictions
2. Request approval if available
3. Modify query to exclude restricted fields
4. Contact admin to update policies

### Debug Mode

Enable verbose logging:

```bash
# Set debug environment variable
export VKIT_DEBUG=true

# Run command
vkit request --aql '{...}'

# Output includes:
# - HTTP request/response details
# - Policy evaluation trace
# - Timing information
```

### Health Check

Verify CLI and control plane connectivity:

```bash
# Check CLI version
vkit --version

# Check control plane health
curl $VKIT_API_URL/health

# Test authentication
vkit whoami
```

---

## Contributing

We welcome contributions to the VaultKit CLI!

### Development Setup

```bash
# Clone repository
git clone https://github.com/yourorg/vaultkit-cli.git
cd vaultkit-cli

# Install dependencies
bundle install

# Run tests
bundle exec rspec

# Run CLI locally
bundle exec bin/vkit --help
```

### Running Tests

```bash
# Unit tests
bundle exec rspec spec/unit

# Integration tests (requires running control plane)
bundle exec rspec spec/integration

# All tests
bundle exec rspec
```

### Code Style

```bash
# Run RuboCop
bundle exec rubocop

# Auto-fix issues
bundle exec rubocop -a
```

---

## Additional Resources

- **Main Repository**: [github.com/ndbaba1/vaultkitcli.git](https://github.com/ndbaba1/vaultkitcli.git)
- **Documentation**: [docs.vaultkit.io](https://docs.vaultkit.io)
- **AQL Specification**: [docs.vaultkit.io/aql](https://docs.vaultkit.io/aql)
- **Policy Reference**: [docs.vaultkit.io/policies](https://docs.vaultkit.io/policies)
- **API Documentation**: [docs.vaultkit.io/api](https://docs.vaultkit.io/api)

---

## License

VaultKit CLI is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

---

**Built with ❤️ by the VaultKit team**