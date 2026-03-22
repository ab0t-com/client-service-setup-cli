# OAuth and Hosted Login

## OAuth Client Config (config/oauth-client.json)

```json
{
  "client_name": "Sandbox Platform Frontend",
  "redirect_uris": [
    "https://sandbox.dev.ab0t.com/auth/callback",
    "http://localhost:3000/auth/callback"
  ],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "scope": "openid profile email"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `client_name` | Yes | Display name shown in consent screens |
| `redirect_uris` | Yes | Exact-match callback URLs. Include both prod and localhost for dev |
| `grant_types` | Yes | Use `["authorization_code", "refresh_token"]` for web apps |
| `response_types` | Yes | Use `["code"]` for authorization code flow |
| `token_endpoint_auth_method` | Yes | `"none"` for public clients (SPA/mobile) — uses PKCE instead of client_secret |
| `scope` | Yes | Space-separated OAuth scopes. Standard: `"openid profile email"` |

### Key Rules

- **Public clients** (SPA, mobile) must use `token_endpoint_auth_method: "none"` + PKCE
- **Redirect URIs** must match exactly during OAuth flow (no wildcards)
- **Include localhost** in redirect_uris for local development
- Step 02 registers via RFC 7591 (`POST /auth/oauth/register`)
- Output includes `registration_access_token` for RFC 7592 updates

## Hosted Login Config (config/hosted-login.json)

```json
{
  "branding": {
    "primary_color": "#111111",
    "background_color": "#FAFAFA",
    "page_title": "Sandbox Platform - Sign In",
    "hide_powered_by": false,
    "login_template": "default"
  },
  "content": {
    "welcome_message": "Sign in to Sandbox Platform",
    "signup_message": "Create your account",
    "footer_message": "Sandbox Platform by ab0t"
  },
  "auth_methods": {
    "email_password": true,
    "signup_enabled": true,
    "invitation_only": false
  },
  "registration": {
    "default_role": "member",
    "require_email_verification": false,
    "collect_name": true
  },
  "security": {
    "post_logout_redirect_uri": "https://sandbox.dev.ab0t.com",
    "remember_me_enabled": true
  }
}
```

### Fields

| Section | Field | Description |
|---------|-------|-------------|
| `branding.primary_color` | Hex color for buttons, links |
| `branding.page_title` | Browser tab title |
| `branding.login_template` | `"default"`, `"minimal"`, or `"corporate"` |
| `content.welcome_message` | Main heading on login page |
| `content.signup_message` | Heading on registration form |
| `auth_methods.signup_enabled` | Allow self-registration |
| `auth_methods.invitation_only` | Override — if true, signup_enabled is ignored |
| `registration.default_role` | Role for new users. Step 04 overrides to `"member"` |
| `registration.default_team` | Team ID for auto-join. Step 04 injects this |
| `security.post_logout_redirect_uri` | Where to send users after logout |

### Important

- Step 03 applies config via `PUT /organizations/{org_id}/login-config` (full replace, not merge)
- Step 04 **injects** `registration.default_role = "member"` and `registration.default_team = {team_id}` into this config
- The hosted login page is public at: `{AUTH_SERVICE_URL}/login/{org_slug}`

## Hosted Login URLs

| Endpoint | Purpose |
|----------|---------|
| `GET /login/{org_slug}` | Public login page (HTML) |
| `GET /organizations/{org_slug}/login-config/public` | Public config (JSON, for BYOUI) |
| `GET /organizations/{org_slug}/auth/providers` | Available auth methods |
| `POST /organizations/{org_slug}/auth/login` | Org-scoped login |
| `POST /organizations/{org_slug}/auth/register` | Org-scoped registration |
| `POST /organizations/{org_slug}/auth/authorize` | OAuth authorization (PKCE) |

## Two Integration Patterns

### SDK-Based (recommended for SPAs)

Frontend uses `@authmesh/sdk` — handles OAuth 2.1 PKCE flow automatically:

```javascript
import { AuthMeshClient } from '@authmesh/sdk';

const auth = new AuthMeshClient({
  domain: 'https://auth.service.ab0t.com',
  org: 'sandbox-platform-users',     // end-users org slug
  clientId: 'client_UxYgGTDmyVgBwnyzTDhI1Q',
  redirectUri: window.location.origin + '/auth/callback',
  scope: 'openid profile email',
});
```

### BYOUI (custom login form)

Frontend calls org-scoped API directly — no redirects, no SDK:

```javascript
const response = await fetch(
  `https://auth.service.ab0t.com/organizations/${orgSlug}/auth/login`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  }
);
const { access_token, refresh_token } = await response.json();
```

### Which Org Slug?

- **Service org** (`sandbox-platform`): for admin login, API key management
- **End-users org** (`sandbox-platform-users`): for user registration and login. This is what the frontend uses
