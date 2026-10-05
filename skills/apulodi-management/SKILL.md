---
name: apulodi-management
license: MIT
description: >
  Work with APULODI's management surface: organizations, projects, team
  invitations, roles, quotas, and API keys. Use when building admin or
  dashboard tooling for APULODI, automating tenant provisioning, creating or
  revoking project API keys programmatically, managing members and
  invitations, or answering questions about APULODI's multi-tenancy model
  (owner/admin/member roles and permission checks). This surface is
  session-cookie authenticated, unlike the developer file API which uses API
  keys.
---

# APULODI management surface

This is the **control-plane** API that powers the APULODI dashboard. It manages
tenancy — organizations, projects, members, quotas, and the API keys that
authenticate the developer file API (`/api/v1`, see `apulodi` skill).

```text
Organization  ← owner / admin / member roles
     └── Project  ← quotas, webhooks, files
            └── API keys  ← authenticate /api/v1
```

## Authentication: session cookie, NOT an API key

Every route below requires a **Better Auth user session** (cookie). API keys
(`apk_live_…`) do **not** work here — they exist *below* this surface. To
drive it from a script:

```bash
# 1. Sign in and keep the session cookie
curl -s -c cookies.txt -X POST http://localhost:3000/api/auth/sign-in/email \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"secret"}'

# 2. Use the cookie on every management call
curl -s -b cookies.txt http://localhost:3000/api/organizations
```

```typescript
// In a browser app or Next.js route, use the auth client / server session:
import { authClient } from "@/lib/auth-client"; // browser: sends cookies
await authClient.organization.inviteMember({ email: "dev@example.com", role: "member" });
```

```python
import requests

s = requests.Session()
s.post("http://localhost:3000/api/auth/sign-in/email",
       json={"email": "you@example.com", "password": "secret"})
orgs = s.get("http://localhost:3000/api/organizations").json()
```

## Error and envelope format (differs from /api/v1)

Management routes use a **flat** error shape — not the nested `{error:{code}}`
of `/api/v1`:

```json
{ "error": "conflict", "message": "An organization with this slug already exists" }
```

Statuses: 400 validation, 401 no session, 403 insufficient role/permission,
404 unknown id, 409 slug conflicts. Success bodies are named per endpoint
(`{organizations}`, `{projects}`, `{project}`, `{apiKeys}`, `{ok: true}`).

## Organizations

```bash
# List organizations the signed-in user belongs to
curl -s -b cookies.txt http://localhost:3000/api/organizations

# Create one (creator becomes owner). slug: lowercase letters/digits/hyphens
curl -s -b cookies.txt -X POST http://localhost:3000/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"name":"Acme Technologies","slug":"acme-technologies"}'
```

Validation: `name` 2–100 chars, `slug` required (globally unique, 409 on
conflict), optional `logo` URL.

## Projects

```bash
# List an organization's projects (membership required)
curl -s -b cookies.txt "http://localhost:3000/api/projects?organizationId=org_ID"

# Create (owner/admin only). slug optional — derived from name if omitted
curl -s -b cookies.txt -X POST http://localhost:3000/api/projects \
  -H "Content-Type: application/json" \
  -d '{"organizationId":"org_ID","name":"Production API","description":"Main backend"}'

# Details — also returns your role in the owning organization
curl -s -b cookies.txt http://localhost:3000/api/projects/proj_ID
# → { "project": {...}, "organization": {...}, "role": "owner" }
```

Project objects include quota fields (`maxStorageBytes`, `maxFileCount`,
`maxFileSizeBytes`, `monthlyBandwidthBytes`) as numbers (`null` = inherit the
global limit).

```bash
# Update name/description/quotas (owner/admin only)
curl -s -b cookies.txt -X PATCH http://localhost:3000/api/projects/proj_ID \
  -H "Content-Type: application/json" \
  -d '{"maxStorageBytes": 10737418240, "maxFileSizeBytes": null}'

# Delete a project (owner/admin only) — cascades its files and keys
curl -s -b cookies.txt -X DELETE http://localhost:3000/api/projects/proj_ID
```

## API keys

Keys belong to a **project**, are returned in raw form **exactly once**, and
are stored only as SHA-256 hashes. Prefix reveals environment: `apk_test_…` /
`apk_live_…`.

```bash
# List a project's keys (raw secrets are never included again)
curl -s -b cookies.txt http://localhost:3000/api/projects/proj_ID/api-keys
# → { "apiKeys": [ { "id", "name", "keyPrefix", "environment",
#      "lastUsedAt", "expiresAt", "createdAt", "revokedAt" } ] }

# Create one — SAVE rawKey immediately; it cannot be retrieved again
curl -s -b cookies.txt -X POST http://localhost:3000/api/projects/proj_ID/api-keys \
  -H "Content-Type: application/json" \
  -d '{"name":"Production server","environment":"live","expiresInDays":365}'
# → { "apiKey": {...metadata only...}, "rawKey": "apk_live_x..." }

# Revoke a key (owner/admin)
curl -s -b cookies.txt -X DELETE http://localhost:3000/api/api-keys/key_ID
```

Validation: `name` 1–100 chars; `environment` `test` (default) or `live`;
`expiresInDays` 1–3650, omit for a non-expiring key. Pass the raw key to the
file API as `Authorization: Bearer <rawKey>`.



## Team: members and invitations

Invitations are handled by Better Auth's organization plugin (routes under
`/api/auth/organization/*`); the dashboard calls them via `authClient`:

```typescript
await authClient.organization.inviteMember({ email, role: "member" }); // owner/admin
await authClient.organization.removeMember({ memberIdOrEmail: email });
await authClient.organization.cancelInvitation({ invitationId });
// Invitee:
await authClient.organization.acceptInvitation({ invitationId });
await authClient.organization.rejectInvitation({ invitationId });
```

Pending invitations are listed server-side via Prisma (`invitation` model,
`status: "pending"`) and accepted at `/invitations/:id` in the dashboard.

## Roles and permissions

`owner > admin > member`. Checks are centralized in `lib/roles.ts` — never
duplicated in route handlers. Enforcement pattern:
`requireOrganizationMember` → `requireOrganizationAdmin` →
`requireProjectAccess` / `requireProjectAdmin(permission)`.

| Permission | Roles |
| --- | --- |
| `org:update`, `org:manage-members`, `org:manage-projects`, `org:manage-api-keys`, `org:manage-webhooks` | owner, admin |
| `org:delete`, `org:transfer-ownership` | owner only |
| `project:read` | owner, admin, member |
| `project:update`, `project:delete` | owner, admin |

## Rules for agents

1. **Never trust client-supplied tenant ids** — every server-side operation
   must derive authorization from the session: user → membership →
   organization → project → resource.
2. **Never log or persist raw API keys.** The create response is the only
   time a raw key exists; key-creation UIs must warn users to save it.
3. **Use `slugSchema`/`slugify`** (`lib/validation.ts`) for slugs; treat 409
   conflicts as user-visible errors, not retries.
4. **Quota fields are `number | null`** over the wire (BigInt columns are
   serialized server-side); `null` means "inherit the global limit".
5. Billing routes (`/api/billing/*`, Paychangu webhook) exist but belong to a
   later phase — do not wire them into tenant-automation flows yet.

## Related skills

- `apulodi` — overview, auth model, envelopes for the developer file API
- `apulodi-file-uploads` — what the API keys created here authenticate
- `apulodi-webhooks` — project webhooks managed in the dashboard
