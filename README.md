# Auth Mesh Client Setup

Set up your service with Auth Mesh in minutes. This CLI registers your service,
configures authentication, and ensures users can access your application with
zero ongoing maintenance.

## Quick Start

```bash
# 1. Copy this directory into your project
cp -r setup/ my-project/setup/
cd my-project/setup/

# 2. Copy example configs and edit them
cp config/permissions.json.example config/permissions.json
cp config/oauth-client.json.example config/oauth-client.json
cp config/hosted-login.json.example config/hosted-login.json
# Edit each file for your service

# 3. Run setup
./setup
```

The interactive CLI guides you through each step. Or run everything at once:

```bash
./setup run
```

## How It Works

Auth Mesh uses a Zanzibar-based permission model where **permissions are
inherited through organizational hierarchy**. The setup creates this structure:

```
Service Org (your-service)                  <- admin, API key, permission schema
  |
  └── End-Users Org (your-service-users)    <- users sign up HERE
        |   parent_id = service org
        |   default_role = member
        |   permissions inherited from org membership
        |
        ├── User A (member) -> gets all default_grant permissions
        ├── User B (member) -> gets all default_grant permissions
        └── User C (member) -> gets all default_grant permissions
```

**Zero-config permission model:**
- Permissions are defined once in `config/permissions.json`
- Permissions marked `default_grant: true` are set at the org level
- Users who sign up via hosted login join the end-users org as `member`
- Members inherit permissions through Zanzibar's org hierarchy
- No per-user grants. No callbacks. No cron jobs. No maintenance.

## Requirements

- **bash** 4.0+
- **curl**
- **jq**
- **python3**
- Network access to your Auth Mesh instance

## Setup Steps

| Step | Script | What It Does |
|------|--------|-------------|
| **01** | `register-service-permissions.sh` | Creates service org, registers permissions, creates admin + API key |
| **02** | `register-oauth-client.sh` | Registers OAuth 2.1 public client (PKCE) for your frontend |
| **03** | `setup-hosted-login.sh` | Configures hosted login page (branding, registration settings) |
| **04** | `setup-end-users-org.sh` | Creates end-users child org with inherited permissions |
| **05** | `verify-setup.sh` | Verifies everything is configured correctly |

Each script is **idempotent** — safe to run multiple times.

## Configuration

Edit the config files before running setup. See the `.example` files for the
full schema.

### `config/permissions.json`

Defines your service's permissions, roles, and metadata. This is the source of
truth for what your service can do and who can do it.

```bash
cp config/permissions.json.example config/permissions.json
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `service.id` | Unique service identifier (used in org slugs, API keys, permission IDs) |
| `service.audience` | JWT audience claim for token validation |
| `registration.actions` | Actions your service supports (read, write, delete, etc.) |
| `registration.resources` | Resources your service manages (items, reports, etc.) |
| `permissions[].id` | Permission ID in format `{service}.{action}.{resource}` |
| `permissions[].default_grant` | If `true`, users get this permission via org membership |
| `roles[].default` | If `true`, this is the default role for self-registered users |

See `config/permissions.json.example` for full schema with all fields.

### `config/oauth-client.json`

Configures the OAuth 2.1 client for your frontend application.

```bash
cp config/oauth-client.json.example config/oauth-client.json
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `client_name` | Display name for the OAuth client |
| `redirect_uris` | Allowed callback URLs (must include localhost for dev) |
| `token_endpoint_auth_method` | Use `"none"` for public clients (SPAs, mobile) |
| `scope` | OAuth scopes (typically `"openid profile email"`) |

See `config/oauth-client.json.example` for full schema.

### `config/hosted-login.json`

Customizes the hosted login page and registration settings.

```bash
cp config/hosted-login.json.example config/hosted-login.json
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `branding.primary_color` | Brand color for login page |
| `branding.page_title` | Browser tab title |
| `auth_methods.signup_enabled` | Allow self-registration |
| `registration.default_role` | Role for new users (overridden to `member` in step 04) |
| `security.post_logout_redirect_uri` | Where to send users after logout |

See `config/hosted-login.json.example` for full schema.

## CLI Usage

```bash
# Interactive menu
./setup

# Run all steps
./setup run

# Run a specific step
./setup run 01
./setup run 04

# Check what's configured
./setup status

# Verify everything works
./setup verify

# Preview without making changes
./setup dry-run

# Help
./setup help
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `AUTH_SERVICE_URL` | `https://auth.service.ab0t.com` | Auth Mesh server URL |
| `DRY_RUN` | `0` | Set to `1` to preview without making changes |
| `PERMISSIONS_FILE` | `config/permissions.json` | Override permissions config path |
| `CLIENT_CONFIG` | `config/oauth-client.json` | Override OAuth client config path |
| `LOGIN_CONFIG` | `config/hosted-login.json` | Override hosted login config path |

