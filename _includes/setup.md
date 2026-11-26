# Set up VaultKit

You are setting up VaultKit for the person you're working with. Follow every step in order.

## Goal

VaultKit lets AI agents query a database through policies instead of holding database credentials. When you're done, the person has:

- the `vkit` CLI (2.0.0 or later), on a setup session with their VaultKit control plane, never on the person's own login;
- one datasource: their database, connected through a read-only role;
- a policy project folder in git: the starter policy pack, any extra packs they chose, and a reviewed registry of their tables and columns;
- that policy bundle deployed and active;
- an agent token that only they have seen, in their environment as `VAULTKIT_TOKEN`, with `VAULTKIT_BASE_URL` and `VAULTKIT_ORG`;
- the VaultKit MCP server configured for this agent client, so you can query through VaultKit.

Setup is done when `vkit doctor --format json` reports no failures.

## Rules

- Never ask for, read, print or store the database password or any token. The person types the password into their own terminal, and only they ever see the agent token. Don't write a token or password into any file, and don't echo, `cat` or print environment variables that hold one.
- Every step marked **ASK** is the person's decision or action. Explain it, then wait for their answer. Never answer it yourself, and never pick a default for them unless the step gives one.
- Don't edit policy YAML (`config/policies/`) at all, and edit `datasets/registry.yaml` only as step 7 says. Never pass `--allow-dirty` or `--version`.
- Run every `vkit` command you run yourself with `--format json`. It prints one JSON object on stdout: `status` (`ok`, `error` or `needs_input`), `result`, `next`, and, when the status isn't `ok`, `error` with a `code`, a `message` and a `fix`. Exit code 0 means `ok`, 1 means `error`, 3 means `needs_input` (the person has to do something). Ignore stderr; it's for humans.
- Your shell tool times out, so never run a command that waits without a limit. The only waiting command here is `vkit login --setup --wait`, and it always gets `--timeout 90`.
- Each step lists the error codes it can return and what to do about each. These codes can come from any step:
  - `control_plane_unreachable`: check the control plane URL with the person (step 2), then rerun.
  - `not_logged_in`: go back to step 2.
  - `setup_session_expired`, `setup_session_revoked`, `setup_session_invalid`: the setup session has ended (it lasts 30 minutes). Do step 2 again, then rerun the command that failed.
  - `setup_approver_not_admin`: stop. Tell the person the admin who approved the session is no longer an admin, so an admin has to approve a new one (step 2).
  - `setup_scope_forbidden`: see "Setup sessions" in step 2.
  - `invalid_arguments`: you got a flag wrong. Fix it (`vkit help <command>` lists them) and rerun.
- On any error code not listed in this file, stop. Show the person `error.message` and `error.fix`, and ask what to do.
- Only run `vkit` on a setup session, never on the person's own login. vkit keeps one saved login per computer, shared by your commands and the person's terminal. Before your first `vkit` command after step 9, and whenever you resume, run `vkit whoami --format json`. If `result.setup_session` isn't there, run nothing else in vkit. If it shows the person signed in (`result.email` and no `setup_session`), ask them to run `vkit logout` in their terminal. Then do step 2.
- If setup was started before (for example, the person restarted you in step 10), first check `vkit whoami` as the rule above says, then run `vkit doctor --format json` in the policy project folder. Tell the person what it shows, then continue from the first step that isn't done. Every step is safe to rerun.

## Checklist

1. Check Ruby 3.0+ and install or upgrade the vkit CLI to 2.0.0+.
2. Start a setup session. **ASK** the person to approve it in the console.
3. **ASK** for the database details, then **ASK** the person to add the datasource with its password.
4. **ASK** which policy packs to add to starter.
5. Create the policy project folder with `vkit init`, and commit it.
6. Scan the database, apply the scan, and export the registry.
7. **ASK** the person to review the classifications, and apply their changes.
8. Commit, then bundle, validate and deploy the policies.
9. **ASK** the person to create the agent token in the console and set the three environment variables.
10. Configure the VaultKit MCP server for this client. **ASK** the person to restart you.
11. Run `vkit doctor --format json` and fix what it reports.
12. Tell the person what was set up, and run one safe query.

## Steps

### 1. Install the vkit CLI

```bash
ruby -e 'print RUBY_VERSION'
vkit --version
```

