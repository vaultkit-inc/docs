# VaultKit CLI (`vkit`)

**Applies to:** vkit 1.2.0 with VaultKit 0.9.0-alpha

`vkit` is the command-line client for VaultKit. People use it to request governed
data and approve requests; admins use it to register datasources, scan schemas, and
build and deploy policy bundles from git.

## Contents

- [Install](#install)
- [Sign in](#sign-in)
- [Credentials from the environment](#credentials-from-the-environment)
- [Requesting data](#requesting-data)
- [Approvals](#approvals)
- [Grants](#grants)
- [Agent tokens](#agent-tokens)
- [Datasources](#datasources)
- [Schema scans and the registry](#schema-scans-and-the-registry)
- [Policy projects and bundles](#policy-projects-and-bundles)
- [Policy packs](#policy-packs)
- [Scripting and CI](#scripting-and-ci)
- [Troubleshooting](#troubleshooting)

---

## Install

Requires Ruby 3.0 or later and a running VaultKit control plane.

```bash
gem install vaultkit
vkit --version      # vkit 1.2.0
vkit help           # every command; vkit help <command> for one
```

---

## Sign in

```bash
vkit login [--endpoint URL] [--email EMAIL]
```

The endpoint comes from `--endpoint`, then `VKIT_ENDPOINT`, then the last endpoint
you used, then a prompt. The control plane decides how you sign in:

- **SSO:** `vkit` opens your browser and waits for you to finish signing in.
- **Email and password:** it asks for your email (unless you passed `--email`) and
  your password. The password is never accepted as an option.
- **API token:** it reads `VAULTKIT_TOKEN`, or asks for a token.

| Command | What it does |
| --- | --- |
| `vkit whoami` | Shows who you're signed in as. If the control plane can't be reached, it shows the cached identity with a warning. |
| `vkit logout` | Signs out of the saved login (the token is deleted; the endpoint is kept). |
| `vkit reset` | Asks for confirmation, then deletes every stored credential and setting. |

Credentials are stored in the macOS Keychain, in the Secret Service on Linux when
`secret-tool` is installed, or otherwise in `~/.vkit/credentials.json` (mode 0600).

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

With `--format json`, the response is printed as JSON (a denial still exits 2).

Your role, clearance, region and tenant come from your account or agent token, and
the environment and dataset region from the datasource. A request can't set them:
the older `--env`, `--requester_region`, `--dataset_region` and
`--requester_clearance` options are still accepted, but the control plane ignores
them.

### `vkit requests list`

```bash
vkit requests list [--state all|pending|approved|denied] [--format table|json]
```

Your past requests, newest first.

---

## Approvals

When a policy requires approval, someone in each named approval group must sign off.
Admins and users who can approve any request can sign too. Nobody can approve their
own request, and each group's sign-off has to come from a different person. You can
also approve in the console, under **Approvals**.

| Command | What it does |
| --- | --- |
| `vkit approval:list [--state pending\|approved\|denied\|all]` | Lists approvals (default: pending), with the requester, fields, required and outstanding groups, and the reason. |
| `vkit approval:approve ID [--ttl SECONDS]` | Signs off for one group you belong to. The grant is issued once every group has signed. `--ttl` can shorten the grant, never extend it past the policy's TTL. |
| `vkit approval:deny ID [--reason TEXT]` | Denies the whole request. Asks for a reason if you don't give one. |
| `vkit approvals:watch [--interval 3] [--since 10m] [--format table\|json] [--pretty]` | Watches for new pending approvals. |

`ID` is the number `approval:list` shows. Approving fails if the policy bundle
changed while the request waited; the requester then asks again.

---

## Grants

A grant is VaultKit's permission to run one exact query, for a limited time.

| Command | What it does |
| --- | --- |
| `vkit fetch --grant REF [--format json\|table]` | Runs the granted query and prints the rows (default: JSON). |
| `vkit grant:revoke --grant REF [--reason TEXT] [--force]` | Admin only. Revokes a grant before it expires. Asks for confirmation unless `--force`. |

A grant stops working when it expires, is revoked, or the active policy bundle
changes. Asking the same question again within the grant's lifetime reuses it. See
[Approvals and grants](approvals_and_grants.md).

---

## Agent tokens

Agents authenticate with agent tokens rather than a user login. Only admins can
issue and revoke them.

### `vkit agents tokens create`

```bash
vkit agents tokens create --name NAME [--expires-in 24h] [--tenant ID] \
  [--clearance low|medium|high] [--region EU|US|CA|APAC]
```

- `--expires-in` takes hours or days: `1h`, `24h`, `30d`.
- `--tenant` confines the token to one tenant's rows on datasets marked with
  `tenant_column`. See [Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets).
- `--clearance` and `--region` are what policies see as `requester_clearance` and
  `requester_region`.
- Tenant, clearance and region are fixed when the token is issued; the agent can't
  change them. Every agent token has the role `agent` (the `--role` option is
  ignored by the control plane).
- The secret is shown once.

For an assistant that serves several customers, issue a short-lived token per
customer session:

```bash
vkit agents tokens create --name support-assistant --tenant org_42 --expires-in 1h
```

### `vkit agents tokens list` and `revoke`

```bash
vkit agents tokens list [--format table|json]
vkit agents tokens revoke --token ID_OR_PREFIX [--force]
```

`revoke` asks you to type `revoke` to confirm, unless `--force`.

---

## Datasources

Admin only. VaultKit stores the database credential (encrypted) and runs queries
itself, so people and agents never hold it. PostgreSQL is supported today.

### `vkit datasource add`

Every option is required.

```bash
vkit datasource add \
  --id prod_db \
  --engine postgres \
  --username vaultkit_reader \
  --password "$DB_PASSWORD" \
  --config '{"host": "db.internal", "port": 5432, "database": "app"}' \
  --region US \
  --environment production
```

- `--region`: `EU`, `US`, `CA` or `APAC`. Policies see it as `dataset_region`.
- `--environment`: `production`, `staging`, `development` or `sandbox`. Policies see
  it as `environment`.
- Use a read-only database role: VaultKit only reads.

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
vkit scan DATASOURCE [--mode diff_only|apply]
```

`diff_only` (the default) shows what changed since the last scan: added (`+`),
removed (`-`) and changed (`~`) fields. `apply` saves the scan to the control
plane's registry.

The scan labels each column from its name, type and table:

- **Sensitivity:** `high`, `medium`, `low` or `unknown`.
- **Category:** `pii`, `financial`, `credential`, `internal` or `unknown`.

It reads names and types only, never data. `unknown` means a column that could hold
anything (a `payload` jsonb, a `notes` text) that it couldn't place: review it.

A rescan keeps labels you've reviewed (from a deployed bundle or a manual edit). Where
the classifier disagrees, it lists a suggestion instead of changing the label:

```
  api_keys                 ✓
    ? key_value  low/internal → high/credential (suggested)
```

See [Schema discovery](schema_discovery.md).

### `vkit registry export` and `diff`

```bash
vkit registry export [--dir .] [--out PATH] [--force]
vkit registry diff [--dir .] [--format human|json]
```

`export` writes the control plane's registry to `datasets/registry.yaml`, offering to
overwrite, show a diff or stop if the file exists. `diff` compares the two and exits 0
when they match, 1 when they differ and 2 on error.

---

## Policy projects and bundles

Policies are YAML files in git. You compile them, with the registry, into a bundle and
deploy it; the bundle version is the git commit. See the
[Policy Guide](policy_guide.md).

### `vkit init`

```bash
vkit init [--dir .] [--with starter,ai_safety]
```

Creates `config/policies/`, `datasets/registry.yaml`, `dist/` and `.vkit/`, and
installs the packs you name (plus their dependencies). With no `--with`, it installs
`starter`.

### `vkit policy bundle`

```bash
vkit policy bundle [--policies_dir config/policies] [--registry_dir datasets] \
  [--out dist/policy_bundle.json] [--org ORG]
```

Needs a login, and commits in git: the bundle version is `git rev-parse HEAD`.
`--org` must match the org you're signed in to.

The bundle is refused, with every problem listed, when a policy can never match or a
dataset would leak across tenants:

- a key the engine ignores, such as `match.fields.category` (use `contains`);
- a tag or sensitivity no field has and the classifier can't produce;
- a dataset that isn't in `datasets/registry.yaml`;
- an invalid environment, region or clearance value, or `time` under top-level
  `context` (it works under `match.context`);
- a mask policy with no fields to mask;
- a `tenant_column` that isn't one of its dataset's fields, or a dataset that has
  another dataset's tenant column but isn't marked (`tenant_column: false` declares a
  table every tenant may read).

If the problems are in pack policies, `vkit policy pack upgrade` replaces them.

### `vkit policy validate`, `deploy` and `revoke`

```bash
vkit policy validate [--bundle dist/policy_bundle.json] [--schema PATH]
vkit policy deploy [--bundle dist/policy_bundle.json] [--no-activate]
vkit policy revoke --bundle_version VERSION [--reason TEXT] [--force]
```

- `deploy` uploads the bundle and makes it active; `--no-activate` only uploads it.
  A bundle version can only be deployed once, so commit a change before redeploying.
- `revoke` revokes a bundle version: grants issued under it stop working.

---

## Policy packs

Packs are ready-made, versioned policies. See [Policy Packs](pack_system.md).

| Command | What it does |
| --- | --- |
| `vkit policy pack list` | Shows available packs, and which are installed or have an upgrade. |
| `vkit policy pack info NAME` | Shows a pack's version, dependencies and policies. |
| `vkit policy pack add NAME [--force]` | Installs a pack into `config/policies/`. |
| `vkit policy pack remove NAME [--force]` | Removes a pack's files (only the ones it installed). |
| `vkit policy pack upgrade NAME [--force]` | Replaces an installed pack with the shipped version. |

Every pack command takes `--dir` (default `.`). Commit the changes, then bundle and
deploy.

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

Exit codes: `0` success; `1` an error or invalid usage; `2` a denied request, or a
registry diff error; `130` interrupted. `vkit registry diff` exits `1` when the files
differ, which suits drift checks in CI.

`vkit fetch --format json` prints a status line and metadata before the rows, so
strip those before piping to `jq`.

---

## Troubleshooting

| Message | Cause |
| --- | --- |
| `Not logged in. Run vkit login.` | No saved login, and no `VKIT_ENDPOINT`/`VKIT_TOKEN`. |
| `VKIT_ENDPOINT is set but VKIT_TOKEN isn't` | Set both, or unset the one that's set. |
| `API 401: …` | The token expired or was revoked. Sign in again or issue a new token. |
| `❌ DENIED` | A policy denied the request; the reason says why. |
| `Policy bundle has changed since grant issuance` | A new bundle was deployed. Ask again for a new grant. |

Set `VKIT_DEBUG=true` to print a Ruby backtrace for unexpected errors. API calls
time out after 15 seconds. The control plane's health check is `GET /up`.
