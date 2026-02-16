# VaultKit

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)]()
[![Version](https://img.shields.io/badge/version-0.1.0-orange.svg)]()
[![Go Version](https://img.shields.io/badge/go-1.21+-00ADD8.svg)](https://golang.org)
[![Ruby Version](https://img.shields.io/badge/ruby-3.1+-CC342D.svg)](https://www.ruby-lang.org)

> A secure, policy-driven control plane for unified data access across heterogeneous data sources.

**TL;DR**: VaultKit prevents credential sprawl and enforces data access policies across databases. Write queries in AQL (vendor-neutral JSON), policies decide who sees what, FUNL executes safely. Schema changes require explicit review—no silent data exposure.

---

## 🔐 Security Guarantees

VaultKit is built with zero-trust governance and tamper-proof enforcement at its core. It ensures that:

* **Grants carry cryptographically signed runtime constraints** — every token includes a checksum over the policy bundle and authorization context.
* **Policy bundles are pinned to grants** — if a bundle changes, previous grants are no longer valid.
* **Runtime constraints cannot be tampered with** — policy constraints in the grant token are authenticated via digest.
* **Revocation kill switches exist** for both individual grants and entire policy bundles, enabling emergency governance control.

These characteristics go beyond traditional RBAC/ABAC systems and enable compliance-grade data access control.

---

## 🔑 Cryptographically Signed Grants

VaultKit issues short-lived, cryptographically signed access tokens (JWT) that embed:

* **Policy bundle checksum** — Pins grant to exact policy version
* **Runtime constraints** — TTL, requester identity, datasource, allowed operations
* **Grant identifiers** — Unique ID for audit trails and revocation
* **Authorization context** — Role, clearance level, region, environment

### Dual-Layer Verification

Tokens are verified at both **control plane** (VaultKit) and **execution plane** (FUNL), ensuring:

✅ **No policy tampering** — Bundle checksum enforced  
✅ **No grant reuse across orgs or bundles** — Org ID and bundle checksum embedded  
✅ **Detection of forged or replayed tokens** — RSA signature + grant ID tracking  
✅ **Time-bound access** — TTL enforced at both verification points  

This enables **credential-less execution** — users and AI agents never handle database passwords, only time-bound, cryptographically-signed authorization tokens.

---

## 🎯 What is VaultKit?

VaultKit provides enterprise-grade governance and security for data access, enabling applications, engineers, and AI agents to query multiple data sources through a unified, policy-controlled interface.

VaultKit is a **control plane** that governs how data is accessed across your organization. It centralizes authentication, authorization, policy evaluation, and access control—ensuring that every data request is properly authenticated, authorized, and audited before execution.

### Core Responsibilities

- **Authentication & Authorization**: Identity verification and access control with RBAC/ABAC support
- **Policy Evaluation**: Fine-grained policies based on user attributes, data sensitivity, geographic regions, and time constraints
- **Data Masking**: Column-level masking rules (full, partial, hash) applied based on user clearance
- **Connection Management**: Centralized datasource configuration and credential management
- **Approval Workflows**: Require explicit approval for accessing sensitive datasets
- **Audit Logging**: Complete audit trail of every data access request
- **Metadata Catalog**: Automated dataset classification and schema discovery
- **Session Tokens**: Short-lived, cryptographically-signed tokens for Zero-Trust security
- **Schema Governance**: Git-backed policy bundles with drift detection for safe schema evolution

---

## ⚡ What is FUNL?

FUNL (Functional Universal Query Language) is the **data plane and execution engine** that powers VaultKit's query capabilities. While VaultKit decides *if* and *how* data can be accessed, FUNL handles the actual execution.

### Core Responsibilities

- **AQL Translation**: Converts VaultKit's Abstract Query Language (AQL) into native query languages
- **Multi-Engine Support**: Executes queries against PostgreSQL, MySQL, Snowflake, BigQuery, and more
- **SQL Masking Engine**: Applies column-level masking at the SQL execution layer
- **Query Execution**: Handles parameterized queries with injection prevention
- **Result Sanitization**: Returns masked and filtered results back to VaultKit
- **JWT Authentication**: Only accepts cryptographically-signed requests from VaultKit

FUNL is designed to be lightweight, stateless, and horizontally scalable—making it ideal for high-throughput data access scenarios.

---

## Architecture

![Architecture](images/architecture.png)

### Component Breakdown

#### VaultKit (Control Plane)

- **Request Orchestrator**: Routes and manages the lifecycle of data access requests
- **Policy Engine**: Evaluates ABAC/RBAC rules with support for sensitivity levels, regions, and time constraints
  - **Policy Priority System**: `deny` > `require_approval` > `mask` > `allow`
  - **Match Rules**: Dataset-based, field sensitivity, categories, specific field names
  - **Context Awareness**: User role, clearance level, region, environment, time windows
- **Authentication Service**: Handles user identity via JWT, OAuth, or SSO integration
- **Schema Registry**: Runtime database storing discovered schemas and serving as drift baseline
- **Active Policy Bundle**: Immutable, Git-backed artifact containing all enforcement rules
- **Connection Manager**: Manages datasource credentials (supports VaultKit store, HashiCorp Vault, AWS Secrets Manager)
- **Approval Workflow**: Implements multi-stage approval for sensitive data access
- **Audit Logger**: Records every request to pluggable sinks (SQLite, PostgreSQL, S3, etc.)

#### FUNL (Data Plane)
- **AQL Translator**: Parses AQL and generates engine-specific SQL with proper escaping
- **Query Executor**: Maintains connection pools and executes parameterized queries
- **SQL Masking Layer**: Injects masking functions into SELECT statements based on policy
- **Engine Plugins**: Extensible architecture for adding new database engines

#### Governance Layer
- **Git Repository**: Source of truth for all policies and dataset registries
- **Schema Scans**: Observational tooling that discovers database changes
- **CI/CD Pipeline**: Automated validation, bundling, and deployment of policy changes

---

## 📋 Schema Discovery & Policy Management

VaultKit's approach to schema evolution is designed for **security, auditability, and safe change management**.

### The Core Philosophy: Intent vs. Reality

VaultKit deliberately separates:

- **Policy Bundles** (Intent) — Git-backed, immutable definitions of *what should be allowed*
- **Schema Scans** (Reality) — Observational discovery of *what actually exists*

This separation prevents silent data exposure. When new tables or sensitive columns appear, they require **explicit policy review** before access is granted.

![Schema Discovery Flow](images/schema-flow.png)

### How It Works

**Policy Bundles** are versioned artifacts that contain:
- Dataset registry (expected schema)
- Datasource definitions
- Compiled policy rules
- Bundle metadata (version, checksum, source)

**Schema Scans** perform these steps:
1. Introspect datasources via FUNL
2. Discover tables and columns
3. Classify fields (PII, financial, internal)
4. Compute diff against runtime registry
5. Surface changes for review

**Key Safety Principle:**

> Scans inform. Bundles enforce. Humans decide.

Scans **never** automatically update active policies. This ensures:
- No silent expansion of policy scope
- No auto-granting access to new sensitive data
- Full Git-based audit trail
- Compliance with SOC2, ISO 27001, GDPR, HIPAA

### Quick Example: Handling Schema Drift

```bash
# 1. Discover schema changes (safe, read-only)
vkit scan production_db

# Output shows drift:
# + dataset: customers
#     + field: ssn (string) [PII] ⚠️  NEW - requires policy
#     ~ field: email (text → varchar) [PII] ✓ Already masked

# 2. Review and update baseline
vkit scan production_db --apply

# 3. Update policies in Git
vim policies/customer_data.yaml

# 4. Build and deploy new bundle
vkit policy bundle
vkit policy deploy
```

### CI/CD Integration (Recommended)

```yaml
name: Schema Drift Check
on: [schedule, push]
jobs:
  drift-detection:
    runs-on: ubuntu-latest
    steps:
      - name: Scan all datasources
        run: vkit scan --all --fail-on-drift
      
      - name: Create PR if drift detected
        if: failure()
        run: |
          vkit scan --all --diff > drift-report.md
          gh pr create --title "Schema Drift Detected" --body-file drift-report.md
```

📘 **[Full Bundling & Scan System Documentation →](./docs/BUNDLING_SCAN_SYSTEM.md)**

---

## 🔁 Policy Bundle Lifecycle

VaultKit treats policy bundles as immutable, versioned governance artifacts.

Every bundle uploaded to the control plane transitions through a defined lifecycle:
```
uploaded → active → superseded → revoked
```

### Bundle States

| State | Meaning |
|-------|---------|
| `uploaded` | Bundle stored but not yet enforcing |
| `active` | Currently enforcing for the organization |
| `superseded` | Previously active but replaced by a newer version |
| `revoked` | Explicitly marked unsafe and permanently blocked |

**Only one bundle can be active at a time per organization.**

### Lifecycle Actions

| Action | Description |
|--------|-------------|
| **Deploy** | Upload a new bundle version |
| **Activate** | Make a specific bundle version active |
| **Rollback** | Activate a previous bundle version |
| **Revoke** | Mark a bundle checksum permanently unsafe |

### Governance Guarantees

- ✅ Bundles are immutable after upload
- ✅ Activation is explicit and audited
- ✅ Revocation is permanent and checksum-based
- ✅ Full version history is preserved
- ✅ All lifecycle transitions generate audit events

---

## Policy Packs

VaultKit ships with curated **Policy Packs** — versioned, layered collections of production-ready governance rules.

Policy Packs allow organizations to bootstrap secure defaults without writing every policy from scratch.

### Why Policy Packs Exist

Writing secure governance policies from scratch is difficult and error-prone.

VaultKit Policy Packs provide:

* Secure defaults
* Versioned policy collections
* Upgradeable governance rules
* Domain-specific policy templates (AI safety, financial compliance, etc.)
* Dependency-aware layering

### Built-in Packs

| Pack | Purpose |
| --- | --- |
| `starter` | Secure default baseline (masking, cross-region protection, approval rules) |
| `ai_safety` | AI/LLM-safe data exposure controls |
| `financial_compliance` | Financial data approval + production restrictions |

### Pack Layering Model

Packs are layered to enforce deterministic policy priority.

| Layer | Purpose |
| --- | --- |
| `foundation` | Core safety rules |
| `domain` | Domain-specific governance |
| `custom` | Organization-defined policies |

Lower layers enforce baseline security. Higher layers refine behavior.

### Dependency System

Packs may depend on other packs.

**Example:**
```
financial_compliance
    ↳ depends on: starter
```

VaultKit enforces dependency installation order.

### Drift Detection

VaultKit tracks:

* Installed pack version
* Pack checksum
* Policy file ownership
* Policy priority bands

If the CLI ships a newer version of a pack, VaultKit detects:
```
⚠ starter (installed v1.0.0, available v1.1.0)
```

This enables safe upgrades without silent behavior changes.

### Installed Pack Metadata in Bundle

When compiling a policy bundle, VaultKit embeds:
```json
"installed_packs": [
  { "name": "starter", "version": "1.0.0" }
]
```

This ensures:

* Bundle reproducibility
* Compliance traceability
* Governance audits
* Safe revocation & rollback

### Pack Safety Guarantees

* Pack-installed files are namespaced.
* Non-pack files cannot be overwritten accidentally.
* Upgrades require explicit intent.
* Force flag required for destructive overwrite.
* Dry-run available for preview.

---

## 🚨 Revocation System (Kill Switch)

VaultKit supports precise and enterprise-grade revocation for misuse detection, governance incidents, or policy errors. The revocation system operates at multiple levels to provide fine-grained control over access termination.

### Revocation Levels

**Grant Revocation** — Invalidate a specific grant before its TTL expires.
* Use case: Individual user's token compromised or misused
* Scope: Single access grant
* Impact: Only affects the specific grant ID
* Recovery: User can request new grant under current policies

**Bundle Revocation** — Prevent any access derived from a compromised or incorrect policy bundle.
* Use case: Policy bundle contains security flaw or incorrect rules
* Scope: All grants issued under that bundle version
* Impact: Every grant referencing the bundle checksum is immediately invalid
* Recovery: Requires activating a different bundle version

### How Revocation Works

Revocations are stored and evaluated at runtime. The policy engine checks both grant-level and bundle-level revocation lists before authorizing any data access:

1. **Grant validation** — Check if the specific grant ID is revoked
2. **Bundle validation** — Check if the bundle checksum is revoked
3. **TTL validation** — Check if the grant has expired
4. **Context validation** — Verify authorization context matches runtime state

If any validation fails, access is immediately denied and an audit event is generated.

### CLI Commands
```bash
# Revoke a specific grant
vkit grant revoke --grant-id grant_abc123xyz \
  --reason "Suspicious access pattern detected"

# Revoke an entire policy bundle
vkit policy revoke --bundle-checksum sha256:abc123... \
  --reason "Critical policy error - emergency rollback required"

# List all active revocations
vkit revocation list --type grant
vkit revocation list --type bundle

# View revocation audit trail
vkit audit query --event-type revocation
```

### Compliance & Security Impact

This capability is crucial for meeting compliance standards:

* **SOC2** — Demonstrates immediate access termination capability
* **HIPAA** — Enables emergency PHI access revocation
* **ISO 27001** — Supports incident response procedures
* **Internal security policies** — Provides kill switch for security incidents

### Important Considerations

> **⚠️ Revocation vs. Rollback**
>
> Revoking a bundle does NOT automatically activate a previous bundle.
> This is intentional — activation must be an explicit administrative action
> to prevent unintended policy regressions.

**Best Practices:**

* Document revocation reasons in audit metadata
* Have rollback bundle ready before revoking
* Test bundle activation in staging first
* Monitor for denied access after revocation
* Communicate revocations to affected teams

---

## 🔐 Git-Reviewed Deployment Workflow

VaultKit enforces policy changes through a Git-backed governance model.

**Policy bundles should never be edited directly in production.**

### Recommended Workflow

1. Engineer modifies policy YAML files
2. Pull Request is opened
3. Code review is completed
4. CI validates bundle integrity
5. `vkit policy bundle` builds artifact
6. `vkit policy deploy` uploads new version
7. Admin explicitly activates the bundle

### Why This Matters

This model ensures:

- ✅ No direct production mutation
- ✅ Immutable governance artifacts
- ✅ Auditable lifecycle transitions
- ✅ Reproducible compliance state
- ✅ Clear separation between authors and activators

**VaultKit enforces the principle:**

> **Scans inform. Bundles enforce. Humans decide.**

---

## 📄 AQL to Native Query Translation

AQL (Access Query Language) is a structured JSON format that describes **what** to fetch, not **how**. This abstraction allows VaultKit to enforce policies consistently across different database engines.

### AQL Structure

AQL is a structured JSON specification with the following top-level fields:

```json
{
  "source_table": "table_name",
  "columns": ["field1", "field2"],
  "joins": [],
  "aggregates": [],
  "filters": [],
  "group_by": [],
  "having": [],
  "order_by": null,
  "limit": 0,
  "offset": 0
}
```

### Translation Examples

**Simple Query:**
```json
{
  "source_table": "customers",
  "columns": ["email", "country", "revenue"],
  "filters": [
    { "field": "country", "operator": "eq", "value": "US" },
    { "field": "revenue", "operator": "gt", "value": 10000 }
  ],
  "limit": 100
}
```

**FUNL Translation → PostgreSQL:**
```sql
SELECT email, country, revenue
FROM customers
WHERE country = $1 AND revenue > $2
LIMIT 100
```

**FUNL Translation → MySQL:**
```sql
SELECT `email`, `country`, `revenue`
FROM `customers`
WHERE `country` = ? AND `revenue` > ?
LIMIT 100
```

**Complex Query with JOINs and Aggregates:**
```json
{
  "source_table": "users",
  "joins": [
    {
      "type": "LEFT",
      "table": "orders",
      "left_field": "users.id",
      "right_field": "orders.user_id"
    }
  ],
  "columns": ["users.email", "users.username"],
  "aggregates": [
    {
      "func": "sum",
      "field": "orders.amount",
      "alias": "total_spent"
    }
  ],
  "group_by": ["users.email", "users.username"],
  "having": [
    {
      "operator": "gt",
      "field": "SUM(orders.amount)",
      "value": 500
    }
  ],
  "order_by": {
    "column": "total_spent",
    "direction": "DESC"
  },
  "limit": 10
}
```

**FUNL Translation → PostgreSQL:**
```sql
SELECT users.email, users.username, SUM(orders.amount) AS total_spent
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.email, users.username
HAVING SUM(orders.amount) > $1
ORDER BY total_spent DESC
LIMIT 10
```

### Advanced Features

**Complex Filtering with OR Logic:**
```json
{
  "source_table": "users",
  "columns": ["id", "email", "role"],
  "filters": [
    {
      "logic": "OR",
      "conditions": [
        { "field": "users.role", "operator": "eq", "value": "admin" },
        { "field": "users.role", "operator": "eq", "value": "manager" }
      ]
    },
    { "field": "orders.amount", "operator": "gt", "value": 100 }
  ]
}
```

**Translation:**
```sql
SELECT id, email, role
FROM users
WHERE (users.role = $1 OR users.role = $2) AND orders.amount > $3
```

**Supported Operators:**
- Comparison: `eq`, `neq`, `gt`, `lt`, `gte`, `lte`
- Pattern matching: `like`
- Set operations: `in`
- Null checks: `is_null`, `is_not_null`

**Supported Aggregations:**
- `sum`, `count`, `avg`, `min`, `max`

**Supported JOINs:**
- `INNER`, `LEFT`, `RIGHT`, `FULL`

### SQL-Level Masking

FUNL's **Masking Dialect System** applies transformations directly in SQL based on VaultKit's policy decisions:

| Masking Type | PostgreSQL | MySQL | Snowflake |
|-------------|------------|-------|-----------|
| **Full** | `'*****' AS email` | `'*****' AS email` | `'*****' AS email` |
| **Partial** | `CONCAT(LEFT(email, 3), '****')` | `CONCAT(LEFT(email, 3), '****')` | `CONCAT(LEFT(email, 3), '****')` |
| **Hash** | `ENCODE(SHA256(email::bytea), 'hex')` | `SHA2(email, 256)` | `SHA2(email, 256)` |

---

## ✨ Key Features

### VaultKit (Control Plane)

| Feature | Description |
|---------|-------------|
| **AQL Orchestration** | Vendor-neutral query language prevents SQL injection and enables policy enforcement |
| **Policy Engine** | Attribute-based access control with support for clearance levels, regions, sensitivity tags, and time windows |
| **Policy Priority System** | Hierarchical decision making: `deny` > `require_approval` > `mask` > `allow` |
| **Field-Level Policies** | Match rules based on dataset name, field sensitivity, categories (pii, financial, etc.), or specific field names |
| **Context-Aware Rules** | Policies evaluate requester role, clearance level, region, environment, and time constraints |
| **Approval Workflows** | Require manager/security approval for accessing PII, financial data, or production environments |
| **Zero-Trust Sessions** | Short-lived, cryptographically-signed tokens with automatic expiration |
| **Credential Abstraction** | Never expose database credentials to users—supports multiple secret backends |
| **Git-Backed Governance** | All policies and registries versioned in Git with full audit trail |
| **Schema Drift Detection** | Automated discovery of database changes with manual approval workflow |
| **Comprehensive Auditing** | Every query logged with user identity, timestamp, policy decisions, and results metadata |
| **CLI & SDK** | Rich command-line tools (`vkit`) and programmatic SDK for integration |

### FUNL (Data Plane)

| Feature | Description |
|---------|-------------|
| **Multi-Engine Translation** | Supports PostgreSQL, MySQL, Snowflake, BigQuery with identical AQL interface |
| **SQL-Level Masking** | Masking applied during query execution—no post-processing overhead |
| **Injection Prevention** | All queries use parameterized execution with proper type binding |
| **Raw SQL Support** | Supports raw SQL for schema introspection and admin operations (policy-controlled) |
| **Horizontal Scaling** | Stateless design enables running multiple FUNL instances behind a load balancer |
| **JWT Verification** | Only executes queries signed by VaultKit's private key |

---

## 🚀 Getting Started

### Prerequisites Checklist

Before installing VaultKit, ensure you have:

- [ ] **Ruby** 3.1+ installed (`ruby --version`)
- [ ] **Go** 1.21+ installed (`go version`)
- [ ] **OpenSSL** for key generation
- [ ] **Git** for policy management
- [ ] Access credentials for at least one supported datasource
- [ ] (Optional) **Docker** for containerized FUNL deployment

### 🚀 Quick Start (Local Development)

The fastest way to get VaultKit running locally using Docker Compose:

#### 1. Clone the repository

```bash
git clone git@github.com:ndbaba1/vaultkit.git
cd vaultkit
```

#### 2. Create secrets

```bash
cp .env.example infra/secrets/.env
```

Edit `infra/secrets/.env`:

```env
POSTGRES_USER=vaultkit
POSTGRES_PASSWORD=secret
POSTGRES_DB=vaultkit_development
DATABASE_URL=postgres://vaultkit:secret@postgres:5432/vaultkit_development
FUNL_URL=http://funl-runtime:8080
RAILS_ENV=development
```

#### 3. Add signing keys

Place your RSA key pair in the secrets directory:

```bash
# Generate keys if you don't have them
openssl genpkey -algorithm RSA -out infra/secrets/vkit_priv.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -pubout -in infra/secrets/vkit_priv.pem -out infra/secrets/vkit_pub.pem
```

These keys are used to sign and verify access grants.

#### 4. Start VaultKit

```bash
cd infra
docker compose up --build
```

**Services:**
- **Control Plane** → http://localhost:3000
- **FUNL Runtime** → http://localhost:8080
- **Postgres** → localhost:5432

🎉 **VaultKit is now running locally!** You can now configure datasources and begin querying data through the control plane.

---

### Quick Start (CLI-Based Setup - 5 Minutes)

This streamlined path gets you querying data quickly with the CLI. For production deployments, see [Detailed Setup](#detailed-setup) below.

#### 1. Generate Cryptographic Keys

```bash
# Generate RSA key pair for token signing
openssl genpkey -algorithm RSA -out vaultkit_private.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -pubout -in vaultkit_private.pem -out vaultkit_public.pem

# Set environment variables
export VKIT_PRIVATE_KEY="$(pwd)/vaultkit_private.pem"
export VKIT_PUBLIC_KEY="$(pwd)/vaultkit_public.pem"
export FUNL_PUBLIC_KEY="$(pwd)/vaultkit_public.pem"
```

⚠️ **Security Note**: Keep your private key secure. Add `*.pem` to your `.gitignore`.

#### 2. Start FUNL

```bash
# Using Docker (recommended)
export FUNL_URL="http://localhost:8080"
docker compose up -d

# Verify FUNL is running
curl http://localhost:8080/health
```

#### 3. Install VaultKit CLI

```bash
git clone https://github.com/yourorg/vaultkitcli.git
cd vaultkitcli
gem build vaultkitcli.gemspec
gem install ./vaultkitcli-0.1.0.gem

# Verify installation
vkit --version
```

#### 4. Initialize & Connect

```bash
# Authenticate
vkit login

# Register your first datasource
vkit datasource add \
  --id demo_db \
  --engine postgres \
  --username readonly_user \
  --password $DB_PASSWORD \
  --config '{
    "host": "localhost",
    "port": 5432,
    "database": "analytics"
  }'

# Discover schema
vkit scan demo_db --apply
```

#### 5. Query Data

```bash
# Submit AQL query
vkit request --datasource demo_db --aql '{
  "source_table": "users",
  "columns": ["id", "email", "created_at"],
  "limit": 10
}'

# Fetch results (use grant ID from previous command)
vkit fetch --grant grant_abc123xyz
```

🎉 **You're now querying data through VaultKit!** Continue to [Detailed Setup](#detailed-setup) for production configuration.

---

### Detailed Setup

#### Installation Options

**Option A: From Source (Recommended for Development)**

```bash
# Clone and install CLI
git clone https://github.com/yourorg/vaultkitcli.git
cd vaultkitcli
bundle install
gem build vaultkitcli.gemspec
gem install ./vaultkitcli-0.1.0.gem

# Clone and build FUNL
git clone https://github.com/yourorg/funl.git
cd funl
go build -o funl ./cmd/funl
./funl serve --port 8080
```

**Option B: Using Docker (Recommended for Production)**

```bash
# docker-compose.yml
version: '3.8'
services:
  funl:
    image: vaultkit/funl:0.1.0
    ports:
      - "8080:8080"
    environment:
      - FUNL_PUBLIC_KEY=/keys/vaultkit_public.pem
    volumes:
      - ./vaultkit_public.pem:/keys/vaultkit_public.pem:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 3s
      retries: 3
```

#### Environment Configuration

Create a configuration file for persistent settings:

```bash
# ~/.vkit/config.yaml
funl_url: "http://localhost:8080"
private_key_path: "/path/to/vaultkit_private.pem"
public_key_path: "/path/to/vaultkit_public.pem"
default_datasource: "production_db"
audit_log_path: "/var/log/vaultkit/audit.log"
```

Add environment variables to your shell profile:

```bash
# ~/.bashrc or ~/.zshrc
export VKIT_PRIVATE_KEY="/path/to/vaultkit_private.pem"
export VKIT_PUBLIC_KEY="/path/to/vaultkit_public.pem"
export FUNL_PUBLIC_KEY="/path/to/vaultkit_public.pem"
export FUNL_URL="http://localhost:8080"
export VKIT_CONFIG_PATH="$HOME/.vkit/config.yaml"
```

#### Datasource Configuration

VaultKit supports multiple credential storage backends:

**Built-in Storage (Development):**
```bash
vkit datasource add \
  --id staging_db \
  --engine postgres \
  --username app_user \
  --password $DB_PASSWORD \
  --config '{
    "host": "staging.db.internal",
    "port": 5432,
    "database": "analytics",
    "ssl_mode": "require",
    "connect_timeout": 10
  }'
```

**HashiCorp Vault (Production):**
```bash
vkit datasource add \
  --id production_db \
  --engine postgres \
  --credential-backend vault \
  --vault-path secret/data/databases/production \
  --config '{
    "host": "prod.db.internal",
    "port": 5432,
    "database": "analytics",
    "ssl_mode": "verify-full"
  }'
```

**AWS Secrets Manager:**
```bash
vkit datasource add \
  --id cloud_db \
  --engine postgres \
  --credential-backend aws-secrets \
  --secret-arn arn:aws:secretsmanager:us-east-1:123456789:secret:db-creds \
  --config '{
    "host": "rds.amazonaws.com",
    "port": 5432,
    "database": "production"
  }'
```

#### Policy Setup

Initialize your policy repository:

```bash
# Create policy repository structure
mkdir -p vaultkit-policies/{policies,registries,datasources}
cd vaultkit-policies

git init
git add .
git commit -m "Initialize VaultKit policies"

# Link VaultKit to your policy repo
vkit policy init --repo $(pwd)
```

Create your first policy (`policies/base_access.yaml`):

```yaml
id: analyst_basic_access
description: Basic read access for analysts with PII masking
priority: 100

match:
  datasets:
    - customers
    - orders
  
context:
  requester_role: analyst
  environment: production

rules:
  - field_category: pii
    action: mask
    mask_type: partial
    reason: "PII must be masked for analysts"
  
  - field_category: financial
    action: require_approval
    approver_role: finance_manager
    ttl: "2h"
    reason: "Financial data requires approval"
  
  - default:
      action: allow
      ttl: "8h"
```

Build and deploy the policy bundle:

```bash
# Validate policies
vkit policy validate

# Build bundle
vkit policy bundle --output bundle.tar.gz

# Deploy to active control plane
vkit policy deploy --bundle bundle.tar.gz
```

---

## Use Cases

### 1. **Secure AI Agent Access**
Enable LLM-powered agents to query production databases without exposing credentials. VaultKit enforces policies that restrict which tables and columns AI agents can access, with automatic masking of sensitive fields.

**Problem:**
- AI agents need data context for decision-making
- Direct database access creates security risks
- Credentials in AI systems can leak
- Uncontrolled queries can expose sensitive data

**VaultKit Solution:**

```yaml
# policies/ai_agent_restrictions.yaml
id: ai_agent_restrictions
match:
  fields:
    category: pii

context:
  requester_role: ai_agent
  environment: production

action:
  mask: true
  mask_type: hash
  reason: "PII must be hashed for AI agents"
  ttl: "15m"
```

**How it works:**
1. AI agents receive short-lived session tokens (15 minutes)
2. All PII fields automatically hashed before results return
3. Agents can analyze patterns without accessing raw sensitive data
4. Every query logged with AI agent identity

**CLI Usage:**
```bash
# AI agent requests data
vkit request \
  --datasource production_db \
  --role ai_agent \
  --aql '{
    "source_table": "customer_behavior",
    "columns": ["user_id", "email", "purchase_amount"],
    "filters": [{"field": "purchase_amount", "operator": "gt", "value": 1000}]
  }'

# Result: email field is hashed
# {
#   "user_id": 12345,
#   "email": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
#   "purchase_amount": 1250.00
# }
```

---

### 2. **Cross-Region Compliance (GDPR, CCPA, HIPAA)**
Automatically mask or deny access to sensitive fields based on user location and data residency requirements.

**Problem:**
- GDPR prohibits transferring EU citizen data outside EU
- CCPA requires disclosure of data access by California residents
- HIPAA restricts PHI access based on business need
- Manual compliance is error-prone and doesn't scale

**VaultKit Solution:**

```yaml
# policies/gdpr_protection.yaml
id: gdpr_cross_region_deny
match:
  dataset: eu_customers
  fields:
    category: pii

context:
  requester_region: US
  dataset_region: EU

action:
  deny: true
  reason: "Cross-region PII access forbidden due to GDPR Article 44"
  audit_flag: compliance_violation

---
id: ccpa_disclosure_logging
match:
  dataset: california_residents

context:
  requester_region: ANY

action:
  allow: true
  ttl: "1h"
  audit_metadata:
    ccpa_disclosure: true
    data_subject_rights: "User has right to know about this access"
```

**How it works:**
- US-based analysts querying EU customer data are automatically denied
- Policy engine evaluates both requester and dataset regions
- Audit log captures denied attempts with compliance flags
- CCPA-flagged queries generate disclosure reports

**Multi-Region Access Pattern:**
```bash
# US analyst attempts to query EU data
vkit request --datasource eu_customers_db --aql '{...}'

# Response:
# ❌ Access Denied
# Reason: Cross-region PII access forbidden due to GDPR Article 44
# Policy: gdpr_cross_region_deny
# Audit ID: audit_20240115_abc123
```

---

### 3. **Just-In-Time Access (Break-Glass)**
Developers can request temporary elevated access to production data with automatic approval workflows and audit trails.

**Problem:**
- Engineers need emergency production access for incident resolution
- Standing privileges violate principle of least privilege
- Manual approval processes slow incident response
- Audit trails are incomplete or manual

**VaultKit Solution:**

```yaml
# policies/financial_requires_approval.yaml
id: financial_break_glass
match:
  fields:
    category: financial

context:
  environment: production
  requester_role: engineer

action:
  require_approval: true
  approver_role: finance_manager
  approval_metadata_required:
    - incident_ticket
    - business_justification
  ttl: "1h"
  reason: "Financial data access requires approval"
  auto_revoke: true
```

**CLI Usage:**
```bash
# Engineer requests emergency access
vkit request \
  --datasource production_db \
  --approval-required \
  --reason "Investigating payment processing failure - Ticket INC-5432" \
  --metadata '{"incident_ticket": "INC-5432", "business_justification": "Payment processor reporting transaction mismatch"}' \
  --duration "2h" \
  --aql '{
    "source_table": "transactions",
    "columns": ["transaction_id", "amount", "status"],
    "filters": [{"field": "status", "operator": "eq", "value": "failed"}]
  }'

# Response:
# ⏳ Approval Required
# Request ID: req_20240115_xyz789
# Status: PENDING
# Approver: finance_manager
# Expires: 2024-01-15 18:30 UTC
```

**Approval Workflow:**
```bash
# Finance manager reviews request
vkit approval list --pending

# Approve with notes
vkit approval grant \
  --request-id req_20240115_xyz789 \
  --notes "Approved for incident resolution. Access limited to failed transactions only."

# Engineer notified and can fetch data
vkit fetch --grant grant_approved_abc123
```

**Audit Trail Output:**
```json
{
  "request_id": "req_20240115_xyz789",
  "requester": "engineer@company.com",
  "requester_role": "engineer",
  "requested_at": "2024-01-15T16:00:00Z",
  "approved_by": "finance.manager@company.com",
  "approved_at": "2024-01-15T16:05:32Z",
  "approval_reason": "Approved for incident resolution",
  "access_granted": "2024-01-15T16:05:32Z",
  "access_expires": "2024-01-15T18:05:32Z",
  "queries_executed": 3,
  "rows_accessed": 147,
  "incident_ticket": "INC-5432",
  "auto_revoked": true
}
```

---

### 4. **Time-Restricted Access Windows**
Enforce access controls based on business hours, maintenance windows, or compliance requirements.

**Problem:**
- Audit logs should only be accessible during business hours
- Production data access outside work hours indicates potential misuse
- Compliance requires time-based controls for certain datasets

**VaultKit Solution:**

```yaml
# policies/time_restricted_access.yaml
id: business_hours_only
match:
  dataset: audit_logs
  environment: production

context:
  time:
    start: "08:00"
    end: "18:00"
    timezone: "America/Toronto"
    days: ["monday", "tuesday", "wednesday", "thursday", "friday"]

action:
  allow: true
  reason: "Access allowed during business hours"
  ttl: "1h"

---
id: after_hours_deny
match:
  dataset: audit_logs
  environment: production

context:
  time:
    outside_window: true

action:
  deny: true
  reason: "Audit log access restricted to business hours only"
  alert_security_team: true
```

**How it works:**
- Queries submitted outside business hours are automatically denied
- Security team alerted to suspicious access attempts
- Time zones properly handled for global teams
- Weekend access automatically blocked

**CLI Usage:**
```bash
# Analyst attempts to query audit logs at 10 PM
vkit request --datasource production_db --aql '{
  "source_table": "audit_logs",
  "columns": ["user_id", "action", "timestamp"],
  "limit": 100
}'

# Response (if outside business hours):
# ❌ Access Denied
# Reason: Audit log access restricted to business hours only
# Policy: after_hours_deny
# Business hours: Mon-Fri 08:00-18:00 America/Toronto
# Current time: Friday 22:15 America/Toronto
```

---

### 5. **Multi-Tenant SaaS Data Isolation**
Ensure customers in a SaaS platform can only access their own data through automatic tenant filtering.

**Problem:**
- SaaS applications need to enforce tenant isolation
- Manual WHERE clauses for tenant filtering are error-prone
- One misconfigured query can leak data across tenants
- Compliance requires provable data isolation

**VaultKit Solution:**

```yaml
# policies/tenant_isolation.yaml
id: enforce_tenant_boundary
match:
  datasets:
    - customers
    - orders
    - invoices
  
context:
  requester_role: customer_user

action:
  allow: true
  inject_filter:
    field: tenant_id
    operator: eq
    value: "{{session.tenant_id}}"
  reason: "Automatic tenant isolation enforced"
  audit_metadata:
    data_isolation: true
```

**How it works:**
1. User authenticates and receives JWT with `tenant_id` claim
2. VaultKit automatically injects `WHERE tenant_id = 'user_tenant'` into every query
3. Users cannot override or remove tenant filter
4. All queries logged with tenant context for compliance

**Example:**
```bash
# Customer user submits query (tenant_id: acme_corp)
vkit request --datasource app_db --aql '{
  "source_table": "orders",
  "columns": ["order_id", "total", "created_at"],
  "filters": [{"field": "status", "operator": "eq", "value": "completed"}]
}'

# VaultKit automatically rewrites to:
# SELECT order_id, total, created_at 
# FROM orders 
# WHERE status = 'completed' AND tenant_id = 'acme_corp'
```

---

## Policy Examples

### Clearance-Based Masking

```yaml
# policies/clearance_levels.yaml
id: pii_clearance_masking
description: Mask PII based on user clearance level

match:
  fields:
    category: pii

context:
  environment: production

rules:
  # Level 0: No clearance - full masking
  - clearance_level: 0
    action: mask
    mask_type: full
    reason: "No clearance - PII fully masked"
  
  # Level 1: Basic clearance - partial masking
  - clearance_level: 1
    action: mask
    mask_type: partial
    reason: "Basic clearance - partial PII exposure"
  
  # Level 2: High clearance - hashed
  - clearance_level: 2
    action: mask
    mask_type: hash
    reason: "High clearance - deterministic hashing"
  
  # Level 3: Full clearance - no masking
  - clearance_level: 3
    action: allow
    reason: "Full clearance granted"
    ttl: "4h"
```

### Environment-Based Restrictions

```yaml
# policies/environment_controls.yaml
id: production_restrictions
description: Stricter controls for production environment

match:
  environment: production

rules:
  # PII always masked in production for non-privileged users
  - field_category: pii
    context:
      requester_role: [analyst, engineer, data_scientist]
    action: mask
    mask_type: partial
  
  # Financial data requires approval in production
  - field_category: financial
    context:
      requester_role: [analyst, engineer]
    action: require_approval
    approver_role: finance_manager
    ttl: "2h"
  
  # Admin users have full access
  - context:
      requester_role: admin
    action: allow
    ttl: "1h"
    audit_metadata:
      privileged_access: true
```

### Category-Based Policies

```yaml
# policies/data_categories.yaml
id: category_based_controls
description: Different rules for different data categories

# Healthcare data (HIPAA)
- match:
    fields:
      category: phi
  action:
    require_approval: true
    approver_role: [compliance_officer, medical_director]
    ttl: "30m"
    reason: "PHI access requires compliance approval"
    audit_flag: hipaa_access

# Payment card data (PCI-DSS)
- match:
    fields:
      category: payment_card
  action:
    deny: true
    reason: "Direct payment card data access forbidden"
    alternative: "Use tokenized_card_id instead"

# Internal-only data
- match:
    fields:
      category: internal
  context:
    requester_role: [employee, contractor]
  action:
    allow: true
    ttl: "8h"
```

---

## Audit Logging

VaultKit provides comprehensive audit logging for compliance and security monitoring.

### Audit Log Fields

Every query execution generates an audit record containing:

```json
{
  "audit_id": "audit_20240115_abc123",
  "timestamp": "2024-01-15T16:30:45Z",
  "requester": {
    "user_id": "user_12345",
    "email": "analyst@company.com",
    "role": "analyst",
    "clearance_level": 2,
    "region": "US",
    "ip_address": "192.168.1.100"
  },
  "request": {
    "datasource_id": "production_db",
    "dataset": "customers",
    "columns_requested": ["id", "email", "revenue"],
    "filters": [{"field": "revenue", "operator": "gt", "value": 10000}],
    "aql_query": {...}
  },
  "policy_evaluation": {
    "policies_applied": [
      {
        "policy_id": "analyst_basic_access",
        "action": "mask",
        "fields_affected": ["email"],
        "mask_type": "partial"
      }
    ],
    "approval_required": false,
    "access_granted": true
  },
  "execution": {
    "funl_request_id": "funl_req_xyz789",
    "execution_time_ms": 245,
    "rows_returned": 47,
    "bytes_transferred": 12840
  },
  "metadata": {
    "environment": "production",
    "session_id": "session_abc456",
    "grant_id": "grant_def789",
    "ttl_expires": "2024-01-16T00:30:45Z"
  }
}
```

### Audit Query Examples

```bash
# View all queries by a specific user
vkit audit query --user analyst@company.com --limit 100

# View denied access attempts
vkit audit query --action denied --days 7

# View queries requiring approval
vkit audit query --approval-required --status pending

# Export compliance report
vkit audit export \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --format csv \
  --output compliance_report.csv

# Search for specific dataset access
vkit audit query --dataset customers --environment production
```

### Audit Sinks

VaultKit supports multiple audit log destinations:

**SQLite (Development):**
```yaml
audit:
  sink: sqlite
  path: /var/log/vaultkit/audit.db
```

**PostgreSQL (Production):**
```yaml
audit:
  sink: postgres
  connection:
    host: audit-db.internal
    port: 5432
    database: vaultkit_audit
    username: audit_writer
    password_secret: vault:secret/audit/db
```

**S3 (Compliance Archive):**
```yaml
audit:
  sink: s3
  bucket: company-audit-logs
  prefix: vaultkit/
  region: us-east-1
  retention_days: 2555  # 7 years for compliance
```

**Multiple Sinks (Recommended):**
```yaml
audit:
  sinks:
    - type: postgres
      name: primary
      connection: {...}
    
    - type: s3
      name: archive
      bucket: audit-archive
      
    - type: webhook
      name: security_monitoring
      url: https://siem.company.com/events
      headers:
        Authorization: "Bearer ${SIEM_TOKEN}"
```

---

## CLI Reference

### Core Commands

```bash
# Authentication
vkit login                           # Interactive login
vkit login --token $JWT_TOKEN        # Token-based login
vkit logout                          # Clear session

# Datasource Management
vkit datasource add                  # Register new datasource
vkit datasource list                 # List all datasources
vkit datasource test --id <id>       # Test connection
vkit datasource remove --id <id>     # Remove datasource

# Schema Discovery
vkit scan <datasource_id>            # Discover schema changes
vkit scan --all                      # Scan all datasources
vkit scan <id> --apply               # Update baseline
vkit scan <id> --diff                # Show changes only

# Policy Management
vkit policy init --repo <path>       # Initialize policy repo
vkit policy validate                 # Validate policy syntax
vkit policy bundle                   # Build policy bundle
vkit policy deploy                   # Deploy bundle
vkit policy list                     # List active policies
vkit policy show --id <policy_id>    # Show policy details

# Data Access
vkit request                         # Submit AQL request
vkit fetch --grant <grant_id>        # Retrieve query results
vkit cancel --grant <grant_id>       # Cancel pending request

# Approvals
vkit approval list                   # List pending approvals
vkit approval grant --request <id>   # Approve request
vkit approval deny --request <id>    # Deny request

# Audit
vkit audit query                     # Query audit logs
vkit audit export                    # Export audit data

# Administration
vkit user list                       # List users
vkit user create                     # Create user
vkit user set-role                   # Update user role
vkit session list                    # List active sessions
vkit session revoke --id <session>   # Revoke session
```

### Request Command Examples

```bash
# Simple query
vkit request \
  --datasource production_db \
  --aql '{"source_table": "users", "columns": ["id", "email"], "limit": 10}'

# Query with filters
vkit request \
  --datasource production_db \
  --aql '{
    "source_table": "orders",
    "columns": ["order_id", "total"],
    "filters": [
      {"field": "status", "operator": "eq", "value": "completed"},
      {"field": "total", "operator": "gt", "value": 100}
    ]
  }'

# Query requiring approval
vkit request \
  --datasource production_db \
  --approval-required \
  --reason "Q4 financial analysis" \
  --metadata '{"project": "annual_review"}' \
  --aql '{...}'

# Query with custom TTL
vkit request \
  --datasource staging_db \
  --ttl "2h" \
  --aql '{...}'
```

---

## Security Best Practices

### Key Management

✅ **DO:**
- Store private keys in HSM or secure key management service
- Use separate key pairs for each environment (dev, staging, prod)
- Rotate keys regularly (recommended: quarterly)
- Use strong key lengths (minimum 2048-bit RSA)

❌ **DON'T:**
- Commit private keys to version control
- Share keys across environments
- Use the same keys for signing and encryption
- Store keys in application code

### Policy Design

✅ **DO:**
- Follow principle of least privilege
- Use deny-by-default policies
- Implement approval workflows for sensitive data
- Set appropriate TTLs (shorter for production)
- Use role-based access with clear hierarchies

❌ **DON'T:**
- Use overly permissive wildcard policies
- Skip approval workflows in production
- Set TTLs longer than necessary
- Hard-code user identities in policies

### Deployment

✅ **DO:**
- Use HTTPS/TLS for all communications
- Run FUNL in isolated network segments
- Enable audit logging to immutable storage
- Monitor for unusual access patterns
- Implement rate limiting
- Use secret managers for credentials

❌ **DON'T:**
- Expose FUNL directly to the internet
- Disable audit logging
- Use self-signed certificates in production
- Store credentials in environment variables
- Skip security reviews for policy changes

---

## Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

### Development Setup

```bash
# Clone repositories
git clone https://github.com/yourorg/vaultkit.git
git clone https://github.com/yourorg/funl.git

# VaultKit (Ruby/Rails)
cd vaultkit
bundle install
rails db:create db:migrate
rails server

# FUNL (Go)
cd funl
go mod download
go run cmd/funl/main.go serve
```

### Running Tests

```bash
# VaultKit tests
bundle exec rspec

# FUNL tests
go test ./...

# Integration tests
./scripts/integration_tests.sh
```

---

## Documentation

- [Architecture Deep Dive](./docs/ARCHITECTURE.md)
- [Bundling & Scan System](./docs/BUNDLING_SCAN_SYSTEM.md)
- [Policy Language Reference](./docs/POLICY_REFERENCE.md)
- [AQL Specification](./docs/AQL_SPEC.md)
- [API Documentation](./docs/API.md)
- [Deployment Guide](./docs/DEPLOYMENT.md)
- [Troubleshooting](./docs/TROUBLESHOOTING.md)

---

## Roadmap

### Q1 2025
- [ ] BigQuery and Snowflake native support
- [ ] Web UI for policy management
- [ ] Enhanced approval workflows with multi-stage approvals
- [ ] Real-time audit log streaming

### Q2 2025
- [ ] MongoDB and DynamoDB support
- [ ] Data anonymization beyond masking
- [ ] Advanced analytics on audit logs
- [ ] Kubernetes operator for deployment

### Q3 2025
- [ ] GraphQL query support
- [ ] Field-level encryption at rest
- [ ] ML-based anomaly detection
- [ ] Self-service data catalog

---

## License

VaultKit is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

---

## Support

- **Documentation**: https://docs.vaultkit.io
- **Issues**: https://github.com/yourorg/vaultkit/issues
- **Discussions**: https://github.com/yourorg/vaultkit/discussions
- **Email**: support@vaultkit.io
- **Slack**: [Join our community](https://vaultkit.slack.com)

---

## Acknowledgments

VaultKit is built with and inspired by:
- [Open Policy Agent](https://www.openpolicyagent.org/) - Policy engine inspiration
- [HashiCorp Vault](https://www.vaultproject.io/) - Secret management patterns
- [Trino](https://trino.io/) - Federated query concepts
- [Osquery](https://osquery.io/) - Security-first data access

---

**Built with ❤️ by the VaultKit team**