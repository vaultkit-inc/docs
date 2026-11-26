---
title: Your First Agent Query | docs
---

# Your First Agent Query

You've deployed VaultKit — now what? This page walks through everything between a running deployment and your first policy-governed agent query. Each step links to its full reference page; this page just gives you the order and the minimum commands to get there.

If you haven't deployed yet, start with the [Deployment Guide](deployment.md).

**The recommended way:** let a coding agent do it. Paste one line into Claude Code,
Cursor or another agent, and it runs these steps for you, asking you only for what it
shouldn't handle: approving its setup session, typing the database password, reviewing
the classifications and creating its token. See
[Set Up VaultKit with a Coding Agent](agent_setup.md).

The steps below are the same setup by hand. They need vkit 2.0.0 or later (see
[Upgrading to vkit 2.0.0](upgrading_vkit_2.md)).

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

This creates a local project directory with starter policy packs already installed, and runs `git init` if the folder isn't in a git repository yet (`--no-git` skips it). Its `.gitignore` keeps `.vkit/packs.yaml` tracked, so commit that file along with your policies. Skip to [Step 4](#4-write-or-install-a-policy-pack) if you use this.

---

## 3. Register a data source

Requires admin access.

```bash
vkit datasource add \
  --id users_db \
  --engine postgres \
  --username <db-user> \
  --config <connection-config> \
  --region US \
  --environment production \
  --password-prompt
```

`--password-prompt` asks for the database password in your terminal, with echo off, so
it never ends up in your shell history. In a script, use `--password-env VAR_NAME`
instead. The old `--password` option still works, but it's deprecated.

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
in the console (Team, Edit member). Until a group has an active member, org admins
approve its requests, and `vkit policy deploy` warns about it (see
[Groups with no members](approvals_and_grants.md#groups-with-no-members)).

Writing your own policies instead of using a pack is covered in the [Policy Schema Reference](policy_guide.md).

---

## 5. Compile, validate, and deploy the policy bundle

The bundle version is your git commit (`git rev-parse HEAD`), so commit your policies, registry and `.vkit/packs.yaml` **before** you bundle. `vkit policy bundle` refuses to build while any of them has uncommitted changes (`bundle_not_committed`), and lists the files and the git commands to commit them.

```bash
git add config/policies datasets .vkit/packs.yaml
git commit -m "Add starter policy pack"
```

Then compile, validate, and deploy:

```bash
vkit policy bundle
vkit policy validate
vkit policy deploy
```

`bundle` compiles your YAML policies (plus the datasource registry) into a single JSON bundle, fingerprinted to your current commit. `validate` checks it before anything goes live. `deploy` activates it against your VaultKit control plane — pass `--no-activate` if you want to deploy without immediately making it the active version.

If you change a policy afterward, you'll need to commit again before re-running `bundle`/`deploy` — the new commit becomes the new fingerprint. Rebuilding and deploying a commit that's already active changes nothing (`already_deployed`).

Full reference: [Policy Management](cli.md#policy-projects-and-bundles) · [Policy Pack Commands](cli.md#policy-packs)

---

## 6. Create an agent identity

Agents authenticate with their own tokens, separate from human users. Only admins
create them.

**In the console:** open **Agents**, click **Create agent token**, give it a name
(such as `billing-bot`) and an expiry, and click **Create token**. The dialog shows
the token once, with three lines to set where the agent runs:

```bash
export VAULTKIT_TOKEN="<paste token>"
export VAULTKIT_BASE_URL="https://your-vaultkit-url"
export VAULTKIT_ORG="<your-org-slug>"
```

Copy the token, tick **I've saved the token**, and click **Done**. See
[Creating agent tokens](mcp_usage.md#creating-agent-tokens).

**Or with the CLI:**

```bash
vkit agents tokens create --name billing-bot --expires-in 24h --secret-file ~/billing-bot.token
```

`--secret-file` writes the token to a new file (mode 600) instead of printing it.
Without it, the token is printed once.

Put the token in your shell profile or a secret manager, not in a `.env` file in a
repository. Use `VAULTKIT_TOKEN`, not `VKIT_TOKEN`: `VKIT_TOKEN` overrides the `vkit`
CLI's own login, so every `vkit` command would run as the agent.

You can also fix what policies see for the token (in the console, under
**Advanced**): `--clearance`
(`low`, `medium`, `high`), `--region` (`EU`, `US`, `CA`, `APAC`), and `--tenant` for
an assistant that serves several customers from shared tables. A token with
`--tenant org_42` only ever reads `org_42`'s rows on datasets marked with
`tenant_column` (see [Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets)).
The agent can't change any of them.

List or revoke tokens with:

```bash
vkit agents tokens list
vkit agents tokens revoke --token <id-or-prefix> --reason "rotating keys"
```

---

## 7. Run your first query — Python SDK

```bash
pip install "vaultkit>=0.5.0"
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

Field names must be the dataset's own, exactly as the registry has them.
`client.schema("users")` lists them, with what a query for each would get (allowed,
masked, or needing approval and from whom). A field the dataset doesn't have is
refused with a `ValidationError` naming the valid fields.

**As LLM agent tools:** the [`agent_openai_demo.py`](https://github.com/vaultkit-inc/vaultkit-sdk-python/blob/main/examples/agent_openai_demo.py)
example has an agent discover datasets first, only query what discovery returned, and
wait for approvals.

---

## 8. Run your first query — MCP

Install the SDK with the `mcp` extra (Python 3.10+), in a virtual environment:

```bash
python3 -m venv vaultkit-env
source vaultkit-env/bin/activate
pip install "vaultkit[mcp]>=0.5.0"
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

To check the whole setup, run `vkit doctor` from your policy project with the same
three variables set. It checks the login, datasources, schema, active bundle,
approval groups, agent token and MCP server, and says how to fix what fails. See
[`vkit doctor`](cli.md#checking-a-setup-vkit-doctor).

The agent gets four tools. It calls `vaultkit_describe` on a dataset before querying
it, so it uses the real field names instead of guessing them.

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
    print(e.approval_progress)  # "Needs 2 approvals: finance_manager waiting, security waiting"
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
`pending`, `granted` or `denied`, and while pending, `approval_progress` ("Needs 2
approvals: finance_manager ✓ Ada, security waiting"). With the MCP server, the agent
uses the `vaultkit_check_approval` tool, which returns the same wording. See
[Approvals and grants](approvals_and_grants.md).

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
