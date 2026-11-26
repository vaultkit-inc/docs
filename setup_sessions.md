# Setup Sessions

**Applies to:** vkit 2.0.0, with a control plane that has setup sessions (see
[Upgrading to vkit 2.0.0](upgrading_vkit_2.md#control-plane-version))

A setup session lets a coding agent set VaultKit up for you without ever holding your
own login. It works like `gh auth login`: the agent asks for a session, you approve it
in the console, and the agent gets a token that can only do setup, for 30 minutes.

[Set Up VaultKit with a Coding Agent](agent_setup.md) uses one. This page explains what
you're approving.

---

## How it works

1. **The agent asks for one** with `vkit login --setup`. vkit shows a code, such as
   `BJCQ-6006`, and a link to the console's approval page.
2. **You approve it** in the console, signed in as an admin of your organization. The
   code must be approved within 10 minutes.
3. **The agent gets a setup token.** It lasts 30 minutes, acts for you, and can only
   reach the setup endpoints listed below. vkit saves it like a normal login.
4. **The agent runs setup**: adds the datasource, scans it, deploys policies.
5. **The session ends** when it expires, when the agent runs `vkit logout`, or when
   you revoke it.

Everything the token does is recorded in Activity as "agent setup on behalf of" you.

See [Setup sessions](cli.md#setup-sessions) in the CLI reference for the commands and
their `--format json` output.

---

## The approval page

The link opens `/setup/approve` in the console. Only admins can approve; a member is
told an admin has to. If the link has no code, or the wrong one, the page asks you to
type the code from your terminal.

The page shows:

- **The code**, with "Does this code match your terminal?" Approve only if it's exactly
  the code your terminal shows. If it isn't, deny it: someone else may have sent you
  the link.
- **What the agent will be able to do:** "A coding agent wants to set up VaultKit for
  Acme. It will be able to: connect a datasource, scan and classify its schema, apply a
  policy pack and deploy policies. It will not be able to query data, approve
  requests, create tokens or manage users. Expires in 30 minutes."
- **Where the request came from:** when, from which IP address, and the client's
  `User-Agent`, such as "Requested 2 minutes ago from 203.0.113.7,
  vkit/2.0.0 (macOS arm64; ruby 3.3.8)". The `User-Agent` is whatever the client sent,
  so treat it as a hint, not proof.
- **"Only approve if you started this in your own terminal just now."** Anyone can
  start a setup session and send an admin the link, so this is the check that matters.
- **What's already set up,** if anything: the active policy bundle and the
  datasources, with the box **Allow this session to replace existing policies and
  datasources** (see below).
- **Approve** and **Deny.**

---

## What a setup token can and can't do

It can do what `vkit` needs for setup, and nothing else:

| It can | vkit commands |
| --- | --- |
| Read who it is | `vkit whoami` |
| List, read and add datasources | `vkit datasource list`, `get`, `add` |
| Scan a datasource, in either mode | `vkit scan` |
| Read the registry | `vkit registry export`, `vkit registry diff` |
| Upload and activate policy bundles | `vkit policy deploy` |
| Read approval groups and setup status | `vkit policy deploy`'s warnings, `vkit doctor` |
| End itself | `vkit logout` |

`vkit policy bundle`, `vkit policy validate` and `vkit init` run locally and need no
endpoint.

It **can't** query data, approve or deny requests, fetch or revoke grants, create,
list or revoke agent tokens, manage users or settings, read Activity, or revoke or
roll back policy bundles. Each of those returns `403 setup_scope_forbidden`. So once
setup is done, the agent token for the agent itself is created by you, in the console
or with your own `vkit` login (see [Creating agent tokens](mcp_usage.md#creating-agent-tokens)).

The token also stops working if the admin who approved it is no longer an active admin
of the organization (`setup_approver_not_admin`).

---

## An organization that's already set up

A setup token sets an organization up; by default it can't change one that already is.
Uploading or activating a bundle, adding a datasource and `vkit scan --mode apply` work
only while the organization has no active policy bundle, or when the active bundle was
deployed by this same setup session. Otherwise they return `403 setup_scope_forbidden`:

```
API 403: This organization is already set up: policy bundle 4b7657fa… is active and was not
deployed by this setup session. An admin must deploy changes to an organization that is
already set up; a setup token can only set up a new one, or redo its own setup.
```

**Steps that change nothing always work,** so rerunning setup with a new session is
safe:

- adding a datasource that exists with the same settings (`already_existed: true`);
- deploying the bundle that's already active (`already_deployed: true`);
- a `scan --mode apply` that finds nothing new (`unchanged: true`).

**To let a new session change an existing setup,** for example to finish one whose
first session expired, tick **Allow this session to replace existing policies and
datasources** when you approve it. The box appears only when there's something to
replace, it's unticked by default, and your choice is recorded with the approval.

---

## Ending a session

- **`vkit logout`** ends it on the control plane, not just locally:
  `🔒 Setup session ended on the control plane`.
- **In the console,** live sessions are listed at the top of the **Agents** page under
  **Setup sessions**: waiting for approval, approved but not collected, or token in use.
  **Revoke session** asks for a reason. A revoked token stops working on its next
  request (`setup_session_revoked`); what it already set up stays.
- **It expires** 30 minutes after approval. An unapproved code expires after 10
  minutes.

## When it expires part way

Commands return `setup_session_expired`. Run `vkit login --setup` again and approve
the new session. Every setup step is safe to rerun. If the first session already
deployed a bundle or added a datasource, the new one can rerun those steps unchanged;
to change them, tick "Allow this session to replace existing policies and datasources"
when you approve it.

---

## In the audit log

The principal is the setup session, shown in Activity as "agent setup on behalf of
ada@example.com":

- `setup_session.requested` (when an admin first opens the link, with the requester's
  IP and User-Agent), `setup_session.approved` (with what the organization already had,
  and whether replacing was allowed), `setup_session.denied`,
  `setup_session.token_issued`, `setup_session.revoked` and `setup_session.expired`;
- `setup_session.action` for every request made with the token, and
  `setup_session.action_refused` for each one refused.

The events of the things it does, such as `datasource.created` and
`policy_bundle.created`, keep you as the principal and add the setup session to their
details.

Starting a session is limited to 10 per IP address per minute, and polling to 60;
over the limit is `429 rate_limited`.
