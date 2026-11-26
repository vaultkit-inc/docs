# Set Up VaultKit with a Coding Agent

A coding agent (Claude Code, Cursor or another) can set VaultKit up for you. It installs `vkit`, connects your database, installs and deploys policies, and configures VaultKit's MCP server so it can query your data through VaultKit. It never sees your database password or the agent token.

Paste this into your agent, in the project you want it to query from:

```text
Set up VaultKit for this project by following https://docs.vaultkit.io/setup.md
```

[setup.md](https://docs.vaultkit.io/setup.md) is written for the agent: a checklist, the exact commands, what their `--format json` output looks like, and what to do for each error. It's served as plain markdown so an agent can read it as it is.

---

## What you need

- A running VaultKit control plane, and its URL. See the [Deployment Guide](deployment.md).
- An admin account in your organization's VaultKit console.
- A PostgreSQL database, reachable from VaultKit, and ideally a read-only role for VaultKit to connect as. The agent can give you the SQL to create one.
- Ruby 3.0 or later for `vkit` (2.0.0 or later), and Python 3.10 or later for the MCP server. The agent checks both and asks before installing anything.

## What you'll be asked to do

The agent does the setup. A few steps are yours, and it stops and asks at each one:

1. **Approve a setup session.** The agent runs `vkit login --setup` and shows you a code and a link. Open the link signed in as an admin, check that the code matches, and approve it.
   - This gives the agent a token that lasts 30 minutes and can only set VaultKit up: add datasources, scan schemas and deploy policies. It can't query data, approve requests or create tokens.
   - Its calls show in Activity as "agent setup on behalf of" you.
   - If your organization is already set up, the page has a box: "Allow this session to replace existing policies and datasources". Leave it unticked unless you want the agent to change what's deployed.
2. **Give the database details**: host, port, database name, the read-only username, region and environment. None of these are secret.
3. **Type the database password yourself.** The agent writes out a `vkit datasource add … --password-prompt` command for you to run in your own terminal. The password goes from your keyboard to VaultKit.
4. **Choose policy packs.** `starter` is always installed. The agent explains `ai_safety` and `financial_compliance`, and you decide.
5. **Review the classifications.** The agent shows you a table of columns that VaultKit couldn't classify, or that hold personal, financial or credential data. It changes only the labels you ask it to. Columns nobody has reviewed stay masked for agents until someone does.
6. **Create the agent token** on the console's Agents page: click **Create agent token**, give it the name the agent suggests, and pick an expiry. Set a tenant under Advanced if the dialog asks for one. The dialog shows the token once, with three `export` lines (`VAULTKIT_TOKEN`, `VAULTKIT_BASE_URL`, `VAULTKIT_ORG`) to put in your shell profile or secret manager.
   - Never put it in a `.env` file in the repository, never paste it into the agent's chat, and never put it in `VKIT_TOKEN`: the `vkit` CLI reads that, so `vkit` itself would act as the agent.
   - On an older control plane without that button, you create it with `vkit agents tokens create` in your own terminal instead, sign out with `vkit logout`, and approve a second setup session. The agent walks you through it.
7. **Restart the agent** from a new terminal, so it picks up the variables and the MCP server, and paste the prompt again. It carries on where it stopped.

## When it's done

Setup is done when `vkit doctor` reports no failures. Two things are normal to see:

- **A warning about the datasource role.** VaultKit can't yet check which database role a datasource connects as ([vaultkit#125](https://github.com/vaultkit-inc/vaultkit/issues/125)).
- **Approval groups with no members.** Requests that need approval from those groups come to org admins until you add people to the groups in the console (Team).

The agent finishes by telling you what it set up and running one safe query, such as a count grouped by a column that isn't sensitive.

You end up with:

- a policy project folder (`vaultkit-policies` by default) in git, with the reviewed `datasets/registry.yaml`;
- an active policy bundle whose version is that folder's commit;
- an MCP config file for your agent client (`.mcp.json` for Claude Code, `.cursor/mcp.json` for Cursor). It refers to the three environment variables and holds no secrets.

## If something goes wrong

- **The code expired.** Approval codes last 10 minutes. The agent asks for a new one.
- **The setup session expired part way.** Setup sessions last 30 minutes. The agent starts a new one, and every step is safe to rerun.
- **`setup_scope_forbidden`.** Either the step needs an admin signed in normally, or your organization was already set up and the session wasn't allowed to replace it. The agent tells you which one, and what you can do.
- **The agent was restarted or lost its place.** Paste the prompt again. It runs `vkit doctor` first and continues from the first step that isn't done.

See also: [CLI reference](cli.md), [MCP Usage Guide](mcp_usage.md), [Schema Discovery](schema_discovery.md), [Policy Packs](pack_system.md).
