---
title: Your First Agent Query | docs
---

# Your First Agent Query

You've deployed VaultKit — now what? This page walks through everything between a running deployment and your first policy-governed agent query. Each step links to its full reference page; this page just gives you the order and the minimum commands to get there.

If you haven't deployed yet, start with the [Deployment Guide](deployment.md).

---

## 1. Authenticate the CLI

```bash
vkit login --endpoint https://your-vaultkit-url --email you@company.com
```

Verify it worked:

```bash
vkit whoami
```

Full reference: [Authentication](cli.md#sign-in)

---

## 2. Scaffold a project (optional fast path)

If you want a working policy pack immediately rather than writing one from scratch:

```bash
vkit init --dir . --with starter,ai_safety
```

This creates a local project directory with starter policy packs already installed. Skip to [Step 4](#4-write-or-install-a-policy-pack) if you use this.

---

## 3. Register a data source

Requires admin access.

```bash
vkit datasource add \
  --id users_db \
  --engine postgres \
  --username <db-user> \
  --password <db-password> \
  --config <connection-config> \
  --region US \
  --environment production
```

Confirm it's registered:

```bash
vkit datasource list
```

Then scan it so VaultKit knows its schema:

```bash
vkit scan users_db --mode apply
```

`--mode diff_only` shows what changed without applying it — useful when re-scanning an existing source. To capture the current schema as a versioned file:

```bash
vkit registry export
```

Full reference: [Schema Discovery & Policy Management](schema_discovery.md)

---

## 4. Write or install a policy pack

If you didn't use `vkit init --with` above, install a pack directly:

```bash
vkit policy pack add starter
```

See what's installed, or inspect a specific pack:

```bash
vkit policy pack list
vkit policy pack info starter
```

The packs send approvals to the groups `data_owner` (starter), `security`
(ai_safety) and `finance_manager` (financial_compliance). Add people to those groups
in the console (Team, Edit member); until then only admins can approve.

Writing your own policies instead of using a pack is covered in the [Policy Schema Reference](policy_guide.md).

---

## 5. Compile, validate, and deploy the policy bundle

The bundle version is your git commit (`git rev-parse HEAD`), so commit your policies and registry **before** you bundle. `vkit policy bundle` doesn't check for uncommitted changes: it builds whatever is on disk and labels it with the last commit.

```bash
git add config/policies datasets
git commit -m "Add starter policy pack"
```

Then compile, validate, and deploy:

```bash
vkit policy bundle
vkit policy validate
vkit policy deploy
```

`bundle` compiles your YAML policies (plus the datasource registry) into a single JSON bundle, fingerprinted to your current commit. `validate` checks it before anything goes live. `deploy` activates it against your VaultKit control plane — pass `--activate false` if you want to deploy without immediately making it the active version.

If you change a policy afterward, you'll need to commit again before re-running `bundle`/`deploy` — the new commit becomes the new fingerprint.

Full reference: [Policy Management](cli.md#policy-projects-and-bundles) · [Policy Pack Commands](cli.md#policy-packs)

---

## 6. Create an agent identity

Agents authenticate with their own tokens, separate from human users.

```bash
vkit agents tokens create --name billing-bot --expires-in 24h
```

This prints a token — save it, you'll pass it to the SDK next.

With vkit 1.2.0 you can also fix what policies see for the token: `--clearance`
(`low`, `medium`, `high`), `--region` (`EU`, `US`, `CA`, `APAC`), and `--tenant` for
an assistant that serves several customers from shared tables. A token with
`--tenant org_42` only ever reads `org_42`'s rows on datasets marked with
`tenant_column` (see [Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets)).
The agent can't change any of them.

List or revoke tokens with:

```bash
vkit agents tokens list
vkit agents tokens revoke --token <id-or-prefix>
```

---

## 7. Run your first query — Python SDK

```bash
pip install "vaultkit>=0.3.0"
```

**Direct query (no LLM):**

```python
from vaultkit import VaultKitClient

client = VaultKitClient(
    base_url="https://your-vaultkit-url",
    token="<agent-token-from-step-6>",
    org="<your-org-slug>",
)

result = client.execute(
    dataset="users",                  # a dataset in your registry, not the datasource id
    fields=["id", "email"],
    limit=10,
    purpose="Analyze user activity",  # recorded and shown to approvers
)

print(result.rows)
```

`VaultKitClient` takes all three settings as arguments; it doesn't read environment
variables. A request needs `fields`, `aggregates`, or both. Aggregates (`count`,
`sum`, `avg`, `min`, `max`) with `group_by` need SDK 0.3.0 or later.

**As LLM agent tools:** the [`agent_openai_demo.py`](https://github.com/vaultkit-inc/vaultkit-sdk-python/blob/main/examples/agent_openai_demo.py)
example has an agent discover datasets first, only query what discovery returned, and
wait for approvals.

---

## 8. Run your first query — MCP

Install the SDK with the `mcp` extra (Python 3.10+), in a virtual environment:

```bash
python3 -m venv vaultkit-env
source vaultkit-env/bin/activate
pip install "vaultkit[mcp]>=0.3.0"
```

The MCP server reads its settings from the environment:

```bash
export VAULTKIT_BASE_URL="https://your-vaultkit-url"
export VAULTKIT_TOKEN="<agent-token-from-step-6>"
export VAULTKIT_ORG="<your-org-slug>"
vaultkit-mcp
```

It waits silently for an MCP host (Claude Code, Claude Desktop, Cursor, MCP Inspector)
to connect; that isn't a hang. To connect a host, see the
[MCP Usage Guide](mcp_usage.md).

---

## 9. Handle an approval-required query

If a policy requires approval, the SDK raises `ApprovalRequiredError`:

```python
from vaultkit.errors.exceptions import ApprovalRequiredError

try:
    client.execute(dataset="payments", fields=["id", "amount", "status"],
                   purpose="Investigate failed refunds for INC-123")
except ApprovalRequiredError as e:
    print(f"Approval required. Request ID: {e.request_id}")
```

An approver in the right group signs off in the console under **Approvals**, or with
the CLI:

```bash
vkit approval:list --state pending
vkit approval:approve <id>
```

The `<id>` is the number `approval:list` shows. It isn't the agent's request ID, so
identify the request by its requester, dataset, fields and reason. You can also
watch for new requests with `vkit approvals:watch`.

Once it's approved, the agent runs **the same request again** and gets the rows: the
approval issued a grant for exactly that query. An agent that wants to wait can
check first with `client.poll_request(request_id=e.request_id)`, which reports
`pending`, `granted` or `denied`. With the MCP server, the agent uses the
`vaultkit_check_approval` tool. See [Approvals and grants](approvals_and_grants.md).

---

## 10. Fetch data directly via CLI (no SDK)

For your own requests, `vkit request` and `vkit fetch` work without any code:

```bash
vkit request --aql '{"source_table": "users", "columns": ["id"], "limit": 5}'
vkit fetch --grant <grant-ref>
```

A grant belongs to whoever requested it, so you can't fetch an agent's grant as a
user. Admins can revoke any grant early:

```bash
vkit grant:revoke --grant <grant-ref> --reason "testing complete"
```

---

## What's next

- [Approvals and grants](approvals_and_grants.md) — how sign-offs, grant lifetimes and reuse work
- [Schema discovery](schema_discovery.md) — review the labels your policies act on
- [Overview](overview.md) — examples beyond this walkthrough
- [Security model](overview.md#security-model) — what VaultKit checks, and where
