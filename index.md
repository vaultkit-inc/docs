# VaultKit Documentation

> A secure, policy-driven control plane for unified data access across heterogeneous data sources.

VaultKit prevents credential sprawl and enforces data access policies across databases. Write queries in AQL (vendor-neutral JSON), policies decide who sees what, FUNL executes safely. Schema changes require explicit review—no silent data exposure.

---

## Documentation

### Getting Started
- [Main Overview](overview.md) - Complete introduction to VaultKit and FUNL
- [Quick Start Guide](overview.md#-getting-started) - Get up and running in 5 minutes
- [Installation](overview.md#installation-options) - Detailed installation instructions
- [Deployment Guide](deployment.md) - Production deployment instructions**

### Core Concepts
- [Architecture Overview](overview.md#-architecture) - How VaultKit and FUNL work together
- [AQL Syntax & Specification](aql_specification.md) - Abstract Query Language reference
- [Policy Schema Reference](policy_guide.md) - Policy configuration guide

### CLI Reference
- [VaultKit CLI (`vkit`)](cli.md) - Complete command-line reference
- [Authentication](cli.md#-authentication) - Login and session management
- [Data Requests](cli.md#data-request-commands) - Querying data with AQL
- [Policy Management](cli.md#policy-management-commands) - Deploy and manage policies
- [Audit Queries](cli.md#audit-commands) - Search and export audit logs

### Advanced Topics
- [Schema Discovery & Policy Management](overview.md#-schema-discovery--policy-management) - Handling schema drift
- [Security Best Practices](overview.md#-security-best-practices) - Key management and deployment
- [Use Cases](overview.md#-use-cases) - Real-world examples and patterns

---

## Quick Links

**For Users:**
- [Submit a data request](cli.md#vkit-request)
- [Fetch query results](cli.md#vkit-fetch)
- [Check request status](cli.md#vkit-requests-list)

**For Approvers:**
- [Review pending requests](cli.md#vkit-approvallist)
- [Approve/deny requests](cli.md#vkit-approvalapprove)

**For Administrators:**
- [Deploy VaultKit](deployment.md)
- [Register datasources](cli.md#vkit-datasource-add)
- [Scan for schema changes](cli.md#vkit-scan)
- [Deploy policy bundles](cli.md#vkit-policy-deploy)
- [Query audit logs](cli.md#vkit-audit-query)

---

## Common Workflows

### Request Data with Auto-Approval
```bash
# Submit AQL query
vkit request --aql '{
  "source_table": "customers",
  "columns": ["id", "email", "revenue"],
  "limit": 10
}'

# Fetch results
vkit fetch --grant gr_abc123xyz
```

### Request Sensitive Data (Requires Approval)
```bash
# Submit request
vkit request --aql '{
  "source_table": "financial_transactions",
  "columns": ["transaction_id", "amount"],
  "filters": [{"field": "amount", "operator": "gt", "value": 10000}]
}'

# Check status
vkit requests list --state pending

# After approval
vkit fetch --grant gr_approved_xyz
```

### Discover Schema Changes
```bash
# Scan database
vkit scan production_db

# Apply changes to baseline
vkit scan production_db --apply

# Update policies and deploy
vkit policy bundle
vkit policy deploy --bundle dist/policy_bundle.json --activate
```

---

## Key Features

| Feature | Description |
|---------|-------------|
| **Zero-Trust Access** | Short-lived, cryptographically-signed tokens |
| **Policy-Driven** | ABAC/RBAC with field-level controls |
| **Multi-Engine** | PostgreSQL, MySQL, Snowflake, BigQuery |
| **SQL-Level Masking** | Masking applied during query execution |
| **Schema Governance** | Git-backed policies with drift detection |
| **Audit Trail** | Complete logging of every data access |
| **Approval Workflows** | Multi-stage approvals for sensitive data |
| **Vendor-Neutral** | AQL abstracts database-specific syntax |

---

## Support

- **GitHub**: [github.com/ndbaba1/vaultkitcli.git](https://github.com/ndbaba1/vaultkitcli.git)
- **Issues**: Report bugs and request features
- **Email**: Contact the VaultKit Engineering Team

---

## License

VaultKit is licensed under the Apache License 2.0.

---

**Built with ❤️ by the VaultKit Engineering Team**