# Auth Mesh — Product-Market Assessment

**Date:** 2026-03-07
**Author:** Platform PMM
**Classification:** Internal Strategy Document
**Version:** 1.0

---

## Executive Summary

Auth Mesh is a multi-tenant identity, authentication, and authorization platform
built on Google Zanzibar's relation-based access control (ReBAC) model. It
provides 171 API endpoints across 26 functional domains, serving as the
foundational identity layer for any application that needs users, permissions,
and organizational structure.

This document assesses Auth Mesh through PMM frameworks used at
Google/Meta/OpenAI/Anthropic: Jobs-to-Be-Done (JTBD), market category
definition, competitive positioning, user segmentation, value metric analysis,
and monetization strategy. The goal is to map the product's current capabilities
against user needs and market opportunity — and identify where the product has
latent value that isn't yet surfaced, packaged, or priced.

---

## Part 1: Market Category & Positioning

### 1.1 Category Definition

Auth Mesh occupies the intersection of three established markets:

| Market | Size (2025) | Growth | Key Players |
|--------|------------|--------|-------------|
| **Identity & Access Management (IAM)** | $18B | 13% CAGR | Okta, Auth0, Microsoft Entra |
| **Customer Identity (CIAM)** | $10B | 15% CAGR | Auth0, Cognito, Firebase Auth |
| **Authorization / Policy Engine** | $2B | 25% CAGR | Oso, Cerbos, Permit.io, SpiceDB |

Most products occupy one of these. Auth Mesh is the rare product that spans
all three — it does identity (authn), fine-grained authorization (authz via
Zanzibar), AND organizational structure (multi-tenancy). This is the
**"full-stack auth"** category.

### 1.2 Category of One: Why This Matters

The typical developer building a SaaS product today needs:

1. **Auth0** or **Clerk** for login (identity)
2. **Oso** or **Cerbos** for authorization (who can do what)
3. **Custom code** for multi-tenancy (orgs, teams, hierarchy)
4. **Custom code** for service accounts and m2m auth
5. **Custom code** for delegation and impersonation
6. **Traefik/Envoy middleware** for edge auth

Auth Mesh replaces all six with a single platform. This is the
**compound product** strategy (cf. Rippling's thesis: compound > point solution
when the integration surface area is high).

### 1.3 Positioning Statement (Geoffrey Moore Framework)

> **For** development teams building multi-tenant applications
> **Who** need authentication, authorization, and organizational hierarchy
> **Auth Mesh is** a full-stack identity and permission platform
> **That** provides Zanzibar-grade authorization with zero-config user permissions
> **Unlike** Auth0 (identity only), Oso (authorization only), or Cognito (basic auth)
> **Auth Mesh** eliminates the integration tax of combining multiple auth vendors
> and removes the maintenance burden of custom multi-tenancy code.

---

## Part 2: User Understanding

### 2.1 User Segments (Pragmatist Adoption Model)