### Targeting different environments

```bash
# Production (default)
./setup run

# Development
AUTH_SERVICE_URL=https://auth.dev.ab0t.com ./setup run

# Local
AUTH_SERVICE_URL=http://localhost:8001 ./setup run
```

The CLI auto-detects the environment and uses separate credential files
(e.g., `service-dev.json` vs `service.json`) so you can target multiple
environments from the same directory.

## Generated Credentials

After setup, the `credentials/` directory contains the generated output.
These files are **gitignored** — never commit them.

See the `.example` files in `credentials/` for the exact schema of each output:

| File | Generated By | Contains |
|------|-------------|----------|
| `{service}.json` | Step 01 | Org ID, admin creds, API key |
| `oauth-client.json` | Step 02 | OAuth client_id, registration token |
| `hosted-login.json` | Step 03 | Login config verification |
| `end-users-org.json` | Step 04 | End-users org ID, login URL, permission list |

## Directory Structure

```
setup/
  setup                              CLI entry point
  README.md                          This file
  INTENT.txt                         Design intent and architecture notes
  config/                            Input configs (edit these)
    permissions.json                 Your service's permission schema
    permissions.json.example         Schema reference
    oauth-client.json                OAuth client config
    oauth-client.json.example        Schema reference
    hosted-login.json                Login page branding
    hosted-login.json.example        Schema reference
  credentials/                       Generated output (gitignored)
    .gitignore                       Prevents committing secrets
    service.json.example             Output schema reference
    oauth-client.json.example        Output schema reference
    hosted-login.json.example        Output schema reference
    end-users-org.json.example       Output schema reference
  scripts/                           Numbered setup scripts
    01-register-service-permissions.sh
    02-register-oauth-client.sh
    03-setup-hosted-login.sh
    04-setup-end-users-org.sh
    05-verify-setup.sh
```

## After Setup

### Frontend Integration

```javascript
import { AuthMeshClient } from '@authmesh/sdk';

const auth = new AuthMeshClient({
  // From credentials/end-users-org.json:
  domain: 'https://auth.service.ab0t.com',
  org: '{org_slug from end-users-org.json}',

  // From credentials/oauth-client.json:
  clientId: '{client_id}',

  redirectUri: window.location.origin + '/auth/callback',
  scope: 'openid profile email',
});
```

### Backend Integration

```python
from ab0t_auth import AuthGuard

guard = AuthGuard(
    auth_service_url="https://auth.service.ab0t.com",

    # From credentials/{service}.json:
    api_key="{api_key.key}",
    audience="{auth.audience}",    # LOCAL:{org_id}
)
```

### Permission Design Tips

- Use `default_grant: true` for any permission a regular user needs
- Use `default_grant: false` for admin-only or sensitive operations
- Permission IDs follow `{service}.{action}.{resource}` format
- Define an `admin` permission with `implies` for admin users
- The `default_grant` permissions are what users get through org membership

## Troubleshooting

### "Permission denied" when running scripts

```bash
chmod +x setup scripts/*.sh
```

### Users getting 403

1. Run `./setup verify` to check the setup
2. Ensure your frontend points to the **end-users org** login URL (not the
   service org). Check `credentials/end-users-org.json` for the correct URL
3. Verify the `audience` in your backend matches `LOCAL:{org_id}` from
   `credentials/{service}.json`
4. Check that the user signed up through the end-users org hosted login page

### "Organization already exists"

Normal — all scripts are idempotent and reuse existing resources.

## Architecture: Why No Per-User Grants

Auth Mesh uses Google Zanzibar (relation-based access control) internally.
When the setup creates a child org with `parent_id`, the auth server
automatically creates a Zanzibar parent relationship:

```
organization:{end-users-org}#parent@org:{service-org}
```

Permission checks walk this relationship chain. A user who is a `member` of
the end-users org inherits permissions through the org hierarchy — the same
way a team member inherits team permissions.

This means:
- **No per-user permission grants needed** — permissions are structural
- **No callbacks or cron jobs** — the auth server handles it at check time
- **New users work immediately** — membership = permissions
- **Revoking is structural too** — remove from org = lose permissions

This is the same model used by Google (Zanzibar), GitHub (org/team
permissions), and Slack (workspace membership). It scales to millions of
users without per-user permission records.

## Support

- Auth Mesh docs: https://auth.service.ab0t.com/docs
- Platform team: platform-team@ab0t.com
