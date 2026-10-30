# VaultKit Deployment Guide

This document explains how to deploy VaultKit in production and production-like environments.
VaultKit is designed to be **easy to deploy for design partners**, while remaining flexible enough for advanced infrastructure setups.

**Deploy:** [github.com/vaultkit-inc/deploy](https://github.com/vaultkit-inc/deploy)

VaultKit consists of four main components:

* **Control Plane** — Governs every data request: authentication, policy evaluation, approvals, and audit logging
* **FUNL Runtime** — Data plane and execution engine: translates queries into native SQL, applies field-level masking, and executes against your datasources
* **Console** — React web UI with a built-in nginx that serves the frontend and proxies API traffic to the Control Plane
* **Postgres** — Metadata and application state

---

## How It Works

VaultKit uses a **single entry point** for both the frontend and backend.

The Console container runs nginx internally. nginx serves the React app and automatically proxies all API and auth requests to the Control Plane over Docker's internal network. This means:

* You access **one URL** for everything
* The Control Plane is never exposed directly to the outside world
* No external reverse proxy is required for evaluation

```
Browser → Console:5173 → nginx (inside console container)
                              ├── /api/*         → Control Plane:3000
                              ├── /auth/oidc/*   → Control Plane:3000
                              ├── /auth/cli/*    → Control Plane:3000
                              └── /*             → React app
```

---

## Deployment Models

VaultKit supports two deployment models out of the box.

### 1. Single-Domain Deployment (Default)

Best for:

* Design partners
* Small teams
* Fast evaluation
* Minimal infrastructure

One URL serves everything — nginx inside the Console handles routing. A reverse proxy only needs to handle TLS.

This is the **recommended starting point**.

---

### 2. Multi-Domain Deployment (Advanced)

Best for:

* Larger organizations
* Existing infra standards
* Strict network separation

Example:

```
https://app.acme.com   → Console (frontend + nginx proxy)
https://api.acme.com   → Control Plane (backend)
```

**No VaultKit images need to be rebuilt** — only routing and `.env` changes.

---

## Prerequisites

* Docker 24+
* Docker Compose v2
* A Linux host (VM, EC2, bare metal, or local machine)
* Optional: A domain name
* Optional: Cloudflare or Let's Encrypt for TLS (required for custom domain with HTTPS)

---

## Quick Start

### 1. Clone the Deployment Repository

```bash
git clone https://github.com/vaultkit-inc/deploy.git
cd deploy
```

---

### 2. Run the Installer

```bash
./scripts/install.sh
```

This does the following:

* Verifies Docker and Docker Compose are installed and running
* Copies `.env.example` → `.env`
* Selects the latest release and saves it as `VAULTKIT_VERSION` in `.env`
* Generates RSA signing keys (`vkit_priv.pem`, `vkit_pub.pem`)
* Displays your machine's IP address for reference

To install a specific release, use `./scripts/install.sh --version 0.9.2-alpha`.

---

### 3. Configure `.env`

Edit `.env` and set `APP_HOST` and `FRONTEND_BASE_URL` to wherever VaultKit will be accessed from. Pick the option that matches your setup:

---

**Option A — Local machine**

```env
RAILS_ENV=production

APP_HOST=http://localhost:5173
FRONTEND_BASE_URL=http://localhost:5173
FORCE_SSL=false

DATABASE_URL=postgres://vaultkit:<password>@postgres:5432/vaultkit
FUNL_URL=http://funl-runtime:8080
VKIT_PRIVATE_KEY=/secrets/keys/vkit_priv.pem
VKIT_PUBLIC_KEY=/secrets/keys/vkit_pub.pem
```

---

**Option B — Remote VM or EC2 (no domain)**

```env
RAILS_ENV=production

APP_HOST=http://54.123.45.67:5173
FRONTEND_BASE_URL=http://54.123.45.67:5173
FORCE_SSL=false

DATABASE_URL=postgres://vaultkit:<password>@postgres:5432/vaultkit
FUNL_URL=http://funl-runtime:8080
VKIT_PRIVATE_KEY=/secrets/keys/vkit_priv.pem
VKIT_PUBLIC_KEY=/secrets/keys/vkit_pub.pem
```

`FORCE_SSL=false` lets the control plane answer over plain HTTP. Leave it out of any
real deployment: tokens and query results would cross the network unencrypted.

Replace `54.123.45.67` with your server's public IP. To find it:

```bash
curl ifconfig.me
```

---

**Option C — Custom domain**

```env
RAILS_ENV=production

APP_HOST=https://vaultkit.yourdomain.com
FRONTEND_BASE_URL=https://vaultkit.yourdomain.com

DATABASE_URL=postgres://vaultkit:<password>@postgres:5432/vaultkit
FUNL_URL=http://funl-runtime:8080
VKIT_PRIVATE_KEY=/secrets/keys/vkit_priv.pem
VKIT_PUBLIC_KEY=/secrets/keys/vkit_pub.pem
```

> For Option C, point your domain's A record to your server's IP address and put a reverse proxy in front of VaultKit for TLS. See [Running behind a reverse proxy](#running-behind-a-reverse-proxy).

---

> **Note:** Both `APP_HOST` and `FRONTEND_BASE_URL` always point to the same URL because nginx inside the Console unifies frontend and backend behind a single entry point.

---

### 4. Start VaultKit

```bash
docker compose up -d
```

Verify services:

```bash
docker compose ps
```

All four containers should be running:

| Container | Port | Purpose |
|---|---|---|
| vaultkit-console | 5173 | Frontend + nginx proxy — your entry point |
| vaultkit-control-plane | 3000 | Backend API (internal only) |
| vaultkit-funl | 8080 | Query execution engine (internal only) |
| vaultkit-db | — | Postgres (internal only) |

---

### 5. Access VaultKit

Open your browser at whichever URL you configured:

```
# Option A
http://localhost:5173

# Option B
http://54.123.45.67:5173

# Option C
https://vaultkit.yourdomain.com
```

Health check:

```
<your-url>/up
```

---

## Running behind a reverse proxy

Forward all paths to the console on port 5173. Its nginx handles routing, so don't send
anything to ports 3000 or 8080, which are internal.

Caddy gets a Let's Encrypt certificate automatically:

```text
vaultkit.example.com {
    reverse_proxy localhost:5173
}
```

Bind the console to localhost in `.env`:

```bash
CONSOLE_BIND=127.0.0.1
```

The console trusts the incoming `X-Forwarded-Proto`, so if port 5173 is reachable from
the internet, anyone can skip the HTTPS redirect. Close 5173 in your firewall or security
group too. Besides SSH, only ports 80 and 443 need to be open.

If Cloudflare is in front, set its SSL mode to **Full** or **Full (strict)**, not
Flexible. Flexible sends plain HTTP to your server, which causes a redirect loop.

---

## OIDC / SSO Configuration

VaultKit supports OIDC providers such as Okta, Auth0, Azure AD, and others.

For browser-based login the redirect URI **must be**:

```
<your-url>/auth/oidc/callback
```

> Most OIDC providers require HTTPS for redirect URIs. For local evaluation without OIDC, skip this step and use token-based access instead.

---

## Multi-Domain Deployment

If you want separate frontend and backend domains:

### Required Changes

1. Update `.env`:

```env
APP_HOST=https://api.acme.com
FRONTEND_BASE_URL=https://app.acme.com
```

2. Configure your reverse proxy to route:

* `app.acme.com` → Console container (port 5173)
* `api.acme.com` → Control Plane container (port 3000)

The control plane isn't published on the host by default. If your proxy doesn't run on
the Compose network, publish the port in `docker-compose.yml`, on an address only the
proxy can reach:

```yaml
  control-plane:
    ports:
      - "127.0.0.1:3000:3000"
```

VaultKit images remain unchanged.

---

## Using an External Postgres Database

By default, VaultKit runs Postgres inside Docker.

If you already have Postgres:

* Remove the `postgres` service from `docker-compose.yml`
* Point `DATABASE_URL` to your existing database

```env
DATABASE_URL=postgres://vaultkit:<password>@db.acme.internal:5432/vaultkit
```

---

## Secrets & Security Notes

* `.env` **must not** be committed to version control
* Signing keys are generated locally and mounted read-only into containers
* TLS termination should happen **outside** the containers in production
* Only port 5173 needs to be publicly accessible — ports 3000 and 8080 are internal only

---

## Updating VaultKit

Back up the database first. This reads the connection, including the database name,
from `DATABASE_URL`, so it works whatever your database is called:

```bash
docker compose exec -T postgres sh -c 'pg_dump "$DATABASE_URL"' > vaultkit-backup.sql
```

Then move to the new version:

```bash
./scripts/install.sh --version <version>
docker compose pull
docker compose up -d
```

Database migrations run automatically on container startup.

An install stays on its version until you change it, so pulling the deploy repository
never upgrades you. The version is saved as `VAULTKIT_VERSION` in `.env`, and all three
images use it. The installer switches the folder to that release, so the compose file
always matches the images. Keep your own settings in `.env`: it stops if tracked files
have local changes.

The installer refuses a version that doesn't exist or is known to be broken. It also
refuses an older version: migrations don't run backwards, so VaultKit can't downgrade in
place. To go back, restore your backup.

Installs from before `VAULTKIT_VERSION` was in `.env` need to run `./scripts/install.sh`
once after updating the deploy repository. If you edited `docker-compose.yml`, undo that
first.

Always run the `control-plane`, `funl-runtime` and `console` images from the same
release. The control plane and funl depend on each other's behaviour (for example,
join aliases and aggregate masking are split between them), so mixing versions can
produce failing queries or unhelpful errors.

Pin all three images to the same version in `docker-compose.yml`, then pull and
restart. Upgrading across several releases? Apply the notes for each version in
between, newest first below.

### Upgrading to 0.9.2-alpha

Back up the database, update all three image tags in `docker-compose.yml` to
`0.9.2-alpha`, then pull and restart. Upgrading from 0.8.0 or earlier? Read the
0.9.1-alpha and 0.9.0-alpha notes too. vkit 1.2.1 and the Python SDK 0.3.1 work
unchanged.

**Skip 0.9.1-alpha.** Its console breaks every deployment behind a TLS proxy, as
item 1 explains. 0.9.2-alpha includes all of its fixes.

**One database migration** runs when the control plane starts. It adds
`policy_approvals.denial_reason` and moves existing denial reasons there. Denying used
to overwrite the policy's reason for asking for approval, so that reason is gone for
requests denied before this release.

1. **The console works behind a TLS proxy.** The 0.9.1-alpha console replaced
   `X-Forwarded-Proto` with `http`, so the control plane redirected every request to
   HTTPS, forever. Check that your proxy sends `X-Forwarded-Proto: https`. If Cloudflare
   is in front, use an SSL mode that reaches your proxy over HTTPS, or the proxy will
   report `http`.
2. **Plain HTTP needs `FORCE_SSL=false`.** The control plane redirects anything that
   isn't HTTPS, so a trial on `http://<server-ip>:5173` needs that setting in `.env`.
   Leave it out of production.
3. **The control plane is no longer published on port 3000 in `deploy/alpha`.** The
   console proxies to it, so point your reverse proxy at port 5173. Check your own
   `docker-compose.yml`: if port 3000 is still published, remove it unless a multi-domain
   proxy outside Docker needs it. **Anything that calls `:3000` directly needs to use the
   console URL instead.**
4. **Every login now lasts one hour.** Password sign-in lasted two hours, and OIDC and
   CLI logins also lasted two despite being meant to last one. Tokens issued before the
   upgrade keep their original expiry. Agent tokens have their own expiry and aren't
   affected. **Scripts that use a login token through `VKIT_TOKEN` and run for more than
   an hour should use an agent token.**
5. **The console handles expired sessions.** It goes to sign-in with "Your session
   expired. Sign in again to continue." and returns to the same page. Failed requests
   show an error with a retry, not an empty page.
6. **The Approvals page is redesigned.**
   - It has three tabs: Waiting on you, Your requests and History. Requests show why
     the policy held them, what the agent would get, and the exact query.
   - Approve and Deny wait five seconds with Undo before they're sent. If your session
     expires during those seconds, the decision is lost and the request stays waiting.
   - **A deny reason is now optional, and it reaches the requester and their agent**
     through `GET /requests/:id`.
7. **Members have their own Overview,** with what's waiting on them, their active
   grants and their recent requests. New members see how to connect an agent. Admins
   keep the existing Overview.
8. **API changes.**
   - `GET /approvals?state=decided` lists past decisions. Use it instead of
     `state=approved`, which returns an empty list.
   - Deciding a request that's already decided, or approving one queued under an older
     policy bundle, now returns `409`. They used to return `403` and `500`.
   - On a denied approval, `reason` is the policy's reason, and the approver's is in
     `denial_reason`. **Integrations that read the approver's reason from `reason` need
     updating.**

### Upgrading to 0.9.1-alpha

Update all three image tags in `docker-compose.yml` to `0.9.1-alpha`, then pull and
restart. There are no database migrations. Upgrading from 0.8.0 or earlier? Read the
0.9.0-alpha notes too.

Upgrade the clients as well: vkit 1.2.1 verifies TLS certificates when signing in,
and the Python SDK 0.3.1 passes policy denial reasons, including the new ones below,
through to MCP agents. vkit 1.2.0 and SDK 0.3.0 still work.

1. **Filters on masked fields are denied.** Full and partial masking change the
   values a query returns, but filters compared the unmasked column, so a filter could
   confirm or recover a masked value.
   - A request whose `filters` name a field that is fully or partially masked for the
     requester is denied. This includes nested groups and joined tables.
   - `having` is denied on any masked field, hash masking included.
   - Hash-masked fields can still be filtered in `filters`; the comparison is against
     the hash.
   - Grants are rechecked when reused, so an existing grant whose query filters a
     masked field stops working. Request again without that filter.
   - **Agents and scripts that filtered on masked fields now get a denial.** Drop the
     filter, or ask for a policy that doesn't mask the field for that requester.
2. **The console image is a production build served by nginx on port 80.** The
   0.9.0-alpha image ran the Vite development server on port 5173, which the
   `5173:80` port mapping never reached. The new image proxies the API and sign-in
   routes to `control-plane:3000`, so keep the control plane's service name as
   `control-plane`. The console is still at `http://<host>:5173`.

### Upgrading to 0.9.0-alpha

Update all three image tags in `docker-compose.yml` to `0.9.0-alpha`, then pull and
restart. Upgrading from an older release? Read the notes for each version in between
as well; their changes also apply.

**Two database migrations** run when the control plane starts:
- `users.clearance` and `users.region`;
- `agent_tokens.tenant`, `schema_registry_entries.tenant_column`, and an index on
  `audit_logs (organization_id, details->>'tenant')`.

**Upgrade the control plane before `vkit`.** vkit 1.2.0 can write `tenant_column`
into bundles, which a 0.8.0 control plane rejects. Then upgrade to vkit 1.2.0, which
this release needs for the steps below.

1. **Requester clearance and region are set by admins.**
   - Users and agent tokens have a clearance (`low`, `medium` or `high`) and a region
     (`EU`, `US`, `CA` or `APAC`). Set them in the console (Team, then Edit member),
     through the users and agent-token APIs, or with
     `vkit agents tokens create --clearance --region`.
   - Policies match them as `requester_clearance` and `requester_region`. Values in a
     request are ignored: `vkit request --requester-region` and
     `--requester-clearance`, and the intent API's `requester_region` and
     `requester_clearance`.
   - **Policies that matched a region the request claimed stop matching** until admins
     set regions.
2. **Masking combines every matching mask policy**, evaluated with the requester's
   context. When two policies mask one field, the strictest method wins (full, then
   hash, then partial). An approval lifts the approval requirement, not the masks:
   **some fields that came back unmasked after approval are now masked.**
3. **The schema classifier was rewritten.**
   - It matches whole words, so `description`, `tip` and `company` are no longer
     flagged as IP addresses or card numbers.
   - Secrets, government IDs, payment cards, bank accounts and health data are now
     rated `high`, and secrets have their own `credential` category.
   - A text or JSON column it can't place is rated `unknown` for review, not `low`.
   - Rescans keep labels from a deployed bundle or a manual edit, and list the
     classifier's disagreements as suggestions, which `vkit scan` prints. `unknown`
     labels, and labels that only came from earlier scans, are replaced.

   After upgrading, rescan each datasource, review the changes and suggestions in
   `datasets/registry.yaml`, and redeploy.
4. **The policy packs are now 2.0.0.**
   - Run `vkit policy pack upgrade`. The 1.x clearance, region, analyst and financial
     policies never fired; 2.0.0 replaces them with policies that do.
   - `vkit policy bundle` refuses policies that can never match, including those in
     the 1.x `financial_compliance` and `ai_safety` packs, so a bundle that built before
     may fail until you upgrade.
   - Create the approval groups the packs name (`data_owner`, `security`,
     `finance_manager`); until then only admins can approve.
5. **Tenant isolation for shared tables (opt-in).**
   - Mark a dataset with `tenant_column: <column>` in `datasets/registry.yaml`, and
     issue agent tokens for one tenant each:
     `vkit agents tokens create --tenant <id>`.
   - Every query on a marked dataset, directly or through a join, is confined to the
     token's tenant. Grants and approvals are bound to it, and the audit log records
     it.
   - **A requester without a tenant, including every user, is denied on marked
     datasets.**
   - Nothing changes until a dataset is marked. `vkit policy bundle` refuses a dataset
     that has another dataset's tenant column but isn't marked; set
     `tenant_column: false` for a table every tenant may read.

Use the Python SDK 0.3.0 or later for MCP agents, as in 0.8.0.

### Upgrading to 0.8.0-alpha

Update all three image tags in `docker-compose.yml` to `0.8.0-alpha`, then pull and
restart. Upgrading from an older release? Read the notes for each version in between
as well; their changes also apply.

There are no database migrations.

1. **Reworded agent requests reuse their approval.** Agents send a free-text purpose
   with each request, and it used to count toward how VaultKit recognises a repeat
   request. Rewording it sent an approved request back for approval, and a reworded
   request queued a second approval next to a pending one. The purpose no longer counts:
   the same query, from the same requester in the same context (environment, region,
   clearance), reuses its grant until it expires, or joins the pending approval. The
   purpose is still stored and audited, and approvers see the purpose of the first
   request.

   Grants and pending approvals created before the upgrade were recognised the old way,
   so a rerun of one of them asks for approval once more.
2. **The console has the new VaultKit logo.** The sign-in pages, the sidebar, the
   not-found page and the favicon use the refined mark from vaultkit.io, and there is an
   icon for the iOS home screen. The browser tab now reads "VaultKit Console", and a
   shared console link previews with vaultkit.io's image instead of a broken one.

Use `vkit` 1.1.6 or later:
- it reads credentials from `VKIT_ENDPOINT` and `VKIT_TOKEN` when both are set, for
  scripts and CI;
- `vkit policy bundle` keeps `masking.default_method: partial` (and `full`) as written.
  Older versions compiled every method other than the old names to `full`. **If a
  deployed policy uses `partial`, rebuild and redeploy its bundle with 1.1.6** to get
  partial masking instead of full;
- a policy repo with no `origin` remote can now deploy.

Agents using the MCP server need the Python SDK 0.3.0 or later
(`pip install "vaultkit>=0.3.0"`) to run aggregate queries. With 0.2.0, agents fetch
rows and count them themselves.

### Upgrading to 0.7.0-alpha

Update all three image tags in `docker-compose.yml` to `0.7.0-alpha`, then pull and
restart. Upgrading from an older release? Read the notes for each version in between
as well; their changes also apply.

This release redesigns sign-in and organization registration, and tightens how both
behave. There are no database migrations.

1. **`OIDC_ENABLED` now switches SSO off everywhere.** It used to affect only what the
   CLI was offered. With `OIDC_ENABLED=false`:
   - browser SSO sign-in sends people back to sign in with "SSO sign-in is turned off on
     this control plane";
   - organization activation reports the same reason;
   - the CLI's SSO login answers `403 sso_disabled`.

   **If you run with `OIDC_ENABLED=false` but still sign in with SSO in the browser,
   browser SSO stops working.** Set `OIDC_ENABLED=true`, or remove the line (SSO is on by
   default).
2. **One identity provider, and one SSO organization, per control plane.**
   - SSO always uses the control plane's `OIDC_ISSUER`. Registration no longer asks for an
     issuer URL; it shows the configured provider and checks it server-side
     (`GET /auth/identity_provider`).
   - Organization creation runs the same check, and ignores any `oidc_issuer` it's sent.
   - Only one SSO organization can use the provider (this was already true in the
     database). A second SSO registration is refused with `sso_in_use`, and the page offers
     email and password instead.
   - When `OIDC_ISSUER` isn't set, or the provider is unreachable, SSO registration is
     unavailable and the page says why.
3. **Email-first sign-in.** The sign-in page asks for an email first, and its domain
   decides the next step:
   - an email on a domain that an active SSO organization allows (its allowed email
     domains) goes to that organization's SSO;
   - everyone else gets the password step, which offers "Sign in with <provider>
     instead" for people invited from other domains.

   The new `POST /auth/sign_in_method` never reveals whether an account exists.
4. **Sign-in no longer reveals accounts.** Password sign-in now gives the same `401`
   for a wrong password, an unknown email, and a member of an SSO organization. Before,
   SSO members got `403 "This organization uses SSO"`, with an `OIDC_ENABLED=false`
   variant added in this release. Scripts that looked for that `403` should treat a `401`
   as "use SSO or check the credentials".
5. **Passwords need 12 to 128 characters**, with no rules about character classes. The
   control plane enforces it (it used to accept 6), and the console's uppercase, lowercase
   and digit rules are gone. Existing passwords keep working; the rule applies when a
   password is set.
6. **Organization creation API changes** (`POST /api/v1/organizations` and
   `/organizations/password`):
   - a `slug` is required, and it's used as given, for SSO organizations too: lowercase
     letters and digits in hyphen-separated words, up to 60 characters;
   - a taken slug is `422 slug_taken`. VaultKit no longer adds `-2`;
   - errors are `422 { "error": <code>, "message": … }`, with these codes:
     - `slug_taken`, `slug_invalid`, `name_required`, `email_invalid`, `domains_invalid`;
     - `password_too_short`, `password_too_long`;
     - `sso_unavailable`, `sso_in_use`;
     - `cannot_create`, the generic code, which also covers an email that already has an
       account.
   - The SSO response no longer includes `activate_url`. The activation start path is
     `/auth/oidc/org_activation/start?org=<slug>`.
   - An unfinished SSO sign-up can be resumed with the same slug and admin email.
7. **The activation endpoint shows less.** `GET /api/v1/organizations/:slug/activation`
   returns only the name of an organization waiting for activation, and the same `404`
   for active and unknown organizations. It no longer returns the admin email or issuer.
8. **No open sign-up.** `POST /api/users` (a route with no controller behind it) is gone,
   and returns `404`.
9. **Password sign-up has no email verification**, and the console no longer claims it
   does. A new email-and-password organization goes straight to sign in. There's also no
   "Forgot password?": password users have no self-service recovery yet, so whoever runs
   the control plane resets passwords (for example, with `bin/rails console`).

Use `vkit` 1.1.5 or later: `vkit request` now shows requests that are waiting for
approval, with their approver roles. Before, it printed "Unexpected response" and
exited with status 1. The Python SDK needs no changes.

### Upgrading to 0.6.0-alpha

Update all three image tags in `docker-compose.yml` to `0.6.0-alpha`, then pull and
restart. Upgrading from an older release? Read the notes for each version in between
as well; their changes also apply.

This release redesigns the console (Overview, Team, a new Approvals page, user
drawers and the 404 page) and adds the team access model behind it. It includes one
database migration, which runs automatically when the `control-plane` container
starts.

1. **Users migration.** `users` gains `access_level` (`admin` or `member`),
   `approve_any`, `approval_groups`, `last_seen_at` and `deactivated_at`. Existing
   users are converted without changing what they can do:
   - `admin` stays an admin.
   - `approver` becomes a member who can approve any request.
   - `member`, or no role, becomes a member.
   - Any other role, such as `finance_manager`, becomes a member in that approval
     group.

   `users.role` itself is kept. Policies match it as `requester_role`, so they keep
   applying to the same people. It stays `admin` for admins: promoting or demoting
   someone in the console updates it too.
2. **Approval access model.** A user can sign off on a request when one of its
   outstanding approver roles is in their **approval groups**. Admins and members who
   can **approve any request** can sign off on any one outstanding role. Each role still
   needs a different person, and **nobody can act on their own request**.
   - Manage this on the **Team** page, or with `access_level`, `approve_any` and
     `approval_groups[]` on `POST` and `PATCH /api/v1/orgs/:slug/users`.
   - The old `role` parameter still works: `approver` sets approve-any, and any other
     role becomes an approval group.
   - A refused sign-off now says "You aren't in an approval group for …" instead of
     "Your role (…) cannot act on requests for …".
   - The new **Approvals** page in the console lists what's waiting on you, with
     approve, and deny with a reason. `vkit approval:*` keeps working.
3. **Removing users.** Admins can remove someone from the **Team** page, or with
   `DELETE /api/v1/orgs/:slug/users/:id`. Users are deactivated, not deleted, so their
   history stays intact.
   - They can no longer sign in, by password or SSO, and tokens they already have get
     `401`.
   - Their active grants are revoked.
   - Their pending requests are denied with "Requester was removed from the team".
   - Approvals they already signed off on keep their sign-off.
   - You can't remove yourself or the last admin.
   - Adding the same email again brings them back.
   - **Agent tokens they created keep working.** Revoke those separately with
     `vkit agents tokens revoke`.
4. **FUNL status is real now.** `GET /api/v1/system/status` used to report funl as
   `down` whenever `FUNL_ENDPOINT` was unset, which was always, since deployments set
   `FUNL_URL`. The control plane now probes `FUNL_URL/health` with a one-second
   timeout, and reports `running`, `down` or `not_configured`. Keep `FUNL_URL` set, as the
   Quick Start's `.env` does. The response also includes the release
   `version`.
5. **Another organization's URL now returns `404`.** API calls to
   `/api/v1/orgs/<another org's slug>/...` used to return `403` when that organization
   existed and `404` when it didn't, which revealed which slugs exist. Both now return
   `404 {"error": "Not Found"}`. Policy bundle uploads also stop ignoring the slug in
   the URL. If a script checked for `403` on cross-organization calls, check for
   `404` instead.

Other changes:
- **Last seen** is shown for each user on the Team page. It's updated on API use, at
  most every five minutes.
- **The Team page replaces Teams & Roles.** The Overview shows system health, setup
  progress from data VaultKit already records, and what needs your attention.
- **Unknown console URLs** show the 404 page, signed in or not.
- **The control plane exposes `X-Request-Id`** through CORS, so the console can show
  it on error pages.

Use `vkit` 1.1.4 or later; no CLI or Python SDK update is needed for this release.

### Upgrading to 0.5.0-alpha

Update all three image tags in `docker-compose.yml` to `0.5.0-alpha`, then pull and
restart. Upgrading from an older release? Read the notes for each version in between
as well; their changes also apply.

This release tightens how access requests are approved and how masked fields can be
queried, and fixes two sign-off problems with SSO. It includes one database
migration, which runs automatically when the `control-plane` container starts.

1. **Approval policies must name an approver role.** Uploading a policy bundle now
   fails with `422` if any `require_approval` policy has no `approval.approver_role`.
   The error lists the policy ids. Bundles built with `vkit policy bundle` already
   require the role. Bundles that are active now keep working, but a request that
   matches an approval policy with no role now needs an **admin** (or a user with the
   `approver` role) to approve it. Previously anyone in the organization except the
   requester could approve it. Invalid
   bundles (schema or checksum errors) now also return `422` instead of `500`.
2. **Every matched approver role must sign off.** When a request matches several
   approval policies (for example finance on `amount` and security on `ssn`, or two
   joined datasets), each of their roles must approve. Previously one approver from any
   of them was enough.
   - Each role needs a different person. Nobody can sign off twice, and the requester
     can't sign at all.
   - Admins and users with the `approver` role can still sign for any role, but only one
     role per sign-off.
   - The grant is issued after the last sign-off. Until then, approving returns
     `status: "pending"` with the roles still waiting, and the request stays in the
     queue of the people who still need to act.
   - Any eligible approver can deny the whole request.
   - Approvals pending at upgrade time keep their single routed role.
3. **Grant lifetimes are capped by the policy.** An approved grant now lasts for the
   policy's `ttl_seconds` by default, or the shortest TTL when several approval
   policies match. An approver can choose a shorter lifetime, but not a longer one.
   Previously the approver's `ttl_seconds` (CLI default one hour) applied, with no
   maximum. A `ttl_seconds` that isn't a positive integer now returns `422`. Approvals
   pending at upgrade time are capped at one hour. If your policies set no
   `ttl_seconds`, grants still last one hour.
4. **Grouping or sorting by a fully or partially masked field is denied.** Masking only
   changes returned values, so `group_by`, or `order_by` on a masked field, revealed how
   many distinct values were behind a mask and how they sorted. Such queries now get
   `403` with a reason like `Grouping or sorting by fully or partially masked fields is
   not allowed: GROUP BY email`. This includes a masked field selected as a plain column
   next to aggregates with no `group_by`, because those columns are grouped implicitly.
   Fields with **hash** masking can still be grouped and sorted, and `COUNT` over masked
   fields is still allowed.
5. **Okta sign-in works with email verification.** Okta sends `email_verified` only in
   its userinfo response, not in the ID token, so every Okta sign-in was rejected as
   unverified. The control plane now reads it from userinfo when the ID token doesn't
   include it, for the same user and the same email address. Users whose Okta email is
   still unverified are still rejected.
6. **Fixed: some SSO sign-ins ended in `401` on every page.** Exchanging a login session
   twice at the same time could hand the console an empty token. This was seen mostly
   with the development console, which runs the sign-in step twice. The control plane
   now answers the second exchange with `410`, and the console exchanges each sign-in
   once. Users who hit this only need to sign in again.

Use `vkit` 1.1.4 or later to see which approver roles a request still needs.
`vkit approval:approve` also no longer defaults `--ttl` to one hour, so grants get the
policy's lifetime. Older CLIs keep working, but print an empty grant ref after a
partial sign-off. The Python SDK needs no changes.

### Upgrading to 0.4.0-alpha

Update all three image tags in `docker-compose.yml` to `0.4.0-alpha`, then pull and
restart. Upgrading from an older release? Read the notes for each version in between
as well; their changes also apply.

This release hardens how the control plane talks to funl, how organization data is
scoped, and how access requests are approved. There are no database migrations.

1. **Upgrade the control plane and funl together.** Every call from the control plane
   to funl now carries a token issued for that single request: valid for about a minute
   and bound to a SHA-256 of the exact request body. A new `funl-runtime` rejects
   requests from an older `control-plane`, and a new `control-plane` talking to an
   older `funl-runtime` loses that protection. Deploy all three images from the same
   release.
2. **Funl's port is no longer published.** Run `funl-runtime` with
   `expose: 8080` in `docker-compose.yml`, not `ports: 8080:8080`. The control plane
   reaches funl at `http://funl-runtime:8080` on the Compose network. Anything that
   called funl from outside Docker stops working, which is intended. If your own
   `docker-compose.yml` publishes funl's port, remove that mapping, and don't route funl
   through a reverse proxy. Funl now also accepts only tokens signed by the control
   plane (RS256, issuer `vaultkit-control-plane`), one token type per endpoint. See
   **Secrets & Security Notes**.
3. **Rails console and scripts need an organization.** Querying organization-owned
   records (users, grants, tokens, datasources, audit logs…) with no organization set
   now raises `MultiTenant::MissingOrganization` instead of returning every
   organization's data. In `bin/rails console`, set `Current.organization` first, or
   wrap deliberate cross-organization work in `MultiTenant.without_tenant { }`. See
   **Troubleshooting** for examples. VaultKit has no background jobs or rake tasks
   that need changing.
4. **Approvals are stricter.** Only users whose role matches the request's routed
   approver role, admins, and users with the `approver` role can approve or deny a
   request. Requests with no routed role can be acted on by anyone except the requester.
   **Nobody can approve or deny their own request, admins included.** Users who could
   previously approve requests outside their role now get `403`. Check that each
   approver role your policies route to is held by at least one user other than the
   people who request that data.
5. **In-flight sign-ins fail once with a `400`.** The OIDC `state` parameter is now
   signed and expires after 15 minutes. Any SSO sign-in that's mid-flow while you
   deploy gets a `400` on the callback, because its state came from the old version.
   Signing in again works. A `vkit login` caught mid-flow keeps waiting after the
   browser shows the `400`; press Ctrl+C and run it again.
6. **New optional query limits for `funl-runtime`:** `FUNL_MAX_CONCURRENT_QUERIES`
   (default `50`), `FUNL_MAX_CONCURRENT_QUERIES_PER_ORG` (`10`), `FUNL_QUERY_WAIT`
   (`2s`) and `FUNL_QUERY_TIMEOUT` (`20s`). See **Query limits (optional)** above.
   - The defaults keep today's behaviour under normal load. Invalid values stop funl at
     startup with the variable named.
   - `FUNL_MAX_CONCURRENT_QUERIES` also caps funl's active database connections, so
     keep it within Postgres's `max_connections` divided by the number of funl
     instances.
   - Clients may now get `503` "Query engine is busy, retry shortly" (an organization,
     or funl as a whole, is at its limit; retry after a second) or `504` "Query timed
     out" from grant fetches, instead of the generic `502`.

The Python SDK and the `vkit` CLI need no changes for this release.

### Upgrading to 0.3.0-alpha

Update all three image tags in `docker-compose.yml` to `0.3.0-alpha`, then pull and
restart. Upgrading from `0.1.x`? Read the `0.2.x-alpha` notes below as well; their
behaviour changes also apply.

This release tightens **SSO (OIDC) sign-in**. Organizations that use email and
password are unaffected, apart from gaining the Teams & Roles page described below.

#### Before you upgrade an SSO organization

- **Check that your identity provider sends `email_verified`.** Every SSO sign-in now
  requires the identity provider to assert that the user's email is verified. This
  applies to existing users too, not only new ones. Okta, Google and Auth0 send the
  claim; **Microsoft Entra ID often does not**, and its users will be refused with
  "Your identity provider has not verified your email address". If you use Entra ID,
  stay on `0.2.1-alpha` until this is configurable.
- **Decide who should be able to join.** SSO organizations become **invite-only**
  (see below). Have the organization's admin ready to add people or set allowed email
  domains straight after the upgrade.

#### What changes

- **SSO organizations are invite-only by default.** Previously, anyone your identity
  provider vouched for was given an account on first sign-in, and the first person to
  sign in became an admin. Now:
  - Users who already exist in the organization keep signing in as before.
  - New users can sign in only if an admin has added them, or if their email's domain
    is on the organization's **allowed email domains** list. Exact domains only:
    `acme.com` does not include `eu.acme.com`.
  - Users created on first sign-in are always members. Admins come only from
    organization activation or from an existing admin.
  - Everyone else sees "You are not a member of this organization yet. Ask an
    administrator to add you."
- **Managing members.** Admins can add users and change roles on the console's
  **Teams & Roles** page, and set allowed email domains there for SSO organizations.
  The same is available from the API at `GET/POST/PATCH /api/v1/orgs/:slug/users` and
  `GET/PATCH /api/v1/orgs/:slug/settings`. New SSO organizations can set allowed
  domains during setup. In email-and-password organizations, admins set an initial
  password for each new user; VaultKit does not send invite emails yet.
- **One organization per email address.** An email that already belongs to one
  organization can't be added to, sign up for, or activate another.
- **CLI sign-in.** When an SSO sign-in started by `vkit login` is rejected, the CLI now
  stops straight away instead of waiting about 10 minutes for the login session to
  expire. CLI releases after `1.1.2` (which include vaultkit-inc/vaultkitcli#26) also
  print the reason; `1.1.2` and earlier print `Unexpected response: 403`.
- **Database migrations.** Two migrations add `organizations.allowed_email_domains` and
  three columns to `cli_login_sessions`. Both only add columns and run automatically on
  startup.

#### After you upgrade

1. Sign in to the console as an admin of each SSO organization.
2. On **Settings → Teams & Roles**, either add the people who need access, or enter your
   company's email domains under **Allowed email domains**.
3. Ask a non-admin user to sign in, to confirm your identity provider's emails are
   accepted.

### Upgrading to 0.2.x-alpha

Update all three image tags in `docker-compose.yml` to `0.2.1-alpha`, then pull and
restart. Use `0.2.1-alpha` rather than `0.2.0-alpha`: the code is the same, but the
`0.2.0-alpha` images were published for `linux/arm64` only and do not run on x86_64
hosts. From `0.2.1-alpha` on, images are published for both `linux/amd64` and
`linux/arm64`. The `control-plane` image also moves from Ruby 3.1 on Debian 11, which
no longer receives updates, to Ruby 3.4 on Debian 13. This is internal to the image
and needs no configuration changes.

Behaviour changes to be aware of (compared with `0.1.x`):

- **Row limits.** Queries without a `limit` return at most 100 rows. `limit` must be
  between 1 and 10,000; `limit: 0` (previously "no limit") is rejected.
- **Wider policy checks.** Fields used in filters, `group_by`, `having`, `order_by`,
  aggregates and joins are now policy-checked against the dataset they belong to.
  Requests that filter or aggregate on sensitive fields may now be denied or require
  approval.
- **Aggregates over masked fields.** `MIN`, `MAX`, `SUM` and `AVG` over a masked field
  are denied. `COUNT` is still allowed.
- **Joins.** Joined datasets must exist in the registry. Joined columns keep their
  table prefix, and join aliases now work. When two selected columns share a name
  (e.g. `users.id` and `o.id`), they are returned as `users_id` and `o_id`.
- **Grant reuse.** Grants for queries with joins or without a `limit` get new
  fingerprints, so they are evaluated again on first use after the upgrade.
- **Errors.** Invalid AQL now returns `422` with funl's message at fetch time, and other
  funl failures return `502`, instead of `500`.
- **Intent API.** Supports nested `AND`/`OR` filters, `in` lists, and `aggregates`,
  `group_by`, `having` and `order_by` (single dataset). Use Python SDK `vaultkit>=0.3.0`
  for analytics; older control planes silently ignore those parameters.

---

## Troubleshooting

### Containers start but UI shows 502

* Check `docker compose logs console`
* Confirm the control-plane container is running: `docker compose ps`
* Ensure the Console can reach the Control Plane over Docker's internal network

### OIDC redirect loops or 404s

* Confirm redirect URI matches exactly
* Ensure `/auth/oidc/callback` resolves to the Console URL
* Ensure `/auth/callback` is handled by the React app

### Cannot access the Console

* Confirm port 5173 is open on your firewall or security group
* Check `docker compose logs console`

---

## Recommended Next Steps

After deployment:

* Create your first organization
* Configure OIDC provider (optional for evaluation)
* Generate agent tokens
* Register data sources
* Apply your first policy bundle

---

## Support

If you are a design partner and need help:

* Email: founders@vaultkit.io
* Share logs from `docker compose logs`
* Documentation: [docs.vaultkit.io](https://docs.vaultkit.io)

---

**VaultKit is designed to deploy cleanly, predictably, and without surprises.**
If something feels harder than it should be, that's a signal — and we want to hear about it.