- Ruby must be 3.0 or later. If Ruby is missing or older (macOS ships 2.6), **ASK** the person before installing it, and use the tool they prefer (for example `brew install ruby`, rbenv or asdf, or their system's package manager).
- `vkit --version` prints `vkit <version>`. It must be 2.0.0 or later.
  - Not installed: `gem install vaultkit`
  - Older than 2.0.0: `gem update vaultkit`

  Then run `vkit --version` again.
- If `gem install` fails with a permissions error, don't use `sudo` unless the person says so. Instead, use `gem install --user-install vaultkit` and add the directory it names to `PATH`.
- If `vkit` isn't found after installing, `gem environment` shows the "EXECUTABLE DIRECTORY" it was installed to. Use that path, or add it to `PATH`.

### 2. Start a setup session

A setup session gives you a token for 30 minutes that can only do setup: add datasources, scan, export the registry and deploy policies. It acts for the admin who approves it, and the console's Activity page records it as "agent setup on behalf of <them>".

```bash
vkit login --setup --format json
```

- It exits 3 with `status: needs_input` and `error.code: setup_approval_required`. `result` has `user_code` (like `SSKG-0363`), `verification_url_complete` (the approval link), `expires_at` and `endpoint`. vkit also tries to open the link in a browser on this computer.
- `error.code: endpoint_required`: vkit doesn't know the control plane. **ASK** the person for their VaultKit control plane URL (the address `vkit` and the API use, which an admin can tell them). Then rerun with `--endpoint <url>`.
- Rerunning this command while the code is still valid returns the same code. It never creates a second request.

**ASK**: tell the person:

> Open <verification_url_complete> and sign in to the VaultKit console as an organization admin. Check that the code on the page is <user_code>, then click Approve. Please do it before <expires_at, in their local time>. Only approve it if you just asked me to set up VaultKit.
>
> If your organization already has policies or datasources, the page shows a box: "Allow this session to replace existing policies and datasources". Leave it unticked unless you want this session to change what's already deployed. Rerunning setup steps that are already done works without it.

Then wait for the approval:

```bash
vkit login --setup --wait --timeout 90 --format json
```

- `status: ok` with `result.setup_session: true`: approved. `result.organization` is the organization's slug and `result.expires_at` is when the session ends.
  - If `result.replaced_login` is present, vkit was signed in as the person before. Tell them this replaced that sign-in.
- `status: needs_input` with `setup_approval_required`: the person hasn't approved yet. Remind them, and run the same `--wait` command again.
- `setup_session_denied`: ask the person whether they meant to deny it. If they want to try again, start this step over.
- `setup_session_expired`: the code expired before anyone approved it (codes last 10 minutes). Start this step over to get a new code.
- `setup_session_not_started`: run `vkit login --setup --format json` first.
- `setup_poll_failed`: rerun the `--wait` command. If it fails again, start this step over.

**Setup sessions.** If a later step returns `setup_session_expired`, repeat this step and then rerun the step that failed.

If a step returns `setup_scope_forbidden`, read `error.message`:

- It starts "Setup tokens can only…": a setup session can't do this at all. Stop, and tell the person that this step needs an admin signed in normally (`vkit login`).
- It starts "This organization is already set up…": the organization already has an active policy bundle that this session didn't deploy, so this session can't change it. Tell the person, and **ASK** them which option they want:
  1. Leave it as it is.
  2. Approve a new setup session with "Allow this session to replace existing policies and datasources" ticked. To do this, run `vkit logout --format json`, then start this step over.
  3. Run the step themselves after signing in with `vkit login`.

### 3. Add the datasource

First, see what's already there:

```bash
vkit datasource list --format json
```

`result.datasources` lists each datasource by `name`. If the person's database is already there, ask them whether to use it. If yes, skip to step 4 with that name as `<id>`.

**ASK** the person for these. None of them are secret:

- an id for the datasource: a short name such as `shop` or `analytics`;
- the database's host, port and database name, as VaultKit's servers reach them. `localhost` only works if VaultKit runs on the same machine as the database;
- the username of a read-only database role;
- the region: `EU`, `US`, `CA` or `APAC`;
- the environment: `production`, `staging`, `development` or `sandbox`.

VaultKit supports PostgreSQL today, so the engine is `postgres`.

Suggest a dedicated read-only role, and say why. Every query an agent runs goes through this role. If the role can only read, then no mistake in a policy, a query or VaultKit itself can change or delete data. It shouldn't be a superuser or the owner of the tables, either: those can bypass row-level security. VaultKit can't check the role yet, which is why `vkit doctor` will warn about it. If the person doesn't have one, they (or their database admin) can create it like this. Don't run it yourself:

```sql
CREATE ROLE vaultkit_ro LOGIN;
\password vaultkit_ro
GRANT CONNECT ON DATABASE <database> TO vaultkit_ro;
GRANT USAGE ON SCHEMA public TO vaultkit_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO vaultkit_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO vaultkit_ro;
```

**ASK**: write out the command with their values, and ask the person to run it themselves, in their own terminal on this computer, signed in as the same user account you run as (so it uses the setup session from step 2):

```bash
vkit datasource add --id <id> --engine postgres --username <role> \
  --config '{"host":"<host>","port":<port>,"database":"<database>"}' \
  --region <region> --environment <environment> --password-prompt
```

It asks for the password with echo off, and the password goes straight to VaultKit. Never ask for it, never run this command with a password, never use `--password` or `--password-env`, and never put the password in a file. When the person says it's done, check:

```bash
vkit datasource get <id> --format json
```

- `status: ok` with `result.name` equal to `<id>`: done. The password isn't checked until the scan in step 6.
- `not_found`: it wasn't added. Ask the person what the command printed.

If the person shows you an error from their command, or you run it yourself to check:

- `secret_required` (`needs_input`): the command ran without a terminal for the password. That's expected if you run it. The person has to run it themselves.
- `datasource_exists`: a datasource with this id already exists with different settings, and vkit can't change it. Show the person `vkit datasource get <id> --format json`, and **ASK** whether to use it as it is or add theirs under a new id.
- `invalid_config`: the `--config` JSON is malformed. Fix the command and ask them to run it again.
- `admin_required`: only admins can add datasources. Ask the person to check that the admin who approved the setup session is still an admin.

Rerunning the same command is safe: a datasource with the same id and settings returns `result.already_existed: true`, and the password typed then isn't used.

### 4. Choose policy packs

Read each pack's description from vkit rather than describing the packs yourself:

```bash
vkit policy pack info starter --format json
vkit policy pack info ai_safety --format json
vkit policy pack info financial_compliance --format json
```

Each returns `result.description` (what the pack does) and `result.policies` (its policy ids). Ignore `next` here: step 5 installs the packs.

The starter pack is always installed. Tell the person what it does, using its `result.description`.

**ASK** whether to add `ai_safety`, `financial_compliance`, both or neither. For each one, give its `result.description`. If they want details, list its `result.policies`. Add only the packs they choose.

### 5. Create the policy project

The policies live in a folder in git. By default it's `vaultkit-policies` in the current directory. Tell the person where it will be, and use another place if they ask.

```bash
vkit init --dir vaultkit-policies --with starter,<extra packs> --format json
```

`<extra packs>` is what they chose in step 4, comma-separated, for example `--with starter,ai_safety`. With no extra packs, use `--with starter`.

- `status: ok`. `result.packs` lists what was installed. `result.git_initialized` is `true` if vkit ran `git init`, and `false` if the folder is already inside a git repository, which is fine: the policies are committed there.
- On a rerun, `result.already_existed: true`, and nothing changes.
- `unknown_pack`: `result.available_packs` lists the real names. Fix the name.
- `pack_dependency_missing`: include `starter` in `--with`.

Run every later command from inside the folder:

```bash
cd vaultkit-policies
git add -A . && git commit -m "Add VaultKit policies" -- .
```

`-- .` commits only this folder, so nothing else the person is working on ends up in the commit. If there is nothing to commit (on a rerun), move on. If git refuses because `user.name` or `user.email` isn't set, ask the person to set them. Don't set them yourself.

### 6. Scan the database

Preview first. Scanning reads the schema, never the data, and this run changes nothing:

```bash
vkit scan <id> --format json
```

`result.datasets` lists each table with its `columns`. Each column has `sensitivity`, `category`, `unknown`, `unreviewed`, `change` and `suggestion`. Tell the person in one line how many tables and columns VaultKit found, and how many are unknown (`result.unreviewed_count`).

Then apply it and write the registry file:

```bash
vkit scan <id> --mode apply --format json
vkit registry export --force --format json
```

- The apply scan returns `result.applied: true`. On a rerun with nothing new, it returns `result.unchanged: true`.
- The export returns `result.path` (`datasets/registry.yaml`) and `result.datasets`. `--force` is needed because `vkit init` already created a placeholder `datasets/registry.yaml`.
- On a rerun, after the person has reviewed columns in step 7: if the apply scan returned `unchanged: true` and `datasets/registry.yaml` is committed, skip the export, because it would overwrite their review with the control plane's copy until step 8 deploys it. If you did export, run `git diff datasets/registry.yaml`. If it undoes any of the person's review, restore it with `git checkout -- datasets/registry.yaml`.

Errors:

- `datasource_unreachable`: VaultKit can't reach the host and port. **ASK** the person to check them and that the database is running and reachable from VaultKit's servers. A datasource's settings can't be changed from vkit, so a wrong host means adding it again under a new id (step 3).
- `datasource_connection_failed`: the database refused the connection. A wrong username or password looks like this, and so do the database's connection rules (`pg_hba.conf`, SSL). **ASK** the person to check them. A stored password can't be changed, so a wrong one means adding the datasource again under a new id (step 3).
- `datasource_permission_denied`: the role can't read the schema. **ASK** the person (or their database admin) to grant it `USAGE` on the schema and `SELECT` on the tables (see the SQL in step 3), then rerun.
- `datasource_busy`: wait a minute and rerun.
- `datasource_query_failed`, `query_runtime_unavailable`: stop, and tell the person the message. Whoever runs VaultKit needs to look at it.
- `not_a_project`: you're not in the policy folder. `cd` into it.

### 7. Review the classifications

VaultKit's classifier labels every column with a sensitivity (`low`, `medium`, `high` or `unknown`) and a category (`internal`, `pii`, `financial`, `credential` or `unknown`). Policies decide by these labels, so the person should check them.

From the scan in step 6 (rerun `vkit scan <id> --format json` if you need it again), show the person a short table, one row per column:

- that is `unknown` or `unreviewed`, or has a `suggestion`;
- whose category is `pii`, `financial` or `credential`.

| Column | Sensitivity | Category | Why it's listed |
| --- | --- | --- | --- |
| `customers.region` | unknown | unknown | unknown: not classified |
| `customers.email` | medium | pii | personal data |
| `api_keys.key_value` | high | credential | never returned |

Leave out columns that are `low`/`internal` with no flag. If there are more than about 30 rows, show the unknown, unreviewed and suggested ones first, and give counts for the rest.

Explain what the labels do:

- The starter pack masks unreviewed and unknown columns for agents until someone reviews them.
- `pii` is masked for everyone, even after approval.
- `credential` is never returned.
- `high` is masked, and needs a data owner's approval in production.
- With `ai_safety`, agents also need approval for personal, unreviewed and row-level financial data in production.

**ASK** which labels to change. Make only the changes the person asks for, and never decide one yourself. For each change, edit that column's `sensitivity:` and/or `category:` line under its table's `fields:` in `datasets/registry.yaml`. Don't change anything else in the file. If they're happy with the labels as they are, change nothing. Unknown columns stay masked for agents, and `vkit doctor` will warn about them.

To check the edits:

```bash
vkit registry diff --format json
```

It compares the file with the control plane. After edits, it exits 1 with `result.changed: true`, which is expected until step 8 deploys them. Exit 2 is an error: show the person the message.

### 8. Deploy the policies

The bundle's version is the git commit, so commit first:

```bash
git add -A . && git commit -m "Review VaultKit registry" -- .
vkit policy bundle --format json
vkit policy validate --format json
vkit policy deploy --format json
```

Skip the commit if `git status --porcelain .` prints nothing.

- `bundle` returns `status: ok` with `result.bundle_version` (the commit), `result.dirty: false`, `result.policies` and `result.packs`.
- `validate` returns `result.valid: true`.
- `deploy` returns `result.state: "active"` and `result.already_deployed: false`. On a rerun of a bundle it already deployed, it returns `result.already_deployed: true`.
- `deploy` returns `result.warnings`. For each warning with `code: approval_group_has_no_members`, tell the person: "Approvals for <group> will come to you as an org admin until you add people to that group in the console." Don't ask them to create groups now. A rerun returns an empty `warnings` list. That's fine: `vkit doctor` lists the groups again in step 11.

Errors:

- `bundle_not_committed`: something in the policy folder isn't committed. Run the commit above, then bundle again. Never use `--allow-dirty`.
- `bundle_not_versioned`: the folder isn't in a git repository with a commit. Go back to step 5.
- `policy_invalid`, `bundle_invalid`: show the person the message. If it names `datasets/registry.yaml`, it's probably a value from step 7 (a sensitivity or category that isn't one of those listed there). Fix that edit with the person, commit, and bundle again. For anything else, stop. Don't edit the policies.
- `bundle_version_exists`: a bundle with this version (the commit) but other contents was already uploaded. Rebundling and redeploying the same commit isn't this error, it returns `result.already_deployed: true`. Stop, show the person `error.message`, and ask what to do. Never use `--version`.
- `bundle_checksum_exists`: the same policies were already uploaded under another version, which the message names. If that version is active, the step is done. If not, tell the person that an admin can activate it in the console.
- `setup_scope_forbidden`: see "Setup sessions" in step 2.

### 9. Create the agent token

You'll use an agent token to query VaultKit, and only the person may ever see it. A setup session can't create tokens, so the person creates it in the console. Your `vkit` stays on the setup session throughout.

Get the values that aren't secret:

```bash
vkit whoami --format json
```

`VAULTKIT_BASE_URL` is `result.endpoint` and `VAULTKIT_ORG` is `result.organization`. `result.email` is the admin who approved the setup session.

**ASK** the person for:

- a name for the agent token, such as `<project>-claude-code` or `<project>-cursor`;
- who the agent acts for: usually themselves. Its requests show up in that person's Activity;
- how long the token should last: 1 hour, 24 hours, 30 days, or a custom number of hours or days.

If any table in `datasets/registry.yaml` has a `tenant_column:` other than `false`, it's tenant-scoped. A token without a tenant can't read those tables at all, so also **ASK** which tenant the agent acts for.

**ASK**: ask the person to create the token in the console. The console is the site of the approval link from step 2 (`verification_url_complete`); the page is `/agents` there.

1. Open the Agents page and click **Create agent token**.
2. Fill in **Agent name** with the name they chose. Leave **Acts for** on themselves unless they chose someone else, and pick **Expires in**.
3. If the org has tenant-scoped datasets, the dialog opens **Advanced** and warns that a token without a tenant can't read them. Enter the tenant there. Leave clearance and region unset unless they know their policies use them.
4. Click **Create token**. The dialog shows the token once, and below it three `export` lines with `VAULTKIT_BASE_URL` and `VAULTKIT_ORG` filled in. Check with them that those two match `result.endpoint` and `result.organization` above.
5. Copy the three lines into their shell profile (`~/.zshrc`, `~/.bashrc`) or their secret manager, and replace `<paste token>` with the copied token. Never put it in a `.env` file inside the repository, and never paste the token into this chat.
6. Tick **I've saved the token**, then click **Done**.

Warn them not to use `VKIT_TOKEN` or `VKIT_ENDPOINT` for the agent token. Those are read by the `vkit` CLI itself, so every `vkit` command would then run as the agent.

Don't try to check the agent token. Your environment won't have it until step 10's restart.

**If the Agents page has no Create agent token button,** the control plane is older and only the CLI can create tokens. Ask the person to run these in a separate terminal window that you can't see:

```bash
vkit logout
vkit login --endpoint <VAULTKIT_BASE_URL>
vkit agents tokens create --name <name> --for <email> --expires-in <duration>
vkit logout
```

Add `--tenant <tenant>` when a tenant is needed. `tokens create` prints the token once; with `--secret-file <path outside this repository>` it's written to that file (mode 600) instead. The person sets the three variables as in step 5 above.

The last `vkit logout` matters. vkit keeps one login per computer, shared with your commands, so while they're signed in, your `vkit` commands would act as their full admin login. When they're done, check that `vkit whoami --format json` returns `not_logged_in`. Then do step 2 again, and tell them why a second approval is needed: the rest of setup should run on a session that can only set VaultKit up and ends in 30 minutes, not on their own login.

### 10. Configure the MCP server

The VaultKit MCP server gives you the tools `vaultkit_discover`, `vaultkit_describe`, `vaultkit_query` and `vaultkit_check_approval`. It needs Python 3.10 or later:

