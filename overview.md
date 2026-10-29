# VaultKit Overview

**Applies to:** VaultKit 0.9.0-alpha, vkit 1.2.0, Python SDK 0.3.0

VaultKit lets AI agents and people query production data without holding database
credentials. Every request is checked against policies you keep in git: the answer is
allowed, masked, sent to a person for approval, or denied. Allowed requests get a
short-lived grant for that one query, and everything is recorded.

VaultKit is self-hosted: the control plane, the query runtime and your data stay in
your infrastructure.

---

## How it works

```
Agent or person ──► Control plane ──────────► funl ──────────► PostgreSQL
 (vkit, SDK, MCP)   policies, grants,         translates AQL     your database
                    approvals, audit          to SQL, masks,
                                              runs the query
```

| Component | What it does |
| --- | --- |
| **Control plane** (Rails) | Signs people and agents in, evaluates policies, issues grants, runs approvals, stores encrypted datasource credentials, records the audit log. |
| **funl** (Go) | The query runtime. Translates [AQL](aql_specification.md) into parameterised SQL, applies masking, and runs the query. It only accepts requests signed by the control plane, and isn't exposed outside the deployment. |
| **Console** (React) | Web UI for approvals, the team and sign-in settings. |
| **vkit** (CLI) | Requests, approvals, datasources, schema scans, and building and deploying policy bundles. |
| **Python SDK and MCP server** | Lets agents query through VaultKit, including from MCP hosts such as Claude Code, Claude Desktop and Cursor. |

A request goes through these steps:

1. **Who's asking.** The requester is identified by their login or agent token. Their
   role, clearance, region and tenant come from their account or token, never from the
   request.
2. **What it reads.** The control plane works out every table and field the query
   touches, applies the row limit (100 by default, 10,000 at most), and confines
   tenant-scoped tables to the requester's tenant.
3. **What policy says.** The active policy bundle decides, field by field and table by
   table. The strongest action wins: deny, then require approval, then mask, then
   allow.
4. **A grant.** If the request is allowed, the control plane issues a grant for exactly
   that query, signed and time-limited. If it needs approval, it waits for the right
   people to sign off.
5. **The rows.** funl runs the query with the masking applied and returns the rows. The
   same question asked again while the grant is valid reuses it.

---

## What you can do today

**Policies in git**
- **Policy files:** you write policies as YAML, review them like code, and deploy them
  as a bundle versioned by the git commit. See the [Policy Guide](policy_guide.md).
- **What a policy matches:** a dataset, fields (by name, category tag or sensitivity),
  the requester's role, clearance and region, the datasource's environment and region,
  and time windows.
- **Ready-made packs:** [policy packs](pack_system.md) give you secure defaults to
  start from.
- **Checked before deploy:** `vkit policy bundle` refuses policies that can never match.

**Masking**
- **Methods:** `full` (`*****`), `partial` (first three characters), and `hash`
  (SHA-256).
- **Combining:** every matching mask policy applies, and the strictest method wins.
- **After approval:** masks stay; an approval doesn't unmask a field.
- **No leaking through queries:** queries can't aggregate, group or sort by masked
  values in ways that would reveal them.

**Approvals**
- **Who signs off:** policies name approval groups, and people in those groups sign off
  in the console or the CLI.
- **Several groups:** when policies name more than one group, each must sign off, each
  by a different person.
- **Never the requester:** nobody can approve their own request.
- **Bounded grants:** an approver can shorten a grant, never extend it.

See [Approvals and grants](approvals_and_grants.md).

**Grants**
- **Scope:** one exact query, for one requester, under one policy bundle, for a limited
  time.
- **Reuse:** asking the same question again within the grant's lifetime reuses it.
- **Ending one:** grants expire. Admins can revoke one at any time, and deploying or
  revoking a bundle ends the grants issued under it.

**Schema discovery**
- **Scans:** `vkit scan` labels every column with a sensitivity and a category from its
  name, type and table, without reading data.
