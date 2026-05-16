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

One URL serves everything — nginx inside the Console handles routing. No external reverse proxy required.

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
* Generates RSA signing keys (`vkit_priv.pem`, `vkit_pub.pem`)
* Displays your machine's IP address for reference

---

### 3. Configure `.env`

Edit `.env` and set `APP_HOST` and `FRONTEND_BASE_URL` to wherever VaultKit will be accessed from. Pick the option that matches your setup:

---

**Option A — Local machine**

```env
RAILS_ENV=production

APP_HOST=http://localhost:5173
FRONTEND_BASE_URL=http://localhost:5173

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

DATABASE_URL=postgres://vaultkit:<password>@postgres:5432/vaultkit
FUNL_URL=http://funl-runtime:8080
VKIT_PRIVATE_KEY=/secrets/keys/vkit_priv.pem
VKIT_PUBLIC_KEY=/secrets/keys/vkit_pub.pem
```

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

> For Option C, point your domain's A record to your server's IP address and configure TLS via Cloudflare or Let's Encrypt before starting VaultKit.

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

```bash
docker compose pull
docker compose up -d
```

Database migrations run automatically on container startup.

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