```bash
python3 --version
pipx install 'vaultkit[mcp]'
command -v vaultkit-mcp
```

If `pipx` isn't installed, use `python3 -m pip install --user 'vaultkit[mcp]'`. **ASK** before installing Python or pipx themselves. `command -v vaultkit-mcp` prints the server's full path; use it as `command` below if the client might not share the shell's `PATH`.

Add the server to this client's config. Reference the environment variables; never write their values. Keep any other servers already in the file.

**Claude Code**: `.mcp.json` at the project root:

```json
{
  "mcpServers": {
    "vaultkit": {
      "type": "stdio",
      "command": "vaultkit-mcp",
      "args": [],
      "env": {
        "VAULTKIT_BASE_URL": "${VAULTKIT_BASE_URL}",
        "VAULTKIT_ORG": "${VAULTKIT_ORG}",
        "VAULTKIT_TOKEN": "${VAULTKIT_TOKEN}"
      }
    }
  }
}
```

Claude Code asks the person to approve a project's `.mcp.json` servers the first time.

**Cursor**: `.cursor/mcp.json` in the project, or `~/.cursor/mcp.json` for every project:

```json
{
  "mcpServers": {
    "vaultkit": {
      "type": "stdio",
      "command": "vaultkit-mcp",
      "args": [],
      "env": {
        "VAULTKIT_BASE_URL": "${env:VAULTKIT_BASE_URL}",
        "VAULTKIT_ORG": "${env:VAULTKIT_ORG}",
        "VAULTKIT_TOKEN": "${env:VAULTKIT_TOKEN}"
      }
    }
  }
}
```

