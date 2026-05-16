# VaultKit Deployment Guide

This document explains how to deploy VaultKit in production and production-like environments.
VaultKit is designed to be **easy to deploy for design partners**, while remaining flexible enough for advanced infrastructure setups.

VaultKit consists of four main components:

* **Control Plane** — Governs every data request: authentication, policy evaluation, approvals, and audit logging
* **FUNL Runtime** — Data plane and execution engine: translates queries into native SQL, applies field-level masking, and executes against your datasources
* **Console** — React web UI served via nginx
* **Postgres** — Metadata and application state

---

## Deployment Models

VaultKit supports two deployment models out of the box.

### 1. Single-Domain Deployment (Default)

Best for:

* Design partners
* Small teams
* Fast evaluation
* Minimal infrastructure

Example:

```
https://vaultkit.yourdomain.com
```

In this mode:

* One domain serves **both frontend and backend**
* The Console's nginx proxies API and auth routes to the Control Plane
* No external reverse proxy is required (Cloudflare / ALB optional)

This is the **recommended starting point**.

---

### 2. Multi-Domain Deployment (Advanced)

Best for:

* Larger organizations
* Existing infra standards
* Strict network separation

Example:

```
https://app.acme.com   → Console (frontend)
https://api.acme.com   → Control Plane (backend)
```

In this mode:

* Customer infrastructure (Cloudflare, ALB, NGINX Ingress, etc.) handles routing
* Console nginx serves static assets only
* Backend is exposed separately

**No VaultKit images need to be rebuilt** — only routing changes.

---

## Prerequisites

* Docker 24+
* Docker Compose v2
* A Linux host (VM, EC2, bare metal)
* A domain name (recommended)
* Optional: Cloudflare or another reverse proxy

---

## Quick Start (Single-Domain)

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

* Verifies Docker and Docker Compose
* Copies `.env.example` → `.env` (if missing)
* Generates signing keys (`vkit_priv.pem`, `vkit_pub.pem`)

If `.env` is newly created, the script will stop and ask you to edit it.

---

### 3. Configure `.env`

Edit `.env` and set at least:

* `POSTGRES_PASSWORD`
* `APP_HOST`
* `FRONTEND_BASE_URL`
* OIDC settings (if using SSO)

Example (single-domain):

```env
RAILS_ENV=production

APP_HOST=https://vaultkit.yourdomain.com
FRONTEND_BASE_URL=https://vaultkit.yourdomain.com

DATABASE_URL=postgres://vaultkit:<password>@postgres:5432/vaultkit

FUNL_URL=http://funl-runtime:8080

VKIT_PRIVATE_KEY=/secrets/keys/vkit_priv.pem
VKIT_PUBLIC_KEY=/secrets/keys/vkit_pub.pem
```

---

### 4. Start VaultKit

```bash
docker compose up -d
```

Verify services:

```bash
docker compose ps
```

All containers should be running.

---

### 5. Access VaultKit

* Console UI:

  ```
  https://vaultkit.yourdomain.com
  ```

* Health check:

  ```
  https://vaultkit.yourdomain.com/up
  ```

---

## OIDC / SSO Configuration

VaultKit supports OIDC providers such as Okta, Auth0, Azure AD, and others.

For browser-based login:

* Redirect URI **must be**:

  ```
  https://<your-domain>/auth/oidc/callback
  ```

VaultKit intentionally redirects **back to the frontend** after successful authentication.
The frontend then exchanges the session with the backend.

This design allows:

* SPA-first UX
* Flexible domain layouts
* Clean separation of concerns

---

## Multi-Domain Deployment

If you want separate frontend and backend domains:

### Example

```
app.acme.com → Console
api.acme.com → Control Plane
```

### Required Changes

1. Update `.env`:

```env
APP_HOST=https://api.acme.com
FRONTEND_BASE_URL=https://app.acme.com
```

2. Configure your reverse proxy / ingress to route:

* `/` → Console container
* Backend traffic → Control Plane container

3. Console nginx runs in **static-only mode** (no API proxying required)

VaultKit images remain unchanged.

---

## Using an External Postgres Database

By default, VaultKit runs Postgres inside Docker.

If you already have Postgres:

* Remove the `postgres` service from `docker-compose.yml`
* Point `DATABASE_URL` to your existing database
* Ensure network access is allowed

Example:

```
DATABASE_URL=postgres://vaultkit:<password>@db.acme.internal:5432/vaultkit
```

---

## Secrets & Security Notes

* `.env` **must not** be committed
* Signing keys are generated locally and mounted read-only
* Rails secrets (`secret_key_base`, JWT secrets) are read from credentials or env
* TLS termination is expected to happen **outside** the containers

---

## Updating VaultKit

To update to a new version:

```bash
docker compose pull
docker compose up -d
```

Database migrations run automatically on container startup.

---

## Troubleshooting

### Containers start but UI shows 502

* Verify reverse proxy routing
* Check `docker compose logs console`
* Confirm backend is reachable from console container

### OIDC redirect loops or 404s

* Confirm redirect URI matches exactly
* Ensure `/auth/oidc/callback` is routed to backend
* Ensure `/auth/callback` is handled by frontend

---

## Recommended Next Steps

After deployment:

* Create your first organization
* Configure OIDC provider
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