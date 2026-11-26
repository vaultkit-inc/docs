# Upgrading to vkit 2.0.0

vkit 2.0.0 can be driven by a coding agent: every command takes `--format json`,
secrets never go on the command line, and setup steps are safe to rerun. It adds
[setup sessions](setup_sessions.md) (`vkit login --setup`) and
[`vkit doctor`](cli.md#checking-a-setup-vkit-doctor), and ties every policy bundle to
the git commit it was built from.

```bash
gem install vaultkit
vkit --version      # vkit 2.0.0
```

These changes can break scripts and existing projects. Go through each one.

## Contents

- [`--format json` is wrapped in an envelope](#--format-json-is-wrapped-in-an-envelope)
- [`vkit policy bundle` refuses uncommitted changes](#vkit-policy-bundle-refuses-uncommitted-changes)
- [Commit `.vkit/packs.yaml`](#commit-vkitpacksyaml)
- [`--password` is deprecated](#--password-is-deprecated)
- [Reserved approval group names](#reserved-approval-group-names)
- [Other changes in behaviour](#other-changes-in-behaviour)
- [Control plane version](#control-plane-version)

---

## `--format json` is wrapped in an envelope

Six commands already had `--format json`. Their output is now one envelope, with
the old output under `result`:

```json
{"status": "ok", "result": { }, "next": [ ], "error": {"code": "...", "message": "...", "fix": "..."}}
```

Logs and the human output go to stderr, so stdout is only the JSON. See
[JSON output](cli.md#json-output).

| Command | vkit 1.x printed | In 2.0.0, read |
| --- | --- | --- |
| `vkit request --format json` | the control plane's answer | `.result` (`.result.status`, `.result.grant_ref`, …) |
| `vkit requests list --format json` | an array of requests | `.result.requests` |
| `vkit fetch --format json` | a status line, the metadata, then the rows | `.result.rows` (and `.result.meta`, `.result.grant`) |
| `vkit agents tokens list --format json` | an array of tokens | `.result.tokens` |
| `vkit registry diff --format json` | `{datasets, changed}` | `.result.datasets`, `.result.changed` |
| `vkit policy deploy --format json` | `{status: "deployed", org, bundle_version, state, checksum, warnings}` | `.result` (`.status` is now the envelope's `ok`) |

For example:

```bash
# vkit 1.x
vkit requests list --format json | jq '.[] | .id'
vkit agents tokens list --format json | jq -r '.[].name'

# vkit 2.0.0
vkit requests list --format json | jq '.result.requests[] | .id'
vkit agents tokens list --format json | jq -r '.result.tokens[].name'
```

Exit codes are unchanged: a denied `vkit request` still exits 2, and
`vkit registry diff` still exits 0 (no differences), 1 (differences) or 2 (error).
`vkit fetch` without `--format` prints what it always did. `vkit approvals:watch
--format json` is still a stream of one object per approval.

Under `--format json`, a command that would ask a question now returns
`status: needs_input` and exits 3 instead of prompting. Scripts that piped answers
into prompts need the matching flag (`--yes`, `--force`, `--reason`, `--restore`).
See the table in [JSON output](cli.md#json-output).

---

## `vkit policy bundle` refuses uncommitted changes

A bundle's version is the git commit. vkit 1.x labelled whatever was on disk with
`HEAD`, even when the files differed from that commit. vkit 2.0.0 refuses
(`bundle_not_committed`) when anything under the policies folder, the registry folder
or `.vkit/packs.yaml` isn't committed, and lists the files and the git commands:

```
❌ These files aren't committed, so the bundle's version wouldn't match its policies:
   datasets/registry.yaml
   Code: bundle_not_committed
   Fix:
     git add -A -- config/policies datasets .vkit/packs.yaml
     git commit -m "Update VaultKit policies"
```

**What to do:** commit before you bundle. Changes elsewhere in the repository don't
count. A CI job that bundles a clean checkout is unaffected.

- **`--allow-dirty`** bundles anyway and records `"dirty": true` in the bundle. Its
  version is still the commit, so don't deploy one you mean to keep.
- **`--version`** now only names the bundle. It used to replace the commit SHA too, so
  a label like `v1.2` failed the bundle schema; now `source.commit_sha` always comes
  from git. It doesn't skip the commit check.
- **No git repository** (or no commit) and no `--version` is now
  `bundle_not_versioned`, instead of a bundle with an empty version.
- **`vkit init`** runs `git init` when the folder isn't in a repository (`--no-git`
  skips it).
- **Rebuilding the same commit gives the same checksum,** so deploying it again
  returns `already_deployed` instead of failing. A commit first deployed by vkit 1.x
  gets `bundle_version_exists` once when rebuilt: make a new commit, then bundle and
  deploy.

---

## Commit `.vkit/packs.yaml`

`.vkit/packs.yaml` records each installed pack's version and files, and
`vkit policy bundle` uses it to check that no pack policy is missing. Projects made by
vkit 1.x ignore all of `.vkit/` in `.gitignore`, so a clone had no `packs.yaml`, and
the check was silently skipped.

In an existing project, the first bundle with vkit 2.0.0 fails with
`bundle_not_committed`:

```
❌ These files aren't committed, so the bundle's version wouldn't match its policies:
   .vkit/packs.yaml (ignored by .gitignore)
   Code: bundle_not_committed
   Fix:
     git add -A -- config/policies datasets .vkit/packs.yaml
     git add -f -- .vkit/packs.yaml
     and in .gitignore, replace .vkit/ with two lines: .vkit/* and !.vkit/packs.yaml
     git commit -m "Update VaultKit policies"
```

**What to do:** in `.gitignore`, replace `.vkit/` with:

```
.vkit/*
!.vkit/packs.yaml
```

Then commit `.vkit/packs.yaml`:

```bash
git add .gitignore .vkit/packs.yaml
git commit -m "Track .vkit/packs.yaml"
```

`vkit init` writes the new `.gitignore` for new projects. It doesn't change an
existing one.

---

## `--password` is deprecated

`vkit datasource add --password` still works, but it prints a deprecation warning: a
password on the command line ends up in shell history and agent transcripts. Use one
of:

```bash
vkit datasource add ... --password-prompt          # type it (/dev/tty, echo off)
vkit datasource add ... --password-env DB_PASSWORD # read it from the environment
```

Without any of them, `datasource add` now fails (`secret_required`) instead of
running. `vkit login` has `--password-prompt` too.

---

## Reserved approval group names

`admin`, `member`, `approver` and `agent` are built-in roles, not approval groups.
`vkit policy bundle` refuses a policy that names one as `approver_role` or in
`approver_roles`, and so does the control plane when you deploy.

**What to do:** if a policy names one, use a real group instead (for example
`data_owner`), and add people to it in the console. Org admins can already approve
any request, so a policy doesn't need to name `admin`. No shipped pack uses these
names.

On the control plane, people can't be put in these groups any more, and upgrading
removes them from anyone who was. See
[Approvals and grants](approvals_and_grants.md#approvals).

---

## Other changes in behaviour

These don't break anything, but you may notice them:

- **Reruns succeed.** `vkit init` in a project, `vkit datasource add` with an existing
  id and the same settings, `vkit policy pack add` for an installed pack, and
  `vkit policy deploy` of the active bundle all change nothing and say so
  (`already_existed`, `already_deployed`), instead of failing. A `vkit scan --mode
  apply` that finds nothing new returns `unchanged: true`.
- **`vkit registry export --force`** (or `--yes`) overwrites without asking.
- **`--yes`** works on `reset`, `grant:revoke`, `policy revoke`,
  `agents tokens revoke`, `registry export` and `policy pack upgrade`.
- **`vkit policy deploy` warns** about approval groups with no members: org admins
  approve their requests until someone is added.
- **`vkit policy pack info`** shows the pack's description.
- **`vkit agents tokens create --secret-file PATH`** writes the token to a file
  instead of printing it.
- **`vkit logout`** ends a setup session on the control plane.
- **vkit sends a `User-Agent`**: `vkit/2.0.0 (<os> <cpu>; ruby <version>)`.
- **The bundle checksum** is the same each time a commit is rebuilt.

---

## Control plane version

These need VaultKit 0.13.0-alpha, the control plane release that ships with vkit
2.0.0. With a 0.12.0-alpha control plane, vkit 2.0.0 falls back where it can:

| Feature | Needs | On 0.12.0-alpha |
| --- | --- | --- |
| Setup sessions (`vkit login --setup`) | 0.13.0-alpha | Not available. |
| Creating agent tokens in the console | 0.13.0-alpha | Use `vkit agents tokens create`. |
| Deploying an `--allow-dirty` bundle | 0.13.0-alpha | Refused with `422`. Commit, and bundle without it. |
| Org admins approving for groups with no members, and deploy warnings from the control plane | 0.13.0-alpha | vkit works out the warnings itself from the group list. `vkit doctor`'s approval routing check reports that the control plane is too old. |
| Reruns answered by the control plane (`already_existed` on deploy and datasource add) | 0.13.0-alpha | vkit works it out from the active bundle and the datasource list. |

Upgrade the control plane to 0.13.0-alpha first, then vkit. See
[Updating VaultKit](deployment.md#updating-vaultkit).
