# VaultKit Documentation

VaultKit lets AI agents and people query production data without holding database
credentials. Policies in git decide what each request may return: allowed, masked, sent
to a person for approval, or denied. Allowed requests get a short-lived grant for that
one query, and everything is recorded.

**Current versions:** VaultKit 0.9.0-alpha · vkit 1.2.0 · Python SDK 0.3.0

---

## Start here

- [Overview](overview.md): what VaultKit is, how a request flows, what's supported
  today
- [Deployment Guide](deployment.md): run VaultKit with Docker Compose
- [Upgrading VaultKit](deployment.md#updating-vaultkit): notes for each release
- [Upgrading to vkit 2.0.0](upgrading_vkit_2.md): what changed in the CLI, and what to
  do about it
- [Your First Agent Query](first_agent_query.md): from a running deployment to a
  policy-governed agent query
- [Set Up VaultKit with a Coding Agent](agent_setup.md): paste one line into Claude
  Code, Cursor or another agent, and it sets VaultKit up for you

## Concepts

- [Policy Guide](policy_guide.md): writing policies, requester attributes, which
  policy wins
- [Policy Packs](pack_system.md): ready-made, versioned policies
- [Approvals and Grants](approvals_and_grants.md): who signs off, how long access
  lasts, reuse and revocation
- [Schema Discovery](schema_discovery.md): the registry, scan labels, reviewing them
- [Setup Sessions](setup_sessions.md): a short-lived token that lets a coding agent set
  VaultKit up for you, and what you approve
- [Tenant-Scoped Datasets](policy_guide.md#tenant-scoped-datasets): confine agents to
  one customer's rows in shared tables
- [AQL Specification](aql_specification.md): the query format, including the intent
  format agents use

## Reference

- [CLI (`vkit`)](cli.md): every command and option
- [Agent Tokens](mcp_usage.md#creating-agent-tokens): create them in the console or
  with `vkit`, with a tenant, clearance and region
- [`vkit doctor`](cli.md#checking-a-setup-vkit-doctor): check a whole setup in one
  command
- [MCP Usage Guide](mcp_usage.md): connect Claude Code, Claude Desktop, Cursor or
  another MCP host

---

## Common tasks

**Request data and fetch it**

```bash
vkit request --aql '{"source_table": "orders", "columns": ["id", "status"], "limit": 10}'
vkit fetch --grant <grant-ref-from-the-output>
```

**Approve a request**

```bash
vkit approval:list
vkit approval:approve <id>
```

Or open **Approvals** in the console.

**Scan a database and deploy policies**

```bash
vkit scan prod_db --mode apply
vkit registry export
# review datasets/registry.yaml and your policies, then commit
vkit policy bundle
vkit policy deploy
```

---

## Supported today

| | |
| --- | --- |
| Datasources | PostgreSQL |
| Sign-in | Email and password; SSO with one OpenID Connect provider per deployment |
| Clients | `vkit` CLI, Python SDK, MCP server, web console |
| Masking | `full`, `partial`, `hash` |
| Audit | Stored in the control plane's database (no audit API yet) |

---

## Source

- CLI: [github.com/vaultkit-inc/vaultkitcli](https://github.com/vaultkit-inc/vaultkitcli)
- Python SDK and MCP server: [github.com/vaultkit-inc/vaultkit-sdk-python](https://github.com/vaultkit-inc/vaultkit-sdk-python)
- Deployment files: [github.com/vaultkit-inc/deploy](https://github.com/vaultkit-inc/deploy)
- This documentation: [github.com/vaultkit-inc/docs](https://github.com/vaultkit-inc/docs), licensed under the Apache License 2.0