**Any other client**: add the MCP server with these three environment variables. The command is `vaultkit-mcp` and it takes no arguments. Set `VAULTKIT_BASE_URL`, `VAULTKIT_ORG` and `VAULTKIT_TOKEN` from the environment, using that client's syntax for referencing environment variables, never literal values.

These files hold no secrets. Ask the person whether they want them committed.

**ASK**: you only see the new environment variables and the MCP server after a restart. Ask the person to:

1. quit this agent session;
2. open a new terminal, so the shell profile is read again;
3. start the agent client from there, for example `claude` or `cursor .`. On macOS, an app opened from the Dock doesn't read the shell profile;
4. approve the `vaultkit` MCP server if asked;
5. paste the same prompt again.

Setup then resumes at step 11.

### 11. Check everything

From the policy project folder, check that vkit is on a setup session, then run doctor:

```bash
vkit whoami --format json
vkit doctor --format json
```

Run doctor only if `whoami` returns `result.setup_session`. If it doesn't, follow the rule about setup sessions under Rules, then come back here.

`result.ok` is `true` and the command exits 0 when nothing failed. `result.summary` counts the checks by status, and each entry in `result.checks` has `name`, `status` (`pass`, `warn`, `fail` or `info`), `detail` and `fix`. `result.skipped` lists checks that didn't run because an earlier one failed.

Work through the `fail` checks in order:

- Follow `fix` yourself only when it's a step of this file you're allowed to do, for example step 8 for `policy_bundle` ("isn't deployed"), or step 6 for `schema` changes. Then run doctor again.
- Otherwise, tell the person `detail` and `fix`. Some common ones:
  - `agent_token` "…aren't set": your environment doesn't have the variables. They weren't set, or you weren't restarted from a shell that has them (step 10).
  - `agent_token` "…is a person's login, not an agent token": `VAULTKIT_TOKEN` holds the wrong thing (step 9).
  - `login` "…signed in with an agent token…": `VKIT_TOKEN` is set to the agent token. Ask them to unset `VKIT_TOKEN` and `VKIT_ENDPOINT`.
  - `login` expired: the setup session ended. Do step 2 again, then rerun doctor.
  - `mcp_server` skipped as not installed: go back to step 10.

Tell the person about each `warn`. `datasource_role` always warns for now, because VaultKit can't check the role yet. Pass on each `info` too: an approval group with no members, whose requests org admins approve, or a check the control plane is too old to answer.

Setup is done when doctor reports no failures. Warnings and infos are fine.

### 12. Wrap up

Tell the person what was set up:

- the datasource `<id>`;
- the policy folder and the packs in it;
- the bundle version that's active;
- which approval groups currently come to them as an org admin;
- the agent token's name and when it expires;
- which MCP config file you wrote.

Then try one safe query with the MCP tools:

1. `vaultkit_discover`, to list the datasets.
2. `vaultkit_describe` on one of them.
3. `vaultkit_query` on that dataset: a count grouped by one column whose `access` is `allow` in the `vaultkit_describe` result and that isn't personal data. For example, `{"dataset": "orders", "fields": ["status"], "aggregates": [{"func": "count", "field": "*", "alias": "n"}]}`.

Show the person the result. If the query needs approval or is denied, explain what the response says and don't retry. If the VaultKit tools aren't available in this session, say so and point the person back to step 10.

Finally, end the setup session, since setup is done:

```bash
vkit logout --format json
```

Start with step 1 now. Tell the person what you're doing before each ASK step.