Using the Technology Adoption Lifecycle (Moore's Crossing the Chasm):

| Segment | Who | What They Need | Where Auth Mesh Fits |
|---------|-----|---------------|---------------------|
| **Solo Dev / Indie Hacker** | One person building a SaaS | "Just make login work" | Self-service archetype. Zero-config. |
| **Startup Engineering Team** | 3-15 devs, Series A/B | Multi-tenant, teams, API keys, RBAC | B2B multi-tenant archetype. Growing into corporate. |
| **Platform Team at Scaleup** | 50+ devs, multiple services | Centralized auth, service accounts, forward auth | Corporate archetype. Service mesh integration. |
| **Enterprise IT / Security** | CISO, compliance officers | SSO, audit, delegation, JWKS rotation | Corporate with SSO + delegation + audit. |
| **Reseller / White-Label** | ISVs selling through partners | Permission ceilings, per-reseller branding | Reseller archetype. Multi-tier hierarchy. |
| **AI / Agent Platform** | Agent orchestration teams | Scoped, ephemeral, TTL-based identities | Dynamic archetype. API-driven lifecycle. |

### 2.2 Jobs-to-Be-Done (JTBD) Analysis

JTBD framework (Christensen/Ulwick): users don't buy products, they hire
them to make progress. Here are the core jobs:

#### Job 1: "Help me not think about auth"
**Functional:** I want login, registration, password reset, and token management
to just work so I can build my actual product.

**Emotional:** I'm anxious that I'll build auth wrong and get hacked. I want to
feel confident that security is handled by experts.

**Social:** I want my investors/customers to see a professional login page, not
a janky custom form.

**Auth Mesh fulfills this via:** Hosted login pages (`/login/{org_slug}`),
OAuth 2.1 with PKCE, email templates, password policy enforcement. The setup
CLI gets a service from zero to working auth in minutes.

**Satisfaction metric:** Time from `git init` to first successful user login.

#### Job 2: "Help me organize my users"
**Functional:** I need orgs, teams, roles. My B2B customers each need their own
workspace. Users need different permissions.

**Emotional:** I dread building multi-tenancy from scratch. Every time, it's
the same spaghetti of org_id filters and permission checks.

**Auth Mesh fulfills this via:** Arbitrarily deep org hierarchy via `parent_id`,
Zanzibar permission inheritance, teams within orgs, invitation system, org
switching.

**Satisfaction metric:** Can I model my customer's org structure without custom code?

#### Job 3: "Help me secure my services without changing them"
**Functional:** I have 10 microservices. I need auth in front of all of them.
I don't want to modify each one.

**Auth Mesh fulfills this via:** Forward auth (`/forward-auth/`), API key
validation (`/auth/validate-api-key`), token validation
(`/auth/validate-token`), Traefik integration.

**Satisfaction metric:** Can I protect a new service with zero application code?

#### Job 4: "Help me automate without security tradeoffs"
**Functional:** CI/CD pipelines, monitoring bots, ETL jobs need to
authenticate. I can't use human credentials.

**Auth Mesh fulfills this via:** Service accounts
(`/admin/users/create-service-account`), API keys with granular permissions,
`client_credentials` OAuth grant, password rotation exemption.

**Satisfaction metric:** Can my bot get exactly the permissions it needs and
nothing more?

#### Job 5: "Help me satisfy compliance without a compliance team"
**Functional:** SOC 2, HIPAA, GDPR. I need audit trails, delegation records,
JWKS rotation, session management, account lockout.

**Auth Mesh fulfills this via:** Event subscriptions (23 event types across 6
categories), delegation audit trail, super-admin with approval workflow,
JWKS rotation, password compliance reports, session management per org.

**Satisfaction metric:** Can I generate an audit report without building a
system?

#### Job 6: "Help me let my customers bring their own identity"
**Functional:** Enterprise customers insist on SSO. I need to support Okta,
Azure AD, ADFS without building a SAML implementation.

**Auth Mesh fulfills this via:** Per-org SAML SP
(`/organizations/{org_slug}/auth/sso/initiate`), JIT provisioning from SAML
assertions, 8 supported provider types (internal, Google, GitHub, Microsoft,
Okta, Auth0, Cognito, Keycloak).

**Satisfaction metric:** How many hours to add SSO to an existing integration?

### 2.3 User Journey Map

```
AWARENESS          EVALUATION           ADOPTION             EXPANSION           ADVOCACY
─────────────────────────────────────────────────────────────────────────────────────────
"I need auth"      "Does it do          "Setup CLI runs,     "We need SSO for    "Every new service
                    multi-tenant?"       first login works"   enterprise client"   uses Auth Mesh"

Discovery:         Proof of concept:    Integration:         Feature expansion:   Platform standard:
- /help endpoint   - /openapi.json      - setup CLI          - SSO per org        - Forward auth
- /status          - Hosted login page  - Permissions.json   - Service accounts   - Event subscriptions
- Quota tiers      - Register + login   - OAuth client       - Delegation         - Metrics/observability
                   - Create org + team  - Hosted login       - Teams              - Multi-service estate
```

### 2.4 Anxiety & Friction Points (Demand-Side Forces)

Using the Four Forces of Progress (Moesta/JTBD):

**Forces pushing toward Auth Mesh:**
1. **Push of current situation:** "I've been building custom auth for 6 months
   and it's still not SOC 2 compliant"
2. **Pull of new solution:** "Zero-config permissions through org hierarchy —
   I never have to write a permission sync cron"

**Forces pushing away from Auth Mesh:**
3. **Anxiety of new solution:** "What if Auth Mesh goes down? Auth is
   mission-critical" → Mitigate with: forward-auth caching, JWKS local
   validation, health endpoints, circuit breakers
4. **Allegiance to current behavior:** "We already use Auth0 for login" →
   Auth Mesh can coexist: use Auth0 as a provider via OAuth federation,
   Auth Mesh as the authorization + org layer

---

## Part 3: Feature Inventory & Value Map

### 3.1 Feature Taxonomy (Kano Model)

The Kano Model categorizes features by their effect on user satisfaction:

#### Must-Be (Expected — absence causes dissatisfaction, presence doesn't delight)

| Feature | Endpoints | Status |
|---------|----------|--------|
| User registration | `POST /auth/register` | Shipped |
| Login / logout | `POST /auth/login`, `POST /auth/logout` | Shipped |
| Password reset | `POST /auth/password-reset` | Shipped |
| Token refresh | `POST /auth/refresh`, `POST /token/refresh` | Shipped |
| Token validation | `POST /auth/validate-token` | Shipped |
| JWT with RS256 | `GET /.well-known/jwks.json` | Shipped |
| Password policy | Configurable per-deployment | Shipped |
| Rate limiting | Built-in with tier limits | Shipped |
| Health check | `GET /health` | Shipped |

#### Performance (More is better — linear satisfaction increase)

| Feature | Endpoints | Status |
|---------|----------|--------|
| Organizations (CRUD) | `POST /organizations/`, etc. | Shipped |
| Org hierarchy (deep nesting) | `parent_id` chains, visualization | Shipped |
| Permissions (grant/check/revoke) | `/permissions/*` | Shipped |
| Teams (within-org grouping) | `/teams/*` | Shipped |
| API keys (CRUD, org-scoped) | `/api-keys/*` | Shipped |
| Service accounts | `POST /admin/users/create-service-account` | Shipped |
| Invitations | `POST /organizations/{id}/invite` | Shipped |
| Multi-workspace (org switching) | `POST /auth/switch-organization` | Shipped |
| OAuth 2.1 (PKCE, client reg) | `/auth/oauth/*` | Shipped |
| Quota tiers (4 tiers) | `/quotas/*` | Shipped |
| Event subscriptions | `/events/*`, 23 event types | Shipped |

#### Attractive (Unexpected delight — presence creates disproportionate satisfaction)

| Feature | Endpoints | Status |
|---------|----------|--------|
| Zanzibar ReBAC engine | `/zanzibar/stores/*` | Shipped |
| Forward auth (edge proxy) | `/forward-auth/` (Traefik) | Shipped |
| Delegation (act-on-behalf) | `/delegation/*` | Shipped |
| Hosted login (white-label) | `/login/{org_slug}` | Shipped |
| Per-org SAML SSO with JIT | `/organizations/{slug}/auth/sso/*` | Shipped |
| `client_credentials` grant | `POST /auth/oauth/token` | Shipped |
| JWKS rotation with recovery | `/admin/jwks/*` | Shipped |
| Super admin (with approval) | `/super-admin/*` | Shipped |
| Prometheus metrics | `GET /metrics` | Shipped |
| Hierarchy visualization (D3) | `/zanzibar/stores/{id}/visualize/*` | Shipped |
| Per-org email templates | `/organizations/{id}/emails/*` | Shipped |
| Circuit breaker management | `/admin/circuit-breakers/*` | Shipped |
| 8 auth provider types | Internal, Google, GitHub, Microsoft, Okta, Auth0, Cognito, Keycloak | Shipped |

### 3.2 Feature Surface Area

```
Total API endpoints:        171
Functional domains (tags):   26
Auth provider types:          8
Event types:                 23 (across 6 categories)
Quota tiers:                  4 (free, starter, professional, enterprise)
OAuth grant types:            3 (authorization_code, refresh_token, client_credentials)
Response modes:               3 (query, fragment, form_post)
```

This is an unusually large surface area for an auth platform. For comparison:
- Auth0 Management API: ~120 endpoints
- Firebase Auth: ~15 endpoints
- Clerk: ~60 endpoints

Auth Mesh's 171 endpoints span identity + authorization + org management +
observability — the compound product thesis in practice.

---

## Part 4: Competitive Analysis

### 4.1 Market Map (Ansoff Matrix Perspective)

```
                    EXISTING USERS              NEW USERS
                 ┌─────────────────────┬──────────────────────┐
EXISTING         │ Market Penetration  │ Market Development   │
PRODUCTS         │ - Deepen setup CLI  │ - Packaged archetypes│
                 │ - Better onboarding │ - Industry verticals │
                 │ - Self-serve docs   │ - Partner channel    │
                 ├─────────────────────┼──────────────────────┤
NEW              │ Product Development │ Diversification      │
PRODUCTS         │ - Consent mgmt     │ - Auth-as-a-Service  │
                 │ - Billing integration│ - Identity marketplace│
                 │ - Compliance reports│                      │
                 └─────────────────────┴──────────────────────┘
```

### 4.2 Competitive Positioning Matrix

```
                         Full-Stack Auth (identity + authz + orgs)
                                      ▲
                                      │
                                      │  Auth Mesh ●
                                      │
                                      │
                    WorkOS ●          │          ● Ory
                                      │
             Clerk ●                  │
                                      │
                     Auth0 ●          │     ● SpiceDB
                                      │
   Firebase Auth ●                    │        ● Cerbos
                                      │
    Simple ◄──────────────────────────┼────────────────────► Fine-Grained
    (login only)                      │               (ReBAC / policy engine)
                                      │
                 Cognito ●            │    ● Permit.io
                                      │
                                      │    ● Oso
                                      │
                                      ▼
                            Point Solution (one layer)
```

### 4.3 Feature Parity Table

| Capability | Auth Mesh | Auth0 | Clerk | WorkOS | SpiceDB | Ory |
|-----------|----------|-------|-------|--------|---------|-----|
| Email/password login | Yes | Yes | Yes | Yes | No | Yes |
| Social login (Google, GitHub, etc.) | Yes (8 providers) | Yes | Yes | Yes | No | Yes |
| SAML SSO | Yes (per-org) | Enterprise | Yes | Yes (core product) | No | Partial |
| Multi-tenant orgs | Yes (deep hierarchy) | Flat | Basic | Yes | No | No |
| Teams within orgs | Yes | No | Yes | No | No | No |
| ReBAC / Zanzibar | Yes (full engine) | No | No | No | Yes (core) | Partial (Keto) |
| Forward auth (edge proxy) | Yes (Traefik) | No | No | No | No | Yes |
| Service accounts | Yes (first-class) | M2M tokens | No | No | No | API keys |
| Delegation (act-as) | Yes (scoped, TTL) | No | Impersonation | No | No | No |
| API key management | Yes (CRUD, org-scoped) | Yes | Yes (basic) | Yes | No | Yes |
| Hosted login pages | Yes (per-org branded) | Yes | Yes (components) | Yes | No | No |
| OAuth 2.1 with PKCE | Yes | Yes | Yes | No | No | Yes |
| client_credentials grant | Yes | Yes | No | No | No | Yes |
| Event webhooks | Yes (23 types) | Yes | Yes | Yes | No | Partial |
| JWKS rotation | Yes (auto + manual) | Managed | Managed | N/A | N/A | Yes |
| Quota / rate limiting | Yes (4 tiers) | Yes | Yes | Yes | No | Yes |
| Prometheus metrics | Yes | No (dashboard) | No (dashboard) | No | Yes | Yes |
| Setup CLI | Yes | Terraform | No | No | No | CLI |
| Self-hosted | Yes | No | No | No | Yes | Yes |
| Zero-config permissions | Yes (org hierarchy) | No | No | No | No | No |

### 4.4 Unique Differentiators (Defensible Moats)

1. **Zanzibar + Identity in one platform** — SpiceDB has Zanzibar but no
   identity. Auth0 has identity but no Zanzibar. Auth Mesh has both. This
   eliminates the integration seam between "who is this user" and "what can
   this user do."

2. **Zero-config permission model** — Permissions flow from org membership
   via Zanzibar hierarchy. No per-user grants. No cron jobs. No callbacks.
   This is architecturally unique — no competitor offers this.

3. **Arbitrarily deep org hierarchy** — Most competitors support flat orgs
   (Auth0) or one level of nesting (Clerk). Auth Mesh's `parent_id` chains
   as deep as needed, with visualization up to depth 10.

4. **Per-org everything** — SSO, hosted login, email templates, teams,
   session management — all scoped per org. This enables true white-label.

5. **Self-hosted with full feature set** — Unlike Auth0/Clerk (cloud-only),
   Auth Mesh can run in your infrastructure. For regulated industries
   (healthcare, government, finance), this is table stakes.

---

## Part 5: Monetization Strategy

### 5.1 Current Pricing Architecture (Quota Tiers)

From the live `/quotas/tiers` endpoint:

| Dimension | Free | Starter | Professional | Enterprise |
|-----------|------|---------|-------------|-----------|
| Orgs per user | 100 | 3 | 10 | Unlimited |
| Users per org | 5 | 25 | 100 | 10,000 |
| Teams per org | 2 | 10 | 50 | 500 |
| API keys per org | 10 | 50 | 200 | 1,000 |
| Permissions per user | 50 | 200 | 500 | 2,000 |
| Delegations per user | 3 | 10 | 25 | 100 |
| Storage (MB) | 100 | 1,000 | 5,000 | 50,000 |
| API rate limit/hr | 1,000 | 5,000 | 20,000 | 100,000 |

**Observation:** The free tier has 100 orgs per user but only 5 users per org.
This is inverted from most auth platforms (Auth0: unlimited users, 1 org on
free). This suggests Auth Mesh is optimized for platform builders who create
many small orgs (agent platforms, CI/CD, dynamic provisioning) rather than
single large orgs.

### 5.2 Value Metric Analysis (ProfitWell Framework)

The **value metric** is the unit that best correlates with the value a customer
receives. Getting this right is the single most important pricing decision.

| Candidate Metric | Correlation with Value | Scalability | Ease of Understanding |
|-----------------|----------------------|------------|----------------------|
| **Monthly Active Users (MAU)** | High | Linear | Easy |
| **Organizations** | Medium | Linear | Easy |
| **API calls** | Medium | Linear | Medium (hard to predict) |
| **Connected services** | High | Step function | Easy |
| **Zanzibar checks/month** | High | Linear | Hard (technical) |

**Recommended primary value metric: Connected Services** (how many services
use Auth Mesh for auth). This is:
- Easy to understand ("we have 8 services behind Auth Mesh")
- Correlated with value (more services = more value from centralized auth)
- Naturally expansive (grows as the customer's architecture grows)
- Hard to game (unlike MAU which can be optimized down)

**Recommended secondary metric: MAU** for tier thresholds within each
service count.

### 5.3 Monetization Opportunities (Feature-Gate Matrix)

#### Tier 1: Free (Developer / Indie Hacker)
**What they get:** Everything needed to build one SaaS product with basic auth.
- Email/password login
- 1 org with up to 5 users
- Hosted login page
- Basic permissions (50 per user)
- OAuth 2.1 with PKCE
- API key management (10 keys)

**Why free:** Developer adoption is the top of funnel. Auth decisions are sticky
— once you build on an auth platform, switching costs are enormous (every token,
every permission, every org structure has to be migrated). The free tier is
customer acquisition.

**Unit economics:** Low cost to serve. Auth workloads are lightweight
(small payloads, cacheable responses, stateless validation via JWT).

#### Tier 2: Starter ($29-79/mo per connected service)
**Expansion trigger:** "We need more than 5 users" or "we added a second service"
- Up to 25 users per org
- Up to 3 orgs
- 10 teams
- 50 API keys
- Event subscriptions
- 5,000 API calls/hr

**Why this gate:** The moment a product has real users (>5), it has revenue.
It can pay for auth. The price is trivial relative to the cost of building auth
yourself (2-4 engineer-months).

#### Tier 3: Professional ($199-499/mo per connected service)
**Expansion trigger:** "Enterprise customer wants SSO" or "we need service accounts"
- Up to 100 users per org
- 10 orgs (B2B multi-tenant)
- **SSO/SAML per org** (this is the key gate)
- **Service accounts**
- **Delegation**
- 50 teams
- 200 API keys
- 20,000 API calls/hr
- Prometheus metrics
- JWKS rotation management

**Why this gate:** SSO is the classic enterprise expansion trigger. When a
startup's first enterprise customer says "we need SAML," that startup will pay
for SSO. Auth0 charges $23,000/yr for enterprise SSO. Auth Mesh at $499/mo is
a compelling alternative.

#### Tier 4: Enterprise (Custom pricing)
**Expansion trigger:** "We need unlimited scale" or "compliance requires self-hosted"
- Unlimited orgs and users
- **Forward auth** (edge proxy integration)
- **Super admin with approval workflow**
- **Zanzibar visualization**
- **Full audit trail / compliance reports**
- **Custom email templates per org**
- **Circuit breaker management**
- **Dedicated support**
- Self-hosted deployment option
- 100,000 API calls/hr

**Why this gate:** Enterprise customers expect custom pricing and dedicated
support. The forward auth and audit features are enterprise-specific — a
startup with 3 services doesn't need them, but a platform team managing 30
services does.

### 5.4 Revenue Expansion Levers (Land & Expand)

```
LAND                        EXPAND                      COMPOUND
──────────────────────────────────────────────────────────────────
Free tier                   "We need SSO"               Auth becomes platform
1 service, 5 users          → Professional              standard across all
                                                        services in the org

Solo dev builds MVP         "Enterprise client           Every new microservice
with Auth Mesh              wants SAML"                  uses Auth Mesh
                            → Professional               forward auth

Startup picks Auth Mesh     "We added 4 more            Auth Mesh is the
over Auth0                  microservices"               identity layer
                            → Per-service pricing         for the company
                            → Revenue grows linearly
```

### 5.5 Add-On Monetization (Feature Modules)

These are features that could be sold as add-ons on top of any tier:

| Add-On | What It Is | User Need | Pricing Model |
|--------|-----------|-----------|---------------|
| **Compliance Pack** | Pre-built SOC 2 / HIPAA reports from event data, password compliance reports, audit exports | "My compliance officer needs a report" | $99-299/mo |
| **Advanced SSO** | SAML attribute mapping, multiple IdPs per org, conditional access policies | "Each department has a different IdP" | $149-499/mo per org |
| **Delegation Pro** | Extended delegation (>24hr), delegation chains, delegation analytics | "Support team needs persistent impersonation" | $49-149/mo |
| **Observability Pack** | Enhanced Prometheus metrics, Grafana dashboards, alert rules, anomaly detection | "SRE team needs auth-specific dashboards" | $99-199/mo |
| **Event Streaming** | Real-time event firehose (beyond webhook subscriptions), data lake export, replay | "We need auth events in our analytics pipeline" | Usage-based |
| **White-Label Pack** | Custom domains for hosted login, full CSS control, custom email branding, remove Auth Mesh branding | "Our customers can't see 'Auth Mesh'" | $199-499/mo |
| **Migration Toolkit** | Auth0/Cognito/Firebase import scripts, user migration with zero downtime, token compatibility | "We're switching from Auth0" | One-time $999-2,999 |

---

## Part 6: Market Timing & Macro Trends

### 6.1 Why Now? (Sequoia "Why Now?" Framework)

Every significant product rides a structural change. Auth Mesh rides several:

1. **Microservice proliferation** — The average company now runs 20-50
   microservices. Each needs auth. Centralized auth (forward auth pattern)
   is no longer optional — it's infrastructure. Auth Mesh's forward auth
   endpoint is purpose-built for this.

2. **AI agent identity crisis** — AI agents need scoped, ephemeral
   identities. Traditional auth (username/password) doesn't fit. Auth Mesh's
   dynamic archetype (API-created orgs with TTL, service accounts with
   granular permissions) is exactly what agent platforms need. This market
   didn't exist 18 months ago.

3. **Zero-trust is table stakes** — Every request must be authenticated and
   authorized. "Trust the network" is dead. Zanzibar-style continuous
   authorization (check on every request, not just at login) is the new
   baseline. Auth Mesh's `/permissions/check` and `/forward-auth/` endpoints
   implement this natively.

4. **SOC 2 / compliance pressure on startups** — Series A companies are now
   expected to be SOC 2 compliant. Auth is the #1 audit finding area. Auth Mesh's
   event system, JWKS rotation, password policy, and delegation audit trail
   are compliance infrastructure out of the box.

5. **Self-hosted demand from regulated industries** — Healthcare (HIPAA),
   finance (PCI DSS), government (FedRAMP) can't use cloud-only auth. Auth0
   dropped self-hosted in 2023. Auth Mesh fills this gap.

6. **Developer experience is the distribution channel** — The best dev tools
   win by being delightful to use. Auth Mesh's `/help` endpoint, setup CLI,
   `.example` schema files, and archetype system are developer-experience
   investments that reduce time-to-value.

### 6.2 Technology S-Curves

```
MATURITY ▲
         │                                          ┌── Zanzibar/ReBAC
         │                                    ┌─────┘   (early majority)
         │                              ┌─────┘
         │          ┌───────────────────┘
         │    ┌─────┘ OAuth 2.1 / OIDC
         │    │       (mature, commodity)
         │    │
         │────┘
         │ SAML
         │ (legacy, still required for enterprise)
         └──────────────────────────────────────────► TIME

Auth Mesh rides the Zanzibar curve while supporting the mature OAuth/SAML curves.
The Zanzibar model is in the early majority phase — Google, Airbnb, and
Carta have shipped Zanzibar internally, but few commercial products offer it.
SpiceDB is the closest, but it's authorization-only (no identity).
```

---

## Part 7: User Happiness & Usefulness Assessment

### 7.1 The Hierarchy of User Needs (Maslow for Products)

```
                    ┌─────────────┐
                    │ DELIGHT     │  Visualization, developer UX,
                    │             │  setup CLI, /help endpoint
                    ├─────────────┤
                    │ EFFICIENCY  │  Forward auth (zero code),
                    │             │  zero-config permissions,
                    │             │  service accounts
                    ├─────────────┤
                    │ CAPABILITY  │  SSO, delegation, Zanzibar,
                    │             │  deep hierarchy, teams,
                    │             │  event subscriptions
                    ├─────────────┤
                    │ RELIABILITY │  Circuit breakers, JWKS recovery,
                    │             │  health endpoints, idempotent APIs,
                    │             │  rate limiting
                    ├─────────────┤
                    │ FUNCTION    │  Login, register, tokens,
                    │             │  password reset, permissions
                    └─────────────┘
```

**Assessment:** Auth Mesh is strong at all five levels. Most auth products
are strong at Function and Reliability but weak at Capability and Efficiency.
Auth Mesh's zero-config permission model (Efficiency) and Zanzibar engine
(Capability) are where it pulls ahead.

### 7.2 Time-to-Value Analysis

| User Journey | Without Auth Mesh | With Auth Mesh | Multiplier |
|-------------|-------------------|----------------|-----------|
| Add login to a new app | 2-4 weeks | 1 day (setup CLI) | 10-20x |
| Add multi-tenant orgs | 4-8 weeks | 0 (built-in) | Infinite |
| Add SSO for enterprise customer | 2-4 weeks | 1 hour (per-org SSO config) | 80-160x |
| Add permissions to a service | 1-2 weeks | 1 hour (permissions.json) | 40-80x |
| Add auth to a new microservice | 3-5 days | 0 (forward auth) | Infinite |
| Add service accounts for CI/CD | 1-2 weeks | 10 minutes | 40-80x |
| Add delegation for support team | 2-3 weeks | 5 minutes | 120-180x |
| Generate compliance audit report | 1-2 weeks (custom) | Built-in (events API) | 40-80x |

### 7.3 User Happiness Drivers (NPS Prediction Model)

**Promoter triggers** (would cause users to recommend Auth Mesh):
- "I added auth to a new service with zero code changes" (forward auth)
- "Enterprise customer asked for SSO and I had it working in an hour"
- "I never have to think about permission sync — it just works"
- "The setup CLI got everything working on the first try"
- "The /help endpoint taught me the API without reading docs"

**Detractor triggers** (would cause users to warn others):
- "I couldn't figure out how org hierarchy maps to my business"
  → Mitigation: archetype system, setup CLI onboarding flow
- "I got locked out and couldn't recover"
  → Mitigation: JWKS recovery, circuit breaker reset, super admin
- "The API returned a cryptic error"
  → Mitigation: structured error responses, /help endpoint
- "I needed a feature that required Enterprise tier"
  → Mitigation: transparent pricing, generous free tier

### 7.4 Utility Function (Usefulness Over Time)

```
Usefulness ▲
           │                              ●────────── Platform standard
           │                        ●─────            (all services use it)
           │                  ●─────
           │            ●─────                         Compound value:
           │       ●────                               Each new service
           │    ●──                                    increases value of
           │  ●─                                       ALL existing services
           │ ●                                         (network effect within
           │●                                          the customer's infra)
           └──────────────────────────────────────────► Services connected
            1    2    3    5    10   20   50
```

Auth Mesh exhibits **increasing returns to scale within a customer's
infrastructure**. The 10th service connected is more valuable than the 1st
because:
- Centralized audit trail covers more surface area
- Forward auth eliminates per-service auth code at scale
- Org hierarchy is shared across all services
- Permission schema is consistent (same roles, same grants everywhere)
- Service accounts use the same API keys across services

This is a **demand-side economy of scale** — the customer gets more value
the more they use it, creating natural expansion and lock-in.

---

## Part 8: Strategic Opportunities

### 8.1 Product-Led Growth (PLG) Motions

| Motion | Mechanism | Current Support | Gap |
|--------|-----------|----------------|-----|
| **Self-serve signup** | Free tier → setup CLI → working auth | Strong | Need cloud-hosted option |
| **Viral loop** | Hosted login pages show "Powered by Auth Mesh" | Possible | Not implemented |
| **Usage expansion** | Per-service pricing → natural growth | Designed for | Need metering |
| **Seat expansion** | More users → higher tier | Built in (tier limits) | Need in-app upgrade prompt |
| **Champion creation** | Developer falls in love → brings to next company | Strong (DX is good) | Need community |

### 8.2 Missing Features That Would Unlock New Segments

| Feature | Segment Unlocked | Revenue Impact | Build Effort |
|---------|-----------------|----------------|-------------|
| **Passwordless / WebAuthn** | Consumer apps, fintech | Medium (table stakes soon) | Medium |
| **MFA enforcement per org** | Healthcare, finance, government | High (compliance gate) | Medium |
| **Consent management** | GDPR-regulated markets (EU) | Medium | High |
| **Billing integration** | SaaS platforms (tie auth to billing) | High (sticky) | High |
| **Admin dashboard UI** | Non-technical admins | High (widens market) | High |
| **Terraform provider** | DevOps / IaC teams | Medium (distribution) | Medium |
| **SDK for more languages** | Broader developer adoption | Medium | Medium |
| **Org-level policy engine** | Enterprise security (conditional access) | High | High |
| **User directory sync (SCIM)** | HR-integrated enterprise | High | Medium |
| **Magic links** | Consumer apps, low-friction signup | Medium | Low |

### 8.3 Platform Thesis (Aggregation Theory Perspective)

Auth Mesh has the structural properties of a **platform**, not a tool:

1. **Multi-sided:** Services (demand side) and users (supply side) both
   connect through Auth Mesh. The more services connected, the more valuable
   the user's single identity becomes.

2. **Data gravity:** Once org structures, permission schemas, and user
   identities live in Auth Mesh, moving them is expensive. This creates
   natural retention.

3. **Standardization benefit:** When all services in an organization use
   Auth Mesh, they share a common identity and permission model. This
   reduces integration cost between services to zero.

4. **Extensibility:** Event subscriptions, webhooks, and the Zanzibar API
   allow third parties to build on top of Auth Mesh without modifying the
   core platform.

### 8.4 Build vs. Partner vs. Acquire Matrix

| Capability | Build | Partner | Acquire |
|-----------|-------|---------|---------|
| Admin Dashboard UI | Build (core product) | — | — |
| Billing integration | Partner (Stripe, Paddle) | Yes | — |
| Terraform provider | Build (open source) | — | — |
| SCIM / directory sync | Build (enterprise need) | — | — |
| Passwordless / WebAuthn | Build (already in enterprise module) | — | — |
| Consent management | Build or partner (OneTrust) | Maybe | — |
| Cloud hosting | Build (infrastructure) | Partner (AWS, GCP) | — |

---

## Part 9: Risk Assessment

### 9.1 Strategic Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **Auth0/Okta adds Zanzibar** | Low (architectural shift) | High | Move faster, build community |
| **Open-source competitor (Ory) catches up** | Medium | Medium | Differentiate on DX + compound product |
| **Cloud provider bundling (Cognito, Firebase)** | Medium | High | Self-hosted option + superior DX |
| **Security incident** | Low | Critical | Bug bounty, audit, SOC 2, penetration testing |
| **Complexity overwhelms users** | Medium | Medium | Archetype system, setup CLI, onboarding |

### 9.2 Technical Debt Risks

| Risk | Description | Mitigation |
|------|------------|-----------|
| **DynamoDB scaling** | Single-table design may hit hot partition limits at scale | Monitor, consider read replicas |
| **JWKS rotation** | Currently manual opt-in (`jwks_auto_rotation_enabled: false`) | Enable auto-rotation by default |
| **Enterprise module coupling** | SAML/WebAuthn in separate module may lag core features | Integration testing, feature flags |

---

## Part 10: Recommended Next Actions

### Immediate (0-30 days)
1. **Publish archetype documentation** — The org-structure schema and
   archetype examples are ready. Ship them as part of the setup CLI.
2. **Add "Powered by Auth Mesh"** to hosted login pages — Free viral loop.
3. **Create a landing page** with the competitive positioning matrix from
   Section 4.3.

### Short-term (30-90 days)
4. **Build per-service metering** — Required for the recommended pricing
   model (per connected service).
5. **Build admin dashboard UI** — The biggest market expansion lever. Most
   users are not CLI-comfortable.
6. **Implement MFA enforcement per org** — Unlocks healthcare and finance.

### Medium-term (90-180 days)
7. **Launch cloud-hosted option** — PLG requires self-serve. Can't expect
   every indie hacker to run infrastructure.
8. **Terraform provider** — DevOps distribution channel.
9. **SCIM directory sync** — Enterprise expansion trigger.

### Long-term (180-365 days)
10. **Developer community** — Docs site, Discord, blog, conference talks.
11. **Marketplace** — Pre-built permission schemas for common patterns
    (e-commerce, healthcare, fintech).
12. **Consent management** — GDPR compliance as a feature, not a burden.

---

## Appendix A: Auth Mesh Feature Map (Complete)

### Identity & Authentication (26 endpoints)
- User registration (with/without org)
- Login/logout (direct + org-scoped)
- Password reset (request/validate/confirm)
- Password change
- Token refresh/revoke/introspect
- Email verification
- OAuth 2.1 (authorize, token, PAR)
- OAuth client registration (dynamic, RFC 7591/7592)
- 8 social/enterprise providers
- SAML SSO (per-org initiate + ACS callback)

### Authorization (25 endpoints)
- Permission grant/revoke/check
- Permission registry (register, validate, stats)
- Role management
- User permission listing
- Zanzibar relationship CRUD
- Zanzibar check (single, bulk, wildcard)
- Zanzibar expand (permission graph traversal)
- Zanzibar list-objects / list-users
- Zanzibar hierarchy setup
- Zanzibar namespace management
- Zanzibar permission migration
- Zanzibar visualization (hierarchy + permissions)

### Organization Management (30 endpoints)
- Organization CRUD
- Org hierarchy (deep nesting via parent_id)
- Org user management (list, add, remove)
- Org invitations
- Org teams (CRUD, members, permissions)
- Org session management
- Org email configuration + templates
- Org login config (hosted login branding)
- Org-scoped auth (login, register, logout, refresh, token)
- Org-scoped SSO
- Org-scoped JWKS
- Org OAuth clients listing

### Service Infrastructure (25 endpoints)
- API key CRUD (org-scoped, permission-scoped)
- API key validation
- Service account creation
- Forward auth (GET/POST/HEAD + test endpoints)
- Token validation (JWT + API key)
- Event types listing
- Event subscription management (CRUD, toggle, test, stats)

### Admin & Operations (40+ endpoints)
- JWKS management (generate, rotate, activate, revoke, cleanup, recovery)
- Password policy management (global + per-org)
- Admin user management (elevate, service accounts)
- Super admin (grant, approve, revoke, extend, audit log)
- Circuit breaker management
- Provider management (CRUD, test, supported types)
- Email admin (config, history, stats, template types)
- Audit reports (password events, revocations, compliance)
- Security reports (dismiss, resolve)

### Observability (10 endpoints)
- Health check + JWKS health
- Service status
- Prometheus metrics
- Metrics alerts
- JWKS metrics
- Quota checking + usage

### Developer Experience (5 endpoints)
- OpenAPI spec
- Help guide (interactive)
- Enterprise help
- Well-known endpoints (JWKS, OAuth metadata, OIDC configuration)

---

## Appendix B: Glossary of PMM Frameworks Referenced

| Framework | Source | How Used |
|-----------|--------|----------|
| **Jobs-to-Be-Done (JTBD)** | Christensen / Ulwick | User need analysis (Part 2.2) |
| **Kano Model** | Noriaki Kano | Feature categorization (Part 3.1) |
| **Crossing the Chasm** | Geoffrey Moore | User segmentation (Part 2.1) |
| **Positioning Statement** | Geoffrey Moore | Market positioning (Part 1.3) |
| **Technology Adoption Lifecycle** | Rogers / Moore | Segment maturity (Part 2.1) |
| **Ansoff Matrix** | Igor Ansoff | Growth strategy (Part 4.1) |
| **Four Forces of Progress** | Moesta / JTBD | Adoption friction (Part 2.4) |
| **Value Metric** | ProfitWell / Price Intelligently | Pricing strategy (Part 5.2) |
| **Hierarchy of Needs** | Maslow / Product analogy | User satisfaction (Part 7.1) |
| **S-Curves** | Clayton Christensen | Technology timing (Part 6.2) |
| **Aggregation Theory** | Ben Thompson / Stratechery | Platform thesis (Part 8.3) |
| **Compound Product** | Parker Conrad / Rippling | Competitive positioning (Part 1.2) |
| **Why Now?** | Sequoia Capital | Market timing (Part 6.1) |
| **PLG** | OpenView / Wes Bush | Growth strategy (Part 8.1) |
| **Land & Expand** | David Sacks / Craft | Revenue expansion (Part 5.4) |
| **NPS Prediction** | Bain / Reichheld | Satisfaction drivers (Part 7.3) |

---

*This document is a living artifact. Update quarterly as the product and market evolve.*
