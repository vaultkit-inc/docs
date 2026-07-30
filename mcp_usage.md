# VaultKit MCP — Usage Guide

How to install, run, connect, and test the VaultKit MCP server. This is the
operational companion to `MCP_INTEGRATION.md` (which covers the code/design).

The MCP server exposes VaultKit's governed tools (`vaultkit_discover`,
`vaultkit_query`, `vaultkit_check_approval`) over the Model Context Protocol, so
any MCP host — Claude Desktop, Cursor, the MCP Inspector, custom agents — can
query governed data. It is a **client** of the control plane; it enforces
nothing itself.

---

## Prerequisites

1. **The control plane must be running and reachable.** The MCP server just
   forwards to it. Default `http://localhost:3000`. Verify:
   ```bash
   curl -i http://localhost:3000/up
   ```
   Any HTTP response = up. Connection refused = start it first.

2. **The `[mcp]` extra must be installed** into the right Python environment
   (see below). **Python 3.10+ is required** — this is declared in
   `pyproject.toml` as `requires-python = ">=3.10"`, so pip enforces it at
   install time: on 3.8/3.9 the install fails with a clear version error rather
   than breaking later.

3. **Three credentials**, which are how the server authenticates to the control
   plane. The token IS the principal:
   ```
   VAULTKIT_BASE_URL   e.g. http://localhost:3000
   VAULTKIT_TOKEN      the agent/principal token
   VAULTKIT_ORG        e.g. demolabs
   ```

---

## Install

There are two different install paths depending on who you are. Both need
**Python 3.10+** — the package declares `requires-python = ">=3.10"` in
`pyproject.toml`, so pip refuses to install on anything older (a clear error, not
a silent problem). Check your version first:

```bash
python3 --version        # must be 3.10 or newer
```

Both paths should use a **virtual environment** — it's what makes `pip` and the
`vaultkit-mcp` command resolve to the same, predictable place. Skipping the venv
is the #1 cause of "command not found," "installed but won't import," and the
`spawn ENOENT` error when a host tries to launch the server. If your system
`python3` is older than 3.10, create the venv with a newer interpreter
explicitly (e.g. `python3.11 -m venv ...`).

### A. End users (installing VaultKit to use it)

End users do **not** have the repo. They install the published package by name.
Always create and activate a venv first:

```bash
python3 -m venv vaultkit-env
source vaultkit-env/bin/activate          # Windows: vaultkit-env\Scripts\activate
pip install "vaultkit[mcp]"
```

> `pip install "vaultkit[mcp]"` only works **once VaultKit is published to
> PyPI**. Until then, use the Git install below.

**Before PyPI (design partners, private distribution) — install straight from
Git**, no publishing needed. This is the recommended path during the
design-partner phase:

```bash
python3 -m venv vaultkit-env
source vaultkit-env/bin/activate
pip install "vaultkit[mcp] @ git+https://github.com/vaultkit-inc/vaultkit-sdk-python.git"
```

(For a private repo the user needs access; use an SSH URL or a token-authed
HTTPS URL. Pin a tag/commit by appending `@v0.1.2` for reproducible installs.)

### B. Developers / contributors (working ON the SDK)

Only this path uses `-e` (editable install from a local checkout). It requires
the repo and a **modern pip** — the old macOS system pip (21.x) cannot do
editable installs from a `pyproject.toml`-only project, which is why the venv's
own pip is used:

```bash
git clone https://github.com/vaultkit-inc/vaultkit-sdk-python.git
cd vaultkit-sdk-python
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[mcp]"
```

### Verify (either path)

```bash
python -m pip --version                        # modern pip (23+), from the venv — NOT system 21.x
python -c "import vaultkit.mcp; print('ok')"    # package imports
vaultkit-mcp --help                             # console script is on PATH
```

### If `pip` misbehaves

Inside an activated venv, plain `pip` is correct and points at the venv's
Python. If you're **not** in a venv, or `pip`/`pip3` is missing or points at the
wrong Python (e.g. Xcode's system Python), invoke pip through the interpreter
instead — it always installs into the same Python that runs it:

```bash
python3 -m pip install "vaultkit[mcp]"     # macOS/Linux
python  -m pip install "vaultkit[mcp]"     # Windows (python3 often absent)
```

If `vaultkit-mcp` still isn't found after install, the env isn't active or it
landed elsewhere — use the absolute path to the binary
(`.../vaultkit-env/bin/vaultkit-mcp`, or `...\Scripts\vaultkit-mcp.exe` on
Windows) or run the module directly: `python -m vaultkit.mcp`. You'll need that
absolute path for the Claude Desktop config regardless.

---

## Run (stdio — the local/demo transport)

stdio is the default. The server is launched as a subprocess by an MCP host and
speaks JSON-RPC over stdin/stdout.

```bash
export VAULTKIT_BASE_URL="http://localhost:3000"
export VAULTKIT_TOKEN="<token>"
export VAULTKIT_ORG="demolabs"

vaultkit-mcp
```

Run directly like this, it starts and then **sits silently waiting for input**.
That is correct, not a hang — it's waiting for a client. Drive it with a host
(below), not by hand.

> `--transport http` exists but is not implemented yet (hosted path, deferred).
> Use stdio.

---

## Test it: MCP Inspector (CLI mode) — most reliable

CLI mode does one request and exits. No browser, no session state, no spinner.
Pass credentials with `-e` so it never depends on shell inheritance.

List tools (proves discovery reaches the control plane):

```bash
npx @modelcontextprotocol/inspector --cli vaultkit-mcp \
  -e VAULTKIT_BASE_URL="http://localhost:3000" \
  -e VAULTKIT_TOKEN="<token>" \
  -e VAULTKIT_ORG="demolabs" \
  --method tools/list
```

Expect three tools, and `vaultkit_query` should carry a populated
"VALID FIELDS BY DATASET" block. If fields are missing, the control plane was
unreachable at build time — reconnect once it's up.

Call a tool for real:

```bash
# discover — returns your actual datasets
npx @modelcontextprotocol/inspector --cli vaultkit-mcp \
  -e VAULTKIT_BASE_URL="http://localhost:3000" \
  -e VAULTKIT_TOKEN="<token>" \
  -e VAULTKIT_ORG="demolabs" \
  --method tools/call --tool-name vaultkit_discover
```

For `vaultkit_query`, pass arguments (syntax depends on your Inspector version;
`--tool-arg key=value` or a JSON args flag). The demo query is `customers`
selecting `email` and `ssn` with a `purpose` — that's where masking fires.

---

## Test it: MCP Inspector (UI mode)

Launch **writable** (no `--config`) so you can add credentials in the interface:

```bash
npx @modelcontextprotocol/inspector
```

Then: add/select the `vaultkit-mcp` server → expand it → add the three
`VAULTKIT_*` environment variables → toggle Connect → open the Tools tab.

> If you see a **"Read-only session"** banner, the list was launched with
> `--config` and you **cannot** add env vars in the UI — the server will spin on
> "Connecting…" forever because it starts without credentials. Relaunch with no
> `--config` (or `--catalog`) to get a writable, persistent catalog.

---

## Connect Claude Desktop

Settings → Developer → Edit Config, then add (use the **absolute** path — the
app doesn't run under your shell PATH):

```json
{
  "mcpServers": {
    "vaultkit": {
      "command": "/absolute/path/to/vaultkit-env/bin/vaultkit-mcp",
      "env": {
        "VAULTKIT_BASE_URL": "http://localhost:3000",
        "VAULTKIT_TOKEN": "<token>",
        "VAULTKIT_ORG": "demolabs"
      }
    }
  }
}
```

Fully quit and reopen Claude Desktop (it reads config only at launch). Confirm
the tools appear in the tools/connector menu, then ask in plain English:

- "What datasets can I access in VaultKit?" → discover
- "Show me 5 customers with their email and SSN." → query; sensitive fields come
  back masked — the demo money-shot
- A dataset/field you expect to be denied or approval-gated → shows the deny /
  pending-approval path

Claude Desktop caches the tool list on connect — after SDK or dataset changes,
fully restart it to force a fresh `list_tools`.

---

## Reading results

Every tool result is a structured status from the executor (it never raises):

| status | meaning |
|---|---|
| `ok` / `approved` | data returned (check `note` for masked fields) |
| `denied` | policy blocked it — deterministic, don't retry |
| `pending_approval` | needs human sign-off; poll `vaultkit_check_approval` with the `request_id` |
| `transport_error` | control plane unreachable — start it / check `VAULTKIT_BASE_URL` |
| `validation_error` | malformed query |

A `denied` is a successful governed outcome, not a failure.

---

## Troubleshooting (all seen in practice)

| Symptom | Cause | Fix |
|---|---|---|
| "Missing required VaultKit credentials" | env vars not reaching the process | set all three; in Inspector use `-e` (CLI) or a writable catalog (UI) |
| UI stuck on "Connecting…" | read-only session, no env vars in config | relaunch Inspector writable; add env; or just use CLI mode |
| `transport_error` / "Connection refused" | control plane down | start it on the port in `VAULTKIT_BASE_URL` |
| `spawn ENOENT` / command not found | wrong PATH / wrong env | absolute path to the binary, or `python -m vaultkit.mcp` |
| `pip install -e` fails, "setup.py not found" | old system pip (21.x); `-e` is dev-only | activate the venv and use its modern pip; end users use `pip install "vaultkit[mcp]"` (no `-e`) |
| Tools list but fields empty / dataset free-text | control plane unreachable at build time | ensure CP is up, reconnect/relist |
| A dataset or fields silently missing | transient fetch swallowed to empty (known issue) | reconnect; see hardening backlog §3/§5 |

The `-e`-in-CLI and writable-catalog-in-UI patterns exist specifically because
the Inspector does **not** reliably inherit your shell environment — set
credentials explicitly, always.

---

## Security notes

- **Never paste your token in logs/screenshots.** It rides in `-e` flags and
  server config, so it's easy to leak. Rotate immediately if exposed. Mask the
  middle when sharing output.
- The token is the principal; masking, denials, and audit are decided by the
  control plane, not here. Different tokens/principals may see different
  datasets and fields — the tool list is per-principal.
- stdio has no per-connection auth (identity rides on the token in env). The
  hosted HTTP path (OAuth per session) is not built yet.
