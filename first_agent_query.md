---
title: Your First Agent Query | docs
---

# Your First Agent Query

You've deployed VaultKit — now what? This page walks through everything between a running deployment and your first policy-governed agent query. Each step links to its full reference page; this page just gives you the order and the minimum commands to get there.

If you haven't deployed yet, start with the [Deployment Guide](./deployment.html).

---

## 1. Authenticate the CLI

```bash
vkit login --endpoint https://your-vaultkit-url --email you@company.com
```

Verify it worked:

```bash
vkit whoami
```

Full reference: [Authentication](./authentication.html)

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

Full reference: [Schema Discovery & Policy Management](./schema-discovery.html)

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

Writing your own policies instead of using a pack is covered in the [Policy Schema Reference](./policy-schema.html).

---

## 5. Compile, validate, and deploy the policy bundle

VaultKit fingerprints each policy bundle using your git `commit_sha`, so your project must be inside a git repo with changes committed **before** you deploy — an uncommitted or dirty working tree will fingerprint against the wrong commit (or fail outright).

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

Full reference: [Policy Management](./policy-management.html) · [Policy Pack Commands](./policy-pack-commands.html)

---

## 6. Create an agent identity

Agents authenticate with their own tokens, separate from human users.

```bash
vkit agents tokens create --name billing-bot --expires-in 24h --role agent
```

This prints a token — save it, you'll pass it to the SDK next. List or revoke tokens with:

```bash
vkit agents tokens list
vkit agents tokens revoke --token <id-or-prefix>
```

---

## 7. Run your first query — Python SDK

```bash
pip install vaultkit
```

**Direct query (no LLM):**

```python
from vaultkit import VaultKitClient

client = VaultKitClient(
    base_url="https://your-vaultkit-url",
    token="<agent-token-from-step-6>",
    org="<your-org>",
)

result = client.execute(
    dataset="users_db",
    fields=["id", "email"],
    limit=10,
    purpose="Analyze user activity",
)

print(result.rows)
```

Or set these as environment variables instead of passing them inline:

```bash
export VAULTKIT_URL=https://your-vaultkit-url
export VAULTKIT_TOKEN=<agent-token>
export VAULTKIT_ORG=<your-org>
```

**As LLM agent tools (OpenAI example):**

This is the realistic pattern for a production agent — it discovers datasets first, only queries what discovery returned, and polls for approval without blocking the whole process.

Full working example: [`agent_openai_demo.py`](https://github.com/vaultkit-inc/vaultkit-sdk-python/blob/main/examples/agent_openai_demo.py)

---

## 8. Run your first query — MCP

Install the SDK with the `mcp` extra (requires Python 3.10+) inside a virtual environment:

```bash
python3 -m venv vaultkit-env
source vaultkit-env/bin/activate
pip install "vaultkit[mcp] @ git+https://github.com/vaultkit-inc/vaultkit-sdk-python.git"
```

Set the same three credentials as before:

```bash
export VAULTKIT_BASE_URL="https://your-vaultkit-url"
export VAULTKIT_TOKEN="<agent-token-from-step-6>"
export VAULTKIT_ORG="<your-org>"
```

Start the MCP server:

```bash
vaultkit-mcp
```

It will sit silently waiting for input — that's correct, it's waiting for an MCP host (Claude Desktop, Cursor, MCP Inspector) to connect, not a hang.

For connecting a host, testing with MCP Inspector, and troubleshooting, see the full [MCP Usage Guide](./mcp-usage.html).

---

## 9. Handle an approval-required query

If a policy requires human approval, the SDK raises `ApprovalRequiredError`:

```python
from vaultkit.errors.exceptions import ApprovalRequiredError

try:
    client.execute(dataset="sensitive_data", purpose="Analysis")
except ApprovalRequiredError as e:
    print(f"Approval required. Request ID: {e.request_id}")
```

From another terminal (as the approver):

```bash
vkit approval:list --state pending
vkit approval:approve <id> --ttl 3600
```

Or watch pending approvals live:

```bash
vkit approvals:watch --interval 3
```

Once approved, resume from the agent side:

```python
result = client.poll_request(request_id="req_123")
```

If you're using the tool-based agent pattern from Step 7 instead of direct `client.execute()`, poll with the `vaultkit_check_approval` tool in a loop (with a timeout) rather than calling `poll_request` directly — this lets the agent keep responding instead of blocking. See the approval-wait loop in [`agent_openai_demo.py`](https://github.com/vaultkit-inc/vaultkit-sdk-python/blob/main/examples/agent_openai_demo.py) for the full pattern.

Full reference: [Data Requests](./data-requests.html)

---

## 10. Fetch data directly via CLI (no SDK)

Useful for testing a grant without writing code:

```bash
vkit fetch --grant <grant-id>
```

Revoke a grant early if needed:

```bash
vkit grant:revoke --grant <grant-id> --reason "testing complete"
```

---

## What's next

- [Audit Queries](./audit-queries.html) — search and export the audit log for everything you just did
- [Use Cases](./use-cases.html) — real-world patterns beyond this walkthrough
- [Security Best Practices](./security-best-practices.html) — key management before going to production
