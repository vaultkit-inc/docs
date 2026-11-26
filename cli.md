# VaultKit CLI (`vkit`)

**Applies to:** vkit 2.0.0. Setup sessions need a control plane that has them (see
[Upgrading to vkit 2.0.0](upgrading_vkit_2.md#control-plane-version)).

`vkit` is the command-line client for VaultKit. People use it to request governed
data and approve requests; admins use it to register datasources, scan schemas, and
build and deploy policy bundles from git. A coding agent can drive it too: every
command takes `--format json`, and secrets never go on the command line.

Upgrading from vkit 1.x? Read [Upgrading to vkit 2.0.0](upgrading_vkit_2.md) first:
some JSON output and `vkit policy bundle` changed.

## Contents

- [Install](#install)
- [Sign in](#sign-in)
- [Setup sessions](#setup-sessions)
- [Credentials from the environment](#credentials-from-the-environment)
- [Requesting data](#requesting-data)
- [Approvals](#approvals)
- [Grants](#grants)
- [Agent tokens](#agent-tokens)
- [Datasources](#datasources)
- [Schema scans and the registry](#schema-scans-and-the-registry)
- [Policy projects and bundles](#policy-projects-and-bundles)
- [Policy packs](#policy-packs)
- [Checking a setup: `vkit doctor`](#checking-a-setup-vkit-doctor)
- [JSON output](#json-output)
- [Error codes](#error-codes)
- [Scripting and CI](#scripting-and-ci)
- [Troubleshooting](#troubleshooting)

---

## Install

Requires Ruby 3.0 or later and a running VaultKit control plane.

```bash
gem install vaultkit
vkit --version      # vkit 2.0.0
vkit help           # every command; vkit help <command> for one
```

---

## Sign in

```bash
vkit login [--endpoint URL] [--email EMAIL] [--password-prompt]
```

The endpoint comes from `--endpoint`, then `VKIT_ENDPOINT`, then the last endpoint
you used, then a prompt. The control plane decides how you sign in:

- **SSO:** `vkit` opens your browser and waits for you to finish signing in.
- **Email and password:** it asks for your email (unless you passed `--email`) and
  your password. The password is never accepted as an option. `--password-prompt`
  reads it from the terminal (`/dev/tty`) with echo off, never from stdin.
- **API token:** it reads `VAULTKIT_TOKEN`, or asks for a token.

Under `--format json`, `vkit login` never prompts. It returns `needs_input` (exit 3)
instead: `endpoint_required` without an endpoint, `email_required` for a password
login without `--email`, and `secret_required` for a password or token login without
`--password-prompt`, or with it but no terminal. An SSO login works as it is.

| Command | What it does |
| --- | --- |
| `vkit whoami` | Shows who you're signed in as, and for a setup session, until when. If the control plane can't be reached, it shows the cached identity with a warning. |
| `vkit logout` | Signs out of the saved login (the token is deleted; the endpoint is kept). A setup session is ended on the control plane too. |
| `vkit reset [--yes]` | Asks for confirmation (unless `--yes`), then deletes every stored credential and setting. |

Credentials are stored in the macOS Keychain, in the Secret Service on Linux when
`secret-tool` is installed, or otherwise in `~/.vkit/credentials.json` (mode 0600).
There's one saved login per computer user, shared by every terminal and by any agent
running as that user.

Every request carries a `User-Agent` of the form
`vkit/<version> (<os> <cpu>; ruby <version>)`, such as
`vkit/2.0.0 (macOS arm64; ruby 3.3.8)`. The console's setup approval page shows it.

---

## Setup sessions

A setup session lets a coding agent set VaultKit up for you without holding your own
login. The agent asks for one, you approve it in the console, and it gets a token that
can only do setup, for 30 minutes. See [Setup Sessions](setup_sessions.md) for what
the token can and can't do.

```bash
vkit login --setup [--endpoint URL] [--no-browser]
vkit login --setup --wait [--timeout SECONDS]
```

**In a terminal** it's one command. It prints a code and the console's approval link,
opens the link in your browser (`--no-browser` skips that), and waits:

```
🔐 An admin approves this setup session in the VaultKit console:
   1. Open https://vaultkit.example.com/setup/approve
   2. Check the code matches: BJCQ-6006
   3. Approve it (by 07:17, when the request expires)
   Only approve it if you just started it in your own terminal.
```

**Under `--format json`** it's two commands, so the agent can show you the code
before it waits:

1. `vkit login --setup --format json` returns `needs_input` (exit 3) with
   `error.code: setup_approval_required`. `result` has `user_code`,
   `verification_url`, `verification_url_complete` (the link with the code filled in)
   and `expires_at`. Running it again returns the same code.
2. `vkit login --setup --wait --format json` waits until the session is approved
   (`status: ok`, `result.setup_session: true`, `result.expires_at`), denied
   (`setup_session_denied`) or expired (`setup_session_expired`). With `--timeout`,
   a session still waiting returns `setup_approval_required` again.

The code must be approved within 10 minutes. The token then lasts 30 minutes, and
it's saved where a normal login is. Logging in this way replaces a saved personal
login, and vkit says so; run `vkit login` afterwards to sign in as yourself again.

- **`vkit whoami`** shows `Setup session: setup only, until <time>`, and with
  `--format json`, `result.setup_session.expires_at`.
- **`vkit logout`** ends the session on the control plane, and prints
  `🔒 Setup session ended on the control plane` (`result.setup_session_revoked: true`).
  If the control plane can't be reached, the token is still cleared locally, with a
  warning that it stays valid until it expires.
- **When it expires** part way through setup, commands return `setup_session_expired`.
  Run `vkit login --setup` again and approve the new session; every setup step is
  safe to rerun.

---

## Credentials from the environment

Scripts, CI jobs and demos can supply credentials for one command or shell without
touching your saved login:

```bash
export VKIT_ENDPOINT=https://vaultkit.example.com
export VKIT_TOKEN=<a VaultKit token>
vkit request < query.json
```

- **Both variables are needed.** Together they replace the saved login for every
  command except `vkit login` and `vkit logout`. One without the other is an error.
- **A notice goes to stderr** while they're in use. The token is never printed.
- **To act as someone else for one command,** set the token on that command:
  `VKIT_TOKEN="$APPROVER_TOKEN" vkit approval:approve 42`.
- **Unset both** to go back to your saved login.

**`VKIT_TOKEN` isn't `VAULTKIT_TOKEN`.** `VKIT_TOKEN` is vkit's own login.
`VAULTKIT_TOKEN`, with `VAULTKIT_BASE_URL` and `VAULTKIT_ORG`, is what an agent (the
SDK or the MCP server) uses. Never put an agent token in `VKIT_TOKEN`: every `vkit`
command would then run as the agent. `vkit doctor` fails when it finds one there.

Requires vkit 1.1.6 or later.

---

## Requesting data

### `vkit request`

```bash
vkit request --aql '<AQL JSON>' [--format table|json]
vkit request < query.json
```

The query is [AQL](aql_specification.md). Policy decides what happens:

| Result | Output | Next step |
| --- | --- | --- |
| Granted | `✅ ACCESS GRANTED`, the grant ref, expiry and masked fields | `vkit fetch --grant <ref>` |
| OK | `✅ OK — N rows` and the rows. You already had a grant for this exact query, so it was reused. | — |
| Queued | `⏳ QUEUED for approval`, the request ID, the approver groups and the reason | Wait for approval, then run the same request again |
| Denied | `❌ DENIED` and the reason. Exit code 2. | — |

With `--format json`, the control plane's answer is the envelope's `result`
(`result.status` is `granted`, `ok`, `pending_approval` or `queued`), and `next`
names the command to run next. A denial is `status: error` with
`error.code: request_denied`, and still exits 2.

Your role, clearance, region and tenant come from your account or agent token, and
the environment and dataset region from the datasource. A request can't set them:
the older `--env`, `--requester-region`, `--dataset-region` and
`--requester-clearance` options are still accepted, but the control plane ignores
them.

### `vkit requests list`

```bash
vkit requests list [--state all|pending|approved|denied] [--format table|json]
```

Your past requests, newest first. With `--format json`, they're in
`result.requests`.

---

## Approvals

When a policy requires approval, someone in each named approval group must sign off.
Admins and users who can approve any request can sign too, and org admins approve for
a group nobody is in. The requester can't approve their own request, and each group's
sign-off has to come from a different person. You can also approve in the console,
under **Approvals**. See [Approvals and grants](approvals_and_grants.md#approvals).

| Command | What it does |
| --- | --- |
| `vkit approval:list [--state pending\|approved\|denied\|all]` | Lists approvals (default: pending), with the requester, fields, required and outstanding groups, and the reason. |
| `vkit approval:approve ID [--ttl SECONDS]` | Signs off for one group you belong to. The grant is issued once every group has signed. `--ttl` can shorten the grant, never extend it past the policy's TTL. |
| `vkit approval:deny ID [--reason TEXT]` | Denies the whole request. Asks for a reason if you don't give one (under `--format json`, it returns `reason_required`). |
| `vkit approvals:watch [--interval 3] [--since 10m] [--format table\|json] [--pretty]` | Watches for new pending approvals. With `--format json` it prints one JSON object per approval as it arrives, not an envelope. |

`ID` is the number `approval:list` shows. Approving fails if the policy bundle
changed while the request waited; the requester then asks again.

---

## Grants

A grant is VaultKit's permission to run one exact query, for a limited time.

| Command | What it does |
| --- | --- |
| `vkit fetch --grant REF [--format json\|table]` | Runs the granted query. With no `--format`, it prints a status line, the query metadata, then the rows as JSON. With `--format json`, one envelope: `result.rows`, `result.meta` and `result.grant`. |
| `vkit grant:revoke --grant REF --reason TEXT [--yes]` | Admin only. Revokes a grant before it expires. Asks for confirmation unless `--yes` (or `--force`). |

A grant stops working when it expires, is revoked, or the active policy bundle
changes. Asking the same question again within the grant's lifetime reuses it. See
[Approvals and grants](approvals_and_grants.md).

---

## Agent tokens

Agents authenticate with agent tokens rather than a user login. Only admins can
issue and revoke them. The easiest way to issue one is the console: **Agents**,
**Create agent token**. See [Creating agent tokens](mcp_usage.md#creating-agent-tokens).
The CLI does the same:

### `vkit agents tokens create`

```bash
vkit agents tokens create --name NAME [--expires-in 24h] [--tenant ID] \
  [--clearance low|medium|high] [--region EU|US|CA|APAC] [--label TEXT] [--for EMAIL] \
  [--secret-file PATH]
```

- `--expires-in` takes hours or days: `1h`, `24h`, `30d`.
- `--tenant` confines the token to one tenant's rows on datasets marked with
  `tenant_column`. See [Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets).
  A token without one can't read those datasets at all: VaultKit answers as if they
  didn't exist. So vkit warns, after issuing the token, when the org has
  tenant-scoped datasets. It names up to five of them and suggests `--tenant`.
- `--clearance` and `--region` are what policies see as `requester_clearance` and
  `requester_region`.
- Tenant, clearance and region are fixed when the token is issued; the agent can't
  change them. Every agent token has the role `agent` (the `--role` option is
  ignored by the control plane).
- `--label` describes this token, such as "PagerDuty runner". Tokens with the same
  `--name` are one agent on the console's Agents page.
- `--for` names the person the agent acts for. Its requests then show in that person's
  Activity. Without it, the token acts for no one and only admins see its requests.
- `--secret-file PATH` writes the token to a new file with mode 600 instead of
  printing it (`🔑 Token written to PATH (mode 600)`). It never overwrites a file
  (`secret_file_exists`). Under `--format json` it's required, and the token is never
  in the output; without it, nothing is issued (`secret_file_required`).
- Without `--secret-file`, the secret is printed once.

The agent then needs three variables where it runs:

```bash
export VAULTKIT_TOKEN="<the token>"
export VAULTKIT_BASE_URL="https://vaultkit.example.com"
export VAULTKIT_ORG="<your org slug>"
```

Put them in a shell profile or a secret manager, not in a `.env` file inside a
repository, and not in `VKIT_TOKEN` (see
[Credentials from the environment](#credentials-from-the-environment)).

For an assistant that serves several customers, issue a short-lived token per
customer session:

```bash
vkit agents tokens create --name support-assistant --tenant org_42 --expires-in 1h
```

A setup session can't create agent tokens (`setup_scope_forbidden`): sign in as
yourself, or use the console.

### `vkit agents tokens list` and `revoke`

```bash
vkit agents tokens list [--agent NAME] [--format table|json]
vkit agents tokens revoke --token ID_OR_PREFIX --reason TEXT [--yes]
```

`list --format json` puts the tokens in `result.tokens`; `--agent` shows one agent's.
`revoke` asks you to type `revoke` to confirm, unless `--yes` (or `--force`). The
reason is required and recorded in the audit log, and the agent sees it in its errors,
so don't put secrets in it. Revoking a token also revokes its live grants and denies
its pending approvals, and a query it has running returns no rows.

---

## Datasources

Admin only. VaultKit stores the database credential (encrypted) and runs queries
itself, so people and agents never hold it. PostgreSQL is supported today.

### `vkit datasource add`

```bash
vkit datasource add \
  --id prod_db \
  --engine postgres \
  --username vaultkit_reader \
  --config '{"host": "db.internal", "port": 5432, "database": "app"}' \
  --region US \
  --environment production \
  --password-prompt
```

- `--region`: `EU`, `US`, `CA` or `APAC`. Policies see it as `dataset_region`.
- `--environment`: `production`, `staging`, `development` or `sandbox`. Policies see
  it as `environment`.
- Use a read-only database role: VaultKit only reads.

The password never goes on the command line. Give it one of two ways:

- `--password-prompt`: type it into the terminal (`/dev/tty`, echo off).
- `--password-env VAR_NAME`: read it from an environment variable. A missing or empty
  variable is `secret_env_missing`, named by the variable only.

`--password` still works, but it's deprecated: it ends up in shell history and agent
transcripts, and vkit warns. Without a password source, the command fails
(`secret_required`; under `--format json` it's `needs_input`, exit 3). Giving two is
`conflicting_password_sources`.

`datasource add` stores the credential without connecting: the first scan is what
checks it.

**Running it again** with an existing `--id`:

- with the same settings (engine, config, region, environment), it changes nothing and
  says the datasource already exists (`result.already_existed: true`). No password is
  needed, and one you give isn't used: the stored credential can't be changed from
  vkit.
- with other settings, it's refused (`datasource_exists`). Add it under a new id.

| Command | What it does |
| --- | --- |
| `vkit datasource list` | Lists datasources. |
| `vkit datasource get ID` | Shows one datasource. |

---

## Schema scans and the registry

The registry lists each dataset's fields with a sensitivity and a category, which
policies act on. It lives in your policy project as `datasets/registry.yaml`.

### `vkit scan`

```bash
vkit scan DATASOURCE [--mode diff_only|apply] [--format human|json]
```

`diff_only` (the default) shows what changed since the last scan: added (`+`),
removed (`-`) and changed (`~`) fields. `apply` saves the scan to the control
plane's registry.

The scan labels each column from its name, type and table:

- **Sensitivity:** `high`, `medium`, `low` or `unknown`.
- **Category:** `pii`, `financial`, `credential`, `internal` or `unknown`.

It reads names and types only, never data. `unknown` means a column that could hold
anything (a `payload` jsonb, a `notes` text) that it couldn't place: review it. Until
you do, the `starter` pack (2.1.0 and later) masks it for agents, and the scan lists it
under its dataset, marked `!`, with a count:

```
  customers                ✓
    ! unreviewed: notes, region

  ⚠️  2 columns are unreviewed and masked until you review them
```

To review one, set its sensitivity in `datasets/registry.yaml`, then bundle and deploy.

A rescan keeps labels you've reviewed (from a deployed bundle or a manual edit). Where
the classifier disagrees, it lists a suggestion instead of changing the label:

```
  api_keys                 ✓
    ? key_value  low/internal → high/credential (suggested)
```

With `--format json`, either mode lists every column of the scanned datasets under
`result.datasets[].columns[]`. Each column has `name`, `type`, `sensitivity`,
`category`, `unknown`, `unreviewed`, `change` (`added`, `removed`, `changed` or
`null`) and `suggestion` (the `?` line, as `current` and `suggested`, or `null`).
`result` also has `mode`, `applied`, `unchanged` (the scan found nothing new),
`unreviewed_count`, `suggestion_count` and `columns_complete` (false when the
registry couldn't be read, so only changed columns are listed).

See [Schema discovery](schema_discovery.md).

### `vkit registry export` and `diff`

```bash
vkit registry export [--dir .] [--out PATH] [--force]
vkit registry diff [--dir .] [--format human|json]
```

`export` writes the control plane's registry to `datasets/registry.yaml`. If the file
exists, it offers to overwrite, show a diff or stop; `--force` (or `--yes`) overwrites
without asking. Under `--format json` it never asks: without `--force` it returns
`overwrite_confirmation_required`.

`diff` compares the two and exits 0 when they match, 1 when they differ and 2 on
error. With `--format json`, a difference is still `status: ok`, with
`result.changed: true` and the changes in `result.datasets`.

---

## Policy projects and bundles

Policies are YAML files in git. You compile them, with the registry, into a bundle and
deploy it; the bundle's version is the git commit it was built from. See the
[Policy Guide](policy_guide.md).

### `vkit init`

```bash
vkit init [--dir .] [--with starter,ai_safety] [--no-git]
```

Creates `config/policies/`, `datasets/registry.yaml`, `dist/`, `.vkit/` and a
`.gitignore`, and installs the packs you name (plus their dependencies). With no
`--with`, it installs `starter`.

- **git.** When the folder isn't inside a git repository, `init` runs `git init`
  (`✓ Initialized a git repository in <dir>`). `--no-git` skips it. It never commits:
  commit the project yourself before the first bundle.
- **`.gitignore`** ignores `dist/` and everything in `.vkit/` except
  `.vkit/packs.yaml`, which is committed:

  ```
  dist/
  .vkit/*
  !.vkit/packs.yaml
  ```

- **Running it again** in a project (a folder with `.vkit/packs.yaml`) changes
  nothing, and doesn't reinstall `starter` (`result.already_existed: true`).

### `vkit policy bundle`

```bash
vkit policy bundle [--policies-dir config/policies] [--registry-dir datasets] \
  [--out dist/policy_bundle.json] [--org ORG] [--version LABEL] [--allow-dirty]
```

Needs a login, and a commit: the bundle is built from what's committed. `--org` must
match the org you're signed in to.

**Version and commit:**

- **The version is the commit.** With no `--version`, the bundle's version is
  `git rev-parse HEAD`, and `source.commit_sha` records the same commit.
- **Uncommitted changes are refused.** If anything under the policies folder, the
  registry folder or `.vkit/packs.yaml` differs from `HEAD` (modified, staged, new,
  deleted, or ignored by `.gitignore`), the bundle fails with `bundle_not_committed`
  and writes nothing. It lists the files and the git commands that commit them.
  Changes elsewhere in the repository, such as a README or `dist/`, don't count.
- **`--allow-dirty`** bundles anyway. It lists the files, and the bundle records
  `"dirty": true` in `source`. Its version is still the commit, so it doesn't match
  that commit's policies; use it for experiments, not for what you deploy.
- **`--version LABEL`** names the bundle, such as `v1.2`. It doesn't skip the commit
  check, and `source.commit_sha` still comes from git. With no repository, or a
  repository with no commits, a bundle needs `--version` (otherwise
  `bundle_not_versioned`), and it has no `commit_sha`.
- **Rebuilding the same commit gives the same bundle.** The bundle's `created_at` is
  the commit's time, not the clock, so the checksum is the same every time, and
  deploying it again changes nothing (`already_deployed`). A dirty bundle takes its
  time from the newest input file instead, so the same files give the same checksum.
  A commit first deployed by vkit 1.x still gets `bundle_version_exists` once when
  rebuilt with 2.0.0: commit again, then bundle.

```
❌ These files aren't committed, so the bundle's version wouldn't match its policies:
   config/policies/starter__05__default_ttl_for_allows.yaml
   Code: bundle_not_committed
   Fix:
     git add -A -- config/policies datasets .vkit/packs.yaml
     git commit -m "Update VaultKit policies"
```

The bundle is refused, with every problem listed, when a policy can never match or a
dataset would leak across tenants:

- a key the engine ignores, such as `match.fields.category` (use `contains`);
- a tag or sensitivity no field has and the classifier can't produce;
- a dataset that isn't in `datasets/registry.yaml`;
- an invalid environment, region or clearance value, or `time` under top-level
  `context` (it works under `match.context`);
- a mask policy with no fields to mask;
- a `match.fields.usage` other than `row` or `aggregate`;
- an approval policy with neither `approver_role` nor `approver_roles`, or with both,
  an empty or repeated group, an `approval_mode` other than `all` or `any`, or an
  `approval_mode` with a single `approver_role`;
- an approver group named `admin`, `member`, `approver` or `agent`: these are built-in
  roles, not groups (vkit 2.0.0);
- a `tenant_column` that isn't one of its dataset's fields, or a dataset that has
  another dataset's tenant column but isn't marked (`tenant_column: false` declares a
  table every tenant may read).

If the problems are in pack policies, `vkit policy pack upgrade` replaces them.

The bundle also checks every installed pack. It fails, exits 1 and writes nothing when
a pack is missing files, has files from another version, or is missing policies, and
it names the pack and the files. `vkit policy pack upgrade` restores missing files.
Every bundle prints how many policies come from each pack and from the repo, so a
dropped pack shows even when nothing fails.

### `vkit policy validate`, `deploy` and `revoke`

```bash
vkit policy validate [--bundle dist/policy_bundle.json] [--schema PATH]
vkit policy deploy [--bundle dist/policy_bundle.json] [--no-activate] [--format human|json]
vkit policy revoke --bundle-version VERSION [--reason TEXT] [--yes]
```

- `deploy` and `validate` refuse a bundle that leaves out one of its packs' policies.
- `deploy` uploads the bundle and makes it active; `--no-activate` only uploads it.
- **Deploying the same bundle again** changes nothing: `result.already_deployed: true`
  (it's active) and `result.already_existed: true`. A version can't be uploaded twice
  with different contents (`bundle_version_exists`), and the same contents under
  another version are refused too (`bundle_checksum_exists`).
- **Approval groups with no members.** `deploy` warns about each approval group the
  bundle names that has no active members, and still deploys:

  ```
  ⚠️  data_owner has no members, so org admins will approve its requests until someone is added.
  ```

  With `--format json`, they're in `result.warnings`, each
  `{"code": "approval_group_has_no_members", "group": ..., "message": ...}`.
- `revoke` revokes a bundle version: grants issued under it stop working. It asks
  for confirmation unless `--yes` (or `--force`).

---

## Policy packs

Packs are ready-made, versioned policies. See [Policy Packs](pack_system.md).

| Command | What it does |
| --- | --- |
| `vkit policy pack list` | Shows available packs, and which are installed or have an upgrade. |
| `vkit policy pack info NAME` | Shows a pack's version, description, dependencies and policies. Works outside a project too. With `--format json`, the description is `result.description`. |
| `vkit policy pack add NAME [--force]` | Installs a pack into `config/policies/`. Adding an installed pack changes nothing (`✓ starter is already installed`). |
| `vkit policy pack remove NAME [--force]` | Removes a pack's files (only the ones it installed). |
| `vkit policy pack upgrade NAME [--force] [--restore]` | Replaces an installed pack with the shipped version. If its files are missing, offers to restore them; `--restore` (or `--yes`) does it without asking. |

Every pack command takes `--dir` (default `.`). Commit the changes, then bundle and
deploy. An unknown pack name is `unknown_pack`, and `result.available_packs` lists
the shipped ones.

---

## Checking a setup: `vkit doctor`

```bash
vkit doctor [--dir .] [--format human|json]
```

Run from the policy project (or point `--dir` at it), `vkit doctor` checks whether
VaultKit is set up and working, and says how to fix what isn't. It changes nothing,
and it never prints a token. It exits 0 when no check failed, and 1 otherwise.

```
vkit doctor

  ✓ CLI version       vkit 2.0.0 (the latest).
  ✓ Control plane     Reachable at https://vaultkit.example.com.
  ✓ Login             Signed in as ada@example.com (admin, org acme); the session expires at 14:05 (in 52 minutes).
  ✓ Datasources       1 datasource (shop); VaultKit connected to each and read its schema.
  ! Datasource role   Not verified: VaultKit can't yet check which database role shop connects as (https://github.com/vaultkit-inc/vaultkit/issues/125).
                      Fix: Use a dedicated read-only role, not a superuser or the table owner. See https://github.com/vaultkit-inc/vaultkit/issues/125
  ✓ Schema            3 datasets, 16 columns in the registry, all classified; the schema matches the last scan.
  ✓ Policy bundle     2a54ee901784 is active: this repository's HEAD.
  i Approval routing  2 groups: data_owner (0 members), finance_manager (1 member). data_owner has no members, so org admins approve its requests.
                      To have someone else approve, add them to data_owner in the console (Team).
  ✓ Agent token       Agent support-assistant (agent) is set up for acme at https://vaultkit.example.com, and can see 3 datasets.
  ✓ MCP server        vaultkit-mcp (/usr/local/bin/vaultkit-mcp) starts and lists vaultkit_discover, vaultkit_query, vaultkit_check_approval.

  8 passed, 1 warning, 0 failed, 1 info
```

| Mark | Status | Meaning |
| --- | --- | --- |
| `✓` | `pass` | Working. |
| `!` | `warn` | Worth knowing or fixing, but nothing is broken. |
| `✗` | `fail` | Broken; its fix says what to do. Doctor exits 1. |
| `i` | `info` | Neither a pass nor a problem: something to know (a group org admins approve for), or a check an older control plane can't answer ("the control plane is too old to report it"). |
| `-` | skipped | Didn't run, because a check it needs failed (the line names it), or because it's optional and not installed. |

The checks, in order:

| Check | What it checks |
| --- | --- |
| `cli_version` | Warns when RubyGems has a newer vkit. Passes, saying so, when it can't reach RubyGems. |
| `control_plane` | `GET /up` at the configured endpoint. Fails when it can't be reached, or when only one of `VKIT_ENDPOINT` and `VKIT_TOKEN` is set. |
| `login` | Who vkit is signed in as, for a person's login or a [setup session](#setup-sessions). Warns in the session's last 10 minutes, and fails once it has expired, or when vkit is signed in with an agent token. |
| `datasources` | Lists the datasources and runs a `diff_only` scan of each, which reads the schema and never data. Fails when there are none or none connect, and warns when only some do. Skipped for non-admins. |
| `datasource_role` | Always a warning for now: VaultKit can't yet check that the role is read-only ([vaultkit#125](https://github.com/vaultkit-inc/vaultkit/issues/125)). |
| `schema` | Fails when a datasource has nothing in the registry. Warns when the schema changed since the last applied scan, or a column has no classification or is `unknown`. |
| `policy_bundle` | Compares the active bundle with this repository. Passes when it's `HEAD`, or an earlier commit whose policies, registry and `.vkit/packs.yaml` are the same as `HEAD`'s. Fails when no bundle is active, or those files changed after it. Warns when the active version isn't a commit in this repository (a `--version` label, another clone), or the folder isn't a policy project. |
| `approval_routing` | Lists the approval groups the active policies name and their members. `info` for a group with no members, whose requests org admins approve. Fails only if the groups can't be read. |
| `agent_token` | Checks `VAULTKIT_BASE_URL`, `VAULTKIT_ORG` and `VAULTKIT_TOKEN` are set, that the token is an agent token for that org, and which datasets it can see. Fails when `VKIT_TOKEN` holds an agent token. |
| `mcp_server` | If `vaultkit-mcp` is installed, starts it with the agent's variables and checks it lists `vaultkit_discover`, `vaultkit_query` and `vaultkit_check_approval`. Skipped, with the `pip install` command, when it isn't. |

With `--format json`, `result.checks` lists each check as
`{name, status, detail, fix}`, `result.skipped` lists the checks that didn't run as
`{name, reason}`, and `result.summary` counts them (`pass`, `warn`, `fail`, `info`).
When a check fails, the envelope's `status` is `error` with
`error.code: doctor_checks_failed`, and the checks are still in `result`:

```bash
vkit doctor --format json | jq -r '.result.checks[] | select(.status == "fail") | "\(.name): \(.fix)"'
```

---

## JSON output

Every command takes `--format json`. It prints exactly one JSON object to stdout:

```json
{
  "status": "ok",
  "result": { },
  "next": ["<the command to run next>"],
  "error": { "code": "<stable_code>", "message": "...", "fix": "<one sentence: what to do>" }
}
```

- **`status`** is `ok`, `error` or `needs_input`. `needs_input` means a person has to
  do something first (type a secret, confirm, approve); `error.fix` says what, and
  running the same command again unchanged won't help.
- **`result`** is the command's output. **`next`** lists the commands that usually
  come next. **`error`** is there only when `status` isn't `ok`.
- **Everything else goes to stderr**: the human output, progress, warnings and
  notices. stdout has only the JSON.
- **Secrets never appear** in the output, error messages included.
- **Arguments vkit itself refuses**, such as a missing option or an unknown command,
  are envelopes too (`invalid_arguments`).
- **`vkit approvals:watch --format json`** is the exception: it's a stream of one JSON
  object per approval, as in 1.x. An error before the stream starts is an envelope.

Exit codes:

| Code | Meaning |
| --- | --- |
| `0` | `ok` |
| `1` | `error`, and for `vkit registry diff`, differences found (`status: ok`, `result.changed: true`) |
| `2` | A denied request (`request_denied`), or a `vkit registry diff` error |
| `3` | `needs_input` |
| `130` | Interrupted |

Questions vkit would otherwise ask become `needs_input`, naming the flag that answers
them:

| Command | `error.code` | Rerun with |
| --- | --- | --- |
| `vkit reset` | `confirmation_required` | `--yes` |
| `vkit grant:revoke`, `vkit policy revoke`, `vkit agents tokens revoke` | `confirmation_required` | `--yes` (or `--force`) |
| `vkit registry export` over an existing file | `overwrite_confirmation_required` | `--force` (or `--yes`) |
| `vkit policy pack upgrade` with missing pack files | `confirmation_required` | `--restore` (or `--yes`) |
| `vkit approval:deny` without `--reason` | `reason_required` | `--reason "..."` |
| `vkit datasource add` without a password source | `secret_required` | `--password-prompt` (in the person's terminal) or `--password-env VAR` |
| `vkit login` | `endpoint_required`, `email_required`, `secret_required` | `--endpoint`, `--email`, `--password-prompt` |
| `vkit login --setup` | `setup_approval_required` | `vkit login --setup --wait` once it's approved |
| `vkit agents tokens create` | `secret_file_required` | `--secret-file PATH` |

Some examples:

```bash
vkit whoami --format json | jq -r '.result.organization'
vkit request --aql "$QUERY" --format json | jq -r '.result.grant_ref'
vkit fetch --grant "$GRANT" --format json | jq '.result.rows'
vkit requests list --state pending --format json | jq '.result.requests[] | {id, dataset, state}'
vkit agents tokens list --format json | jq -r '.result.tokens[] | select(.revoked_at == null) | .name'
vkit registry diff --format json | jq '.result.changed'
vkit policy deploy --format json | jq -r '.result.warnings[].message'
```

To act on an error, read `error.code`, not the message:

```bash
out=$(vkit policy bundle --format json); code=$?
[ "$code" -ne 0 ] && echo "$out" | jq -r '.error.code + ": " + .error.fix'
```

---

## Error codes

`error.code` is stable; the message can change. Each envelope's `error.fix` says what
to do, often with the exact command.

| `error.code` | When | What to do |
| --- | --- | --- |
| `not_logged_in` | No saved login, and no `VKIT_ENDPOINT`/`VKIT_TOKEN`. | Run `vkit login`, or set both variables. |
| `session_expired` | The login has expired (logins last an hour), or an agent token has. | Sign in again, or use a token that's still valid. |
| `token_revoked` | The login was logged out elsewhere, or the token was revoked. | Sign in again, or use a token that's still valid. |
| `not_authenticated` | The control plane refused the credentials for another reason. | Sign in again. |
| `control_plane_unreachable` | Connection refused, DNS, timeout or TLS failure. | Check the control plane is running and the endpoint is right (`vkit whoami`, or `VKIT_ENDPOINT`). |
| `endpoint_required` | No control plane URL to log in to. | Rerun with `--endpoint <url>`. |
| `email_required` | A password login under `--format json` without `--email`. | Rerun with `--email`. |
| `secret_required` | A password or token is needed and wasn't given, or `--password-prompt` has no terminal. | Have the person run the command in their own terminal with `--password-prompt`, or use `--password-env`. |
| `secret_env_missing` | The variable `--password-env` names isn't set or is empty. | Set it, or use `--password-prompt`. |
| `conflicting_password_sources` | More than one of `--password-prompt` and `--password-env`. | Use one. |
| `secret_file_required` | `agents tokens create --format json` without `--secret-file`. | Rerun with `--secret-file PATH`. |
| `secret_file_exists` | The `--secret-file` path already exists. | Pass a path that doesn't exist yet. |
| `setup_approval_required` | A setup session is waiting for an admin. | Approve it in the console, then `vkit login --setup --wait`. |
| `setup_session_not_started` | `vkit login --setup --wait` with no session requested. | Run `vkit login --setup` first. |
| `setup_session_denied` | An admin denied the setup session. | Start another with `vkit login --setup` if that was a mistake. |
| `setup_session_expired` | The code wasn't approved within 10 minutes, or the 30-minute session has ended. | Run `vkit login --setup` again. |
| `setup_session_revoked` | The setup session was revoked, in the console or by `vkit logout`. | Start another with `vkit login --setup`. |
| `setup_session_invalid` | The control plane doesn't know this setup token, or its code was already used. | Run `vkit login --setup` again. |
| `setup_approver_not_admin` | The admin who approved the setup session is no longer an admin. | Have a current admin approve a new session. |
| `setup_poll_failed` | The control plane answered the setup poll unexpectedly. | Rerun `vkit login --setup --wait`. |
| `setup_scope_forbidden` | A setup session can't do this, or the org is already set up and the session wasn't allowed to replace it. | Sign in as yourself with `vkit login`, or approve a new session with "Allow this session to replace existing policies and datasources" ticked. |
| `admin_required` | Datasource commands need an admin. | Ask an admin. |
| `forbidden` | Your role isn't allowed to do this. | Ask an admin. |
| `not_found` | No such name or ID. | List what exists first. |
| `invalid_arguments` | A missing or unknown option or command. | `vkit help <command>`. |
| `invalid_config`, `invalid_json`, `aql_required` | `--config` or the AQL isn't valid JSON, or no AQL was given. | Pass valid JSON. |
| `datasource_exists` | `datasource add` with an existing id and other settings. | Use a new id, or the settings it has (`vkit datasource get <id>`). |
| `datasource_unreachable` | `vkit scan`: the database's host or port can't be reached. | Check the host and port in its config, and that VaultKit can reach it. |
| `datasource_connection_failed` | `vkit scan`: the database refused the connection. A wrong password looks like this, and so does any other connection failure. | Check the username and password, and that the database accepts VaultKit's connections. |
| `datasource_permission_denied` | `vkit scan`: the role can't read the schema. | Grant it read access to the tables. |
| `datasource_query_failed`, `datasource_busy`, `query_runtime_unavailable` | `vkit scan`: other failures on the way to the database. | Read the message; for `query_runtime_unavailable`, check funl. |
| `not_a_project` | Not run in, or pointed at with `--dir`, a folder made by `vkit init`. | Run `vkit init`, or pass `--dir`. |
| `registry_not_found` | `vkit registry diff` with no `datasets/registry.yaml`. | `vkit registry export`. |
| `overwrite_confirmation_required` | `vkit registry export` over an existing file, under `--format json`. | Check `vkit registry diff`, then rerun with `--force` if replacing it is fine. |
| `confirmation_required`, `reason_required` | A step that needs confirmation or a reason, under `--format json`. | Ask the person, then rerun with the flag in `error.fix`. |
| `bundle_not_committed` | Policies, registry or `.vkit/packs.yaml` have uncommitted changes. | Commit them (`error.fix` has the commands). |
| `bundle_not_versioned` | No git repository or no commit, and no `--version`. | `git init` and commit, or pass `--version`. |
| `bundle_invalid`, `policy_invalid` | The bundle or a policy fails validation. | Fix what the message names. |
| `bundle_version_exists` | This version was already uploaded with other contents. | Commit your changes and bundle again. |
| `bundle_checksum_exists` | The same policies were already uploaded under another version. | Nothing new to deploy; an admin can activate that version in the console. |
| `unknown_pack` | No such policy pack. | Use one in `result.available_packs`. |
| `pack_dependency_missing`, `pack_upgrade_failed` | A pack needs another pack first, or an upgrade failed. | Add the dependency, or read the message. |
| `request_denied` | A policy denied the request (exit 2). | Ask for less, or ask an admin about the policy. |
| `doctor_checks_failed` | A `vkit doctor` check failed. | Follow each failed check's `fix`. |
| `interrupted` | Ctrl-C (exit 130). | Rerun. |
| `command_failed` | Anything else. | Read `error.message`. |

Other control plane errors keep their HTTP status's code, or the control plane's own
code when it sends one, as `api_error` with `http_status` otherwise.

---

## Scripting and CI

In CI, use [credentials from the environment](#credentials-from-the-environment)
rather than `vkit login`, which needs an interactive terminal for passwords. For
example, in GitHub Actions:

{% raw %}
```yaml
- name: Deploy policies
  env:
    VKIT_ENDPOINT: https://vaultkit.example.com
    VKIT_TOKEN: ${{ secrets.VKIT_TOKEN }}
  run: |
    gem install vaultkit
    vkit policy bundle
    vkit policy deploy
```
{% endraw %}

A clean checkout bundles as before. A job that changes files under the policies or
registry folders before bundling now gets `bundle_not_committed`: commit first, or
pass `--allow-dirty`.

Exit codes are in [JSON output](#json-output), and are the same without
`--format json`. `vkit registry diff` exits `1` when the files differ, which suits
drift checks in CI.

---

## Troubleshooting

| Message | Cause |
| --- | --- |
| `Not logged in. Run vkit login.` | No saved login, and no `VKIT_ENDPOINT`/`VKIT_TOKEN`. |
| `VKIT_ENDPOINT is set but VKIT_TOKEN isn't` | Set both, or unset the one that's set. |
| `API 401: …` | The token expired or was revoked. Sign in again or issue a new token. |
| `Code: setup_scope_forbidden` | A setup session can't do this. See [Setup sessions](#setup-sessions). |
| `Code: bundle_not_committed` | Commit your policy changes, then bundle. |
| `❌ DENIED` | A policy denied the request; the reason says why. |
| `Policy bundle has changed since grant issuance` | A new bundle was deployed. Ask again for a new grant. |

`vkit doctor` checks a whole setup in one go. Set `VKIT_DEBUG=true` to print a Ruby
backtrace for unexpected errors. API calls time out after 15 seconds. The control
plane's health check is `GET /up`.
