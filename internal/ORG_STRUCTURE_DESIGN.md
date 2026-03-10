# Org Structure Design — Thinking Document

## Status: v1.1 — 2026-03-07

## The Problem

Every service that integrates with Auth Mesh needs to answer one question during
setup: **"How should your users be organized?"**

Today, the setup CLI creates a single fixed structure (Service Org → End-Users
Org). This covers self-service SaaS apps but doesn't cover B2B multi-tenant,
reseller, corporate, or dynamic/agent-driven use cases.

We need a modular, config-driven way to express org structure so the same CLI
can provision any of these patterns.


## Platform Capabilities (Ground Truth from OpenAPI)

Before designing archetypes, here's what the Auth Mesh platform actually
provides. Every capability below is verified against the live OpenAPI spec
at `auth.service.ab0t.com/openapi.json`.

### Org Hierarchy
- **Arbitrarily deep nesting** — `POST /organizations/ {parent_id}` chains
  as deep as needed. Any org can be parent of any other.
- **Hierarchy visualization** — `max_depth` up to 10, D3.js compatible
- **Zanzibar parent relationships** — `POST /zanzibar/stores/{id}/hierarchy/setup`
  writes `organization:{child}#parent@org:{parent}`, permission checks walk
  the chain recursively

### Service Accounts
- **First-class entity** — `POST /admin/users/create-service-account`
  `{email, name, permissions[], org_id}` → returns `{id, email, api_key}`
- **Org-scoped** — each service account belongs to a specific org
- **Auto API key** — generated on creation, shown once
- **Password rotation exempt** — designed for automation
- **API key CRUD** — `/api-keys/` with org-scoped, permission-scoped keys

### OAuth Grant Types (all supported)
- `authorization_code` (PKCE) — frontend user auth
- `refresh_token` — session renewal
- `client_credentials` — m2m auth with `client_id` + `client_secret`

### Delegation
- **Grant** — `POST /delegation/grant {actor_id, scope[], expires_in_hours}`
  Can only delegate permissions YOU have.
- **Delegated JWT** — `POST /auth/delegate {target_user_id}` returns token
  with both actor and subject identities
- **Check/List/Revoke** — full lifecycle management
- **Use cases** — vacation coverage, support access, admin assistance

### SSO/SAML
- **Per-org SAML SP** — each org can have its own IdP (Okta, Azure AD, ADFS)
- **Initiate** — `GET /organizations/{org_slug}/auth/sso/initiate`
- **ACS Callback** — `POST /organizations/{org_slug}/auth/sso/callback`
- **JIT Provisioning** — auto-creates users from SAML assertions, adds to org
- **OAuth integration** — SSO initiate accepts PKCE params, feeds into OAuth flow

### Teams
- **Within-org grouping** — `POST /organizations/{org_id}/teams`
- **Nested teams** — parent/child relationships
- **Permission inheritance** — team members inherit via Zanzibar
- **Members CRUD** — `/teams/{team_id}/members`

### Forward Auth
- **Edge proxy decision** — `GET /forward-auth/` (Traefik ForwardAuth compatible)
- **200** = authorized (enrichment headers), **401** = no identity, **403** = no permission

### Super Admin
- **Time-limited privilege elevation** — grant/approve/revoke/extend
- **Audit log** — full trail of elevated access

### Additional
- **Org invitations** — `POST /organizations/{org_id}/invite`
- **Per-org hosted login** — `/login/{org_slug}` with branding
- **Per-org email templates** — customizable per org
- **Org switch** — `POST /auth/switch-organization`
- **Quotas** — per-user org creation limits by billing tier


## The Insight: Tiers + Cross-Cutting Capabilities

Every org structure pattern reduces to **two dimensions**:

1. **Tiers** — the vertical org tree (who reports to whom, who inherits from whom)
2. **Cross-cutting capabilities** — features that apply across archetypes:
   - Service accounts (m2m auth)
   - Delegation (act-on-behalf)
   - SSO/SAML (enterprise identity)
   - Teams (within-org grouping)
   - Forward auth (edge proxy)

