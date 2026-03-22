---
name: auth-mesh-setup
description: Onboard any service to the ab0t Auth Mesh using the setup CLI and numbered scripts (01-07). Use when registering a new service with auth, designing a permissions.json schema, configuring OAuth clients, setting up hosted login pages, creating end-users orgs with team-based permission inheritance, running or debugging setup scripts, understanding the Zanzibar permission model, verifying setup health, troubleshooting registration failures, or adapting the setup system for a new service. Covers the full onboarding lifecycle from permission design through org creation, OAuth registration, hosted login branding, default team setup, verification, and consumer registration.
---

# Auth Mesh Setup

Onboard any service to Auth Mesh using the setup CLI. The system is config-driven, idempotent, and environment-aware.

## How It Works

```
config/permissions.json     →  01: Service org + permissions + API key
config/oauth-client.json    →  02: OAuth 2.1 public client (PKCE)
config/hosted-login.json    →  03: Login page branding + settings
                            →  04: End-users org + default team + permission inheritance
                            →  05: Verification
                            →  06: End-to-end test (new user gets permissions)
clients.d/<provider>.json   →  07: Consumer registration (cross-service API keys)
```

### Org Structure Created

```
Service Org (your-service)              ← step 01
├── admin account + API key
├── permission schema registered
│
└── End-Users Org (your-service-users)  ← step 04
    ├── parent_id = service org (Zanzibar relation)
    ├── Default Team                    ← holds default_grant permissions
    │   └── new users auto-join
    ├── User A (member → team → permissions)
    ├── User B (member → team → permissions)
    └── OAuth client + hosted login
```

### Permission Flow

Users get permissions from **team membership**, not per-user grants:

```
User registers → joins end-users org → auto-joins Default Team → inherits team permissions
```

No webhooks, cron, callbacks, or per-user grants. Zanzibar resolves it at check time.

## CLI Usage

```bash
./setup              # Interactive menu
./setup run          # Run all pending steps
./setup run 04       # Run specific step
./setup status       # Show progress
./setup verify       # Run checks (step 05)
./setup dry-run      # Preview without changes
```

Environment: `AUTH_SERVICE_URL=https://auth.service.ab0t.com` (default). Set to `http://localhost:8001` for local dev.

## Script Reference

| Step | Script | Config Input | Credential Output | Purpose |
|------|--------|-------------|-------------------|---------|
| 01 | `register-service-permissions.sh` | `permissions.json` | `{service}.json` | Org + permissions + API key |
| 02 | `register-oauth-client.sh` | `oauth-client.json` | `oauth-client.json` | OAuth 2.1 PKCE client |
| 03 | `setup-hosted-login.sh` | `hosted-login.json` | `hosted-login.json` | Login page branding |
| 04 | `setup-default-team.sh` | `permissions.json` + `hosted-login.json` | `end-users-org.json` | End-users org + team |
| 05 | `verify-setup.sh` | all credentials | — | Health checks |
| 06 | `test-end-user.sh` | end-users-org.json | — | E2E registration test |
| 07 | `register-consumer.sh` | `clients.d/*.json` | `*-consumer.json` | Cross-service API keys |

All scripts are **idempotent** — safe to re-run. Environment-aware: auto-detects dev vs prod and uses `-dev` credential suffix for local.

## References

- **[Permission design](references/permissions-design.md)** — `permissions.json` schema, field reference, `default_grant` rules, `implies` chains
- **[OAuth and hosted login](references/oauth-hosted-login.md)** — OAuth client config, hosted login branding, redirect URIs, org-scoped login endpoints
- **[Script internals](references/script-internals.md)** — what each numbered script does step-by-step, API endpoints called, gotchas
- **[Credential schemas](references/credential-schemas.md)** — output file formats for all credential files
- **[Zanzibar model](references/zanzibar-model.md)** — how parent orgs, teams, and permission inheritance work
- **[Troubleshooting](references/troubleshooting.md)** — common errors, stale credentials, DB wipe recovery

## Quick Start for a New Service

1. Copy the `setup/` directory into your project
2. Edit `config/permissions.json` — define your service's permissions
3. Edit `config/oauth-client.json` — set redirect URIs for your frontend
4. Edit `config/hosted-login.json` — brand your login page
5. Run `./setup run`
6. Wire credentials into your app (see credential output files)

See [references/permissions-design.md](references/permissions-design.md) for the config schema.