- **Review:** columns it can't place are marked `unknown` for review. You review the
  labels in git, and rescans keep your decisions.

See [Schema discovery](schema_discovery.md).

**Multi-tenant SaaS data**
- Mark shared tables with a `tenant_column`, and issue each agent session a token for
  one tenant. Every query on those tables is then confined to that tenant's rows, and
  anyone without a tenant is denied.

See [Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets).

**Agents**
- Agents use their own tokens and the intent API, through the
  [Python SDK](first_agent_query.md) or the [MCP server](mcp_usage.md). They can
  discover the datasets they're allowed to see, query them with filters and
  aggregates, and wait for approvals.

**Organizations and sign-in**
- **Sign-in options:** email and password, or SSO with one OpenID Connect identity
  provider per deployment.
- **Team:** admins manage the team, approval groups and each person's clearance and
  region in the console.

**Audit**
- **What's recorded:** each request is written to the audit log, with the requester,
  their attributes and tenant, the query, the decision, and the grant and fetch events.

---

## Not yet

- **Datasources:** PostgreSQL only. Hash masking needs the `pgcrypto` extension in the
  target database.
- **Audit log:** stored in the control plane's PostgreSQL database. There's no audit
  API or console view yet, and the log isn't tamper-evident.
- **Console:** has no screens for policies, datasources or the registry; use `vkit` for
  those.
- **Tenancy:** schema-per-tenant and database-per-tenant routing aren't supported; only
  shared tables.

---

## Security model

- **No standing credentials.** Database credentials stay in the control plane, encrypted
  with AES-256-GCM under your `VAULTKIT_ENCRYPTION_KEY`. People and agents never see
  them. Use a read-only database role.
- **Grants are signed.** The control plane signs each grant with your RSA key (RS256).
  funl verifies every call with the public key: each call carries a short-lived token,
  valid for at most 60 seconds and bound to that exact request body, so a captured call
  can't be replayed or altered.
- **Checked on every use.** Before a grant runs, the control plane checks it hasn't
  expired or been revoked, that the policy bundle hasn't changed, and that the
  requester's context still matches.
- **Requests can't claim attributes.** Role, clearance, region and tenant come from the
  account or token; environment and dataset region from the datasource.
- **funl isn't exposed.** Only the control plane reaches it, on the internal network.

---

## Example: an agent that never sees raw personal data

```yaml
# config/policies/mask_pii_for_agents.yaml
id: mask_pii_for_agents
description: Agents see personal data hashed in production.
match:
  fields:
    contains: [pii]
context:
  requester_role: agent
  environment: production
action:
  mask: true
  reason: Personal data is hashed for agents.
  ttl: 15m
masking:
  default_method: hash
```

The agent asks for customer emails through the MCP server and gets SHA-256 hashes. It
can still count and group by them, but never sees an address.

## Example: break-glass access to payments

```yaml
# config/policies/payments_need_review.yaml
id: payments_need_review
description: A payments reviewer approves access to payment records in production.
match:
  dataset: payments
  fields:
    any: [id, amount, status, failure_reason]
context:
  environment: production
action:
  require_approval: true
  approver_role: payments_reviewer
  reason: Payment records need sign-off from a payments reviewer.
  ttl: 1h
```

An engineer's request is queued. Someone in `payments_reviewer` approves it in the
console, and the engineer reruns the request for rows, valid for an hour, with any
masked fields still masked.

## Example: one assistant, many customers

An assistant serves each customer from shared tables. Mark the tables with
`tenant_column: organization_id`, and mint a one-hour token per customer session:

```bash
vkit agents tokens create --name support-assistant --tenant org_42 --expires-in 1h
```

Whatever the assistant asks, it only ever reads `org_42`'s rows.

---

## Next steps

- [Deploy VaultKit](deployment.md)
- [Run your first agent query](first_agent_query.md)
- [Write policies](policy_guide.md)
- [CLI reference](cli.md)