The differences between archetypes are:
1. **How many tiers** (2 for self-service, 3 for reseller, N for corporate)
2. **Who creates orgs at each tier** (setup script, admin, API)
3. **How users join** (self-registration, invite, SSO JIT, programmatic)
4. **Isolation rules** (soft boundaries vs hard walls vs permission ceilings)
5. **Which cross-cutting capabilities are enabled** (service accounts on/off per tier)


## The Six Archetypes

| Archetype | Tiers | Service Accounts | SSO | Delegation | Teams |
|-----------|-------|-----------------|-----|------------|-------|
| **Self-Service SaaS** | 2 | Optional (root) | Optional | No | No |
| **B2B Multi-Tenant** | 2 | Yes (root + customer) | Optional per-customer | Optional | Optional |
| **Reseller / White-Label** | 3 | Yes (root + reseller) | Optional | Optional | Optional |
| **Departmental / Internal** | 2 | Yes (root) | Yes (per-dept) | Yes | Yes |
| **Corporate / Enterprise** | N (deep) | Yes (root + divisions) | Yes (per-org) | Yes | Yes |
| **Dynamic / Agent-Driven** | 2 | Yes (root drives everything) | No | No | No |

### Corporate vs Departmental

The departmental archetype is a **simplified subset** of corporate:
- Departmental: 2 tiers (root → departments), flat
- Corporate: N tiers (root → divisions → departments → teams → ...), deep

Corporate adds: deep nesting, per-org SSO, delegation chains, service accounts
at multiple levels, teams within orgs, forward auth.


## Schema Design

### Core: `tiers[]`

The `tiers` array describes the org hierarchy. Each entry is one level:

```json
{
  "level": 1,
  "role": "customer",
  "slug_template": "{service_id}-{customer_slug}",
  "creation": "dynamic",
  "parent_tier": 0,
  "membership": {
    "default_role": "member",
    "signup_enabled": false,
    "invite_enabled": true
  },
  "permission_model": "org-inherited"
}
```

**Why an array, not a tree?** `parent_tier` makes the tree explicit while
keeping the schema flat. Most archetypes have 2 tiers. Reseller has 3.
Corporate uses a recursive tier (`level: "N"`, `parent_tier: "any"`).

**Key: tiers are not a depth limit.** The `parent_id` field on organizations
chains arbitrarily deep. The tier array describes the *pattern*, not the
*maximum depth*. The corporate archetype's recursive tier makes this explicit.

### Cross-Cutting: `service_accounts`, `delegation`, `sso`, `teams`

These are top-level schema fields, not part of tiers. Each has:
- `enabled: bool` — is this capability active?
- `tiers: int[]` — which tier levels can use it?
- Capability-specific config

This design means any archetype can opt into any cross-cutting capability.
A self-service app can add SSO without changing its tier structure.

### Creation Modes

| Mode | Who | When | Example |
|------|-----|------|---------|
| `setup` | CLI script | Initial setup | Service org, end-users org |
| `admin-provisioned` | Platform admin | Onboarding | Customer orgs, departments |
| `dynamic` | Admin or API | Runtime | Customer orgs (self-service B2B) |
| `api` | Automation/code | Runtime | Agent sessions, CI environments |

### Permission Models

Only one model: `org-inherited`. Permissions flow through Zanzibar parent
relationships. This is by design — the Zanzibar model handles every use case
through hierarchy, not per-user grants.


## Mapping to Zanzibar

Each tier maps directly to Zanzibar relationships:

```
Tier 0 (Service Org):
  → organization:{service_org}#owner@user:{admin}
  → organization:{service_org}#api_key@key:{api_key}

Tier 1 (Customer/Department/Division):
  → organization:{child_org}#parent@org:{service_org}
  → organization:{child_org}#member@user:{user}

Tier 2+ (Deeper nesting):
  → organization:{grandchild_org}#parent@org:{child_org}
  → organization:{grandchild_org}#member@user:{user}

Service Account:
  → Same as user membership, but account_type="service"
  → organization:{org}#member@user:{service_account}
  → Plus API key: validated via POST /auth/validate-api-key

Delegation:
  → delegation:{delegation_id}#actor@user:{actor}
  → delegation:{delegation_id}#target@user:{target}
  → delegation:{delegation_id}#scope@permission:{perm}

Team:
  → team:{team_id}#member@user:{user}
  → team:{team_id}#parent@org:{org}
  → Permissions: team:{team_id}#can_{action}@...
```

Permission checks walk the parent chain recursively:
```
check(user:alice, read, organization:deep_child)
  → alice is member of deep_child
  → deep_child has parent dept
  → dept has parent division
  → division has parent root
  → root has permission read
  → ALLOW (if permission is default_grant at org level)
```

Same model regardless of archetype. The only difference is chain depth and
who creates each level.


## Implementation Plan

### Phase 1: Schema + Documentation (Current — v1.1)
- Define `org-structure.json` schema contract
- Create archetype examples (6 archetypes)
- This document
- Cross-cutting capabilities documented per archetype

**No code changes.** Current CLI creates the self-service pattern.

### Phase 2: CLI Archetype Selection
- Add `./setup init` — "What type of org structure?"
- Present 6 archetypes with descriptions
- Generate `config/org-structure.json` from selection
- Enable cross-cutting capabilities based on archetype defaults
- Modify step 04 to read org-structure.json

### Phase 3: Multi-Tier Provisioning + Service Accounts
- `./setup create-org --tier customer --slug acme --name "Acme Corp"`
- `./setup create-service-account --org acme --name "CI Bot" --permissions "..."`
- SSO configuration per org: `./setup configure-sso --org acme --idp okta`
- Admin UI integration

### Phase 4: Lifecycle Management
- Delegation management in CLI
- TTL support for dynamic archetype
- Org archival/soft-delete
- Cross-org user migration


## Modular Expansion Points

1. **New archetype file** in `config/archetypes/` — no side effects
2. **New tier types** — add entries to `tiers[]` array
3. **New creation modes** — add to `creation` enum
4. **New isolation flags** — add to `isolation` object
5. **New cross-cutting capabilities** — add top-level schema field
6. **Deeper nesting** — already supported, no schema change needed

### What Would Trigger a Schema v2?

- A use case that can't be expressed as a tree (graph-based org relationships)
- Cross-org permission sharing requiring new Zanzibar relationship types
- Multi-service org federation (platform-level, not per-service)

None of these are on the roadmap.


## FAQ from PMM Perspective

**Q: Can a service change archetypes after setup?**
A: Not easily. The org structure is baked into Zanzibar relationships. Migrating
from self-service to B2B would require creating new customer orgs and moving
users. Choose the right archetype at setup time.

**Q: Can a service use multiple archetypes?**
A: No. One service, one org structure. If a service needs both B2B isolation
AND self-service signup, use B2B with `signup_enabled: true` on customer orgs.

**Q: What if none of the archetypes fit?**
A: The tier model is flexible. Create a custom `org-structure.json` with the
tiers you need. Archetypes are starting points, not constraints. The org
hierarchy chains arbitrarily deep via `parent_id`.

**Q: Can we add service accounts to any archetype?**
A: Yes. Service accounts are cross-cutting — set `service_accounts.enabled: true`
on any archetype. They're created via `POST /admin/users/create-service-account`
with org-scoping and auto API key generation.

**Q: How does corporate differ from departmental?**
A: Depth and breadth. Departmental is 2 tiers with hard walls. Corporate is
N tiers with SSO, delegation, service accounts at multiple levels, and teams.
If you need more than root → one level of children, use corporate.

**Q: Does the hierarchy have a depth limit?**
A: No hard limit. `parent_id` chains arbitrarily. The visualization endpoint
supports `max_depth: 10` but that's a query parameter, not a system limit.

**Q: Can we add a 7th archetype later?**
A: Yes. Add a new file in `config/archetypes/`. No changes to existing files
or schema version.
