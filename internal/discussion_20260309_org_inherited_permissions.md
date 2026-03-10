# Discussion: Org-Inherited Permissions — What's Broken and Why It Matters

**Date:** 2026-03-09
**Files investigated:** zanzibar_permission_resolver.py, script 09, script 01, Zanzibar client/repository/models, zanzibar_sync.py, integration.py
**Branch:** feature/schema-branch-01

---

## The Business Problem

Script 04 (team-inherited) works today. Script 09 (org-inherited) does not.

Both scripts set up a child org for end-users. The difference is HOW default permissions flow to new users:

| | Script 04 (team-inherited) | Script 09 (org-inherited) |
|---|---|---|
| **How it works** | Creates "Default Users" team → grants permissions to team → new users auto-join team → inherit via team membership | Creates child org → grants permissions to org → new users join org → inherit via org membership |
| **Setup complexity** | More moving parts (team + login config with default_team) | Simpler (just org + permissions) |
| **Maintenance** | Must keep team in sync with login config | Zero maintenance — org membership is automatic |
| **Works today?** | Yes | No |

**Why does it matter?** The org-inherited model is the design goal stated in INTENT.txt — "zero-config, zero-maintenance." The team model works but adds a team abstraction that only exists to carry permissions.

---

## Root Cause Analysis (3 gaps)

### Gap 1: Resolver doesn't walk org memberships

**File:** `appv2/services/authz/zanzibar_permission_resolver.py` line 146

```python
# CURRENT — only walks teams
team_rels = [
    r for r in relationships
    if r.relation == "member" and r.object.namespace in ("team", "group")
]
```

When a user joins an org, the Zanbibar sync handler writes 3 records (see `zanbibar_sync.py:301-332`):
- Record 1: `object=role:{org_id}, relation={role}` — for role resolution
- Record 2a: `object=organization:{org_id}, relation={role}` — queryable by role
- Record 2b: `object=organization:{org_id}, relation=member` — uniform membership

Record 2b exists specifically so that `relation == "member"` filters find ALL org users. But the resolver ignores namespace `"organization"` — it only looks at `"team"` and `"group"`.

**Fix:** Add `"organization"` to the namespace filter (or add a separate step 3b). The rest of the resolution logic already works — it creates `SubjectReference(type=rel.object.namespace, id=rel.object.object_id)` and queries `get_permissions_for_subject()`. For `namespace="organization"` this produces `SubjectReference(type="organization", id=org_id)` — correct.

**Effort:** ~15 lines of code.

### Gap 2: No way to grant permissions to an org subject

**What script 09 does today:**
```bash
curl -X POST "$AUTH_SERVICE_URL/permissions/grant?user_id=$ADMIN_USER_ID&org_id=$END_USERS_ORG_ID&permission=$PERM"
```

This grants to `subject=user:{admin_id}`. Only the admin gets the permissions — NOT all org members.

**What it needs to do:**
Grant to `subject=organization:{org_id}` so all members inherit via the resolver (once Gap 1 is fixed).

**Two APIs exist:**

| Endpoint | Subject support | Auth required |
|---|---|---|
| `POST /permissions/grant?user_id=&org_id=&permission=` | user only (hardcoded `user_id` param) | `users.write` |
| `POST /zanzibar/stores/{store_id}/permissions/grant` | any subject (`"organization:xxx"`, `"team:xxx"`, etc.) | `zanbibar.admin` |

The Zanzibar endpoint supports org subjects, but requires `zanbibar.admin` — which the service admin doesn't have by default.

**Good news:** `_validate_store_access()` allows parent-org admins to access child-org stores (line 309: `is_ancestor_of`). So the service admin (parent org) can grant on the end-users org (child).

**Options:**
1. Grant `zanbibar.admin` to service admin in script 01, then script 09 uses the Zanbibar endpoint
2. Extend `/permissions/grant` to accept an optional `subject_type` parameter alongside `user_id`
3. Both

**Decision needed:** Which approach? Option 1 is a script-only change. Option 2 is an API change.

### Gap 3: Missing pydantic validation for permissions.json

Scripts 01, 04, and 09 all parse `permissions.json` with raw `jq` calls. If a required field is missing or malformed, the script silently produces empty values and fails downstream with confusing errors.

**Example:** If `permissions.json` is missing `.service.id`, script 01 sets `SERVICE_ID=""` and creates an org with an empty slug.

**Options:**
1. Add a `validate-config.py` script that checks the schema before any setup step runs
2. Add jq validation guards in each script
3. Both

---

## Business Questions to Decide

### Q1: Do we need org-inherited permissions at all?

Script 04 (team-inherited) works today. Is the simpler org-inherited model worth fixing?

**Arguments for:**
- INTENT.txt design goal: "zero-config, zero-maintenance"
- Removes the artificial "Default Users" team that exists only to carry permissions
- Simpler mental model for SaaS clients ("your users join your org and get permissions")
- Some org archetypes (from DISCUSSION.md) don't naturally map to teams

**Arguments against:**
- Script 04 works now — shipping matters
- Team model gives more flexibility (multiple teams with different permission sets)
- Resolver change touches a critical auth path — needs careful testing

### Q2: Should script 09 use the Zanbibar API or should we extend the flat grant API?

**Zanbibar API (option 1):**
- Pro: Already supports any subject type, no API change needed
- Con: Requires granting `zanbibar.admin` to every service admin — increases privilege surface

**Extend flat grant API (option 2):**
- Pro: Consistent with existing setup scripts, no new permission needed
- Con: API change, needs backward compat, more code to write and test

**Both (option 3):**
- Most flexible long-term, but more work now

### Q3: Where should permissions.json validation live?

**Standalone script (`validate-config.py`):**
- Pro: Can be run independently, reusable
- Con: Another file to maintain, clients might skip it

**Built into the `setup` CLI:**
- Pro: Automatic, can't be skipped
- Con: Adds Python dependency to a bash-based CLI

**jq guards in each script:**
- Pro: No new files, fail-fast at point of use
- Con: Duplicated validation, inconsistent error messages

---

## What Exists Today (for reference)

There's already an implementation plan from 2026-03-07:
- `appv2/tickets/20260307_team_permission_inheritance_fix/IMPLEMENTATION_PLAN_org_inherited_permissions.md`
- `appv2/tickets/20260307_team_permission_inheritance_fix/tasklist_org_permission_path_20260307.md`
- `appv2/tickets/20260307_team_permission_inheritance_fix/root_cause_org_team_inconsistency_20260307.md`

These docs identified the same 3 gaps. The resolver fix was planned but not implemented.

---

## Proposed Changes (pending discussion)

| Change | File | Risk | Effort |
|---|---|---|---|
| Resolver: add org namespace walking | zanbibar_permission_resolver.py | Medium — critical auth path | ~15 lines |
| Script 01: grant zanbibar.admin to admin | 01-register-service-permissions.sh | Low — additive | ~5 lines |
| Script 09: use Zanbibar API for org grants | 09-setup-end-users-org-inherited.sh | Low — script only | ~20 lines |
| Validation: check permissions.json schema | new: validate-config.py or in setup CLI | Low — additive | ~50 lines |
| Update script 09 TODO/comments | 09-setup-end-users-org-inherited.sh | None | ~10 lines |
