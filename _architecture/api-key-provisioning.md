# API Key Provisioning Flow

End-to-end data flow for provisioning an API key for a new user.

**Last updated:** 2026-03-27

---

## Architecture

```
Flutter App
  │
  └─→ Sender Worker  (~/code/is-public-sites/IntegrityLandingPage/workers/sender-worker/)
      │  Signs request with HMAC-SHA256
      │
      └─→ Receiver Worker  (services/api-provisioning-receiver/)
          │  Validates signature, parses payload, validates email/domain
          │
          ├─→ Auth0  (/userinfo)
          │    Validates JWT is live, extracts sub + email
          │
          ├─→ Supabase (REST API, service role key)
          │    Looks up user by auth0_id
          │    Upserts team org by domain
          │    Adds user as org member
          │
          └─→ Supabase Edge Function  (api-keys-create)
               Creates API key, returns token + metadata
```

---

## Step-by-step

### 1. Sender Worker

- User submits the signup/onboarding form
- Sender builds payload: `{ action: "provision_api_key", jwt, name, email, tier?, org_name? }`
- Generates `x-timestamp` (Unix ms)
- Signs `HMAC-SHA256(timestamp.body, SHARED_SECRET)` → `x-signature`
- `POST /inbox` to receiver with both headers

### 2. Receiver — transport validation

Validated by `index.ts` before any business logic:

| Check | Failure |
|---|---|
| `x-timestamp` + `x-signature` headers present | 401 `MISSING_AUTH_HEADERS` |
| Timestamp within ±5 min replay window | 401 `INVALID_TIMESTAMP` |
| HMAC-SHA256 signature matches (constant-time) | 401 `INVALID_SIGNATURE` |
| Body is valid JSON | 400 `JSON_PARSE_ERROR` |

### 3. Receiver — payload validation

`inboxPayloadSchema` is a discriminated union on `action`:

- `provision_api_key` — requires `jwt` (`z.string().jwt()`), `name`, `email`; optional `tier` (default `'starter'`), `org_name`
- `sign_in` — requires `email` (stub, not yet implemented)
- Any other `action` value — 400 `UNKNOWN_ACTION`
- Missing `action` — 400 `MISSING_FIELDS`

Then:
- `emailToRegistrableDomainSchema.safeParse(email)` — extracts registrable domain via tldts
- `hasMxRecords(domain)` — Cloudflare DoH MX check, fail-open after 2 retries
- Either failure → 400 `INVALID_EMAIL_DOMAIN`

### 4. Auth0 — JWT validation

`GET https://{AUTH0_DOMAIN}/userinfo` with `Authorization: Bearer {jwt}`

- Non-2xx → throws `"JWT validation failed: invalid or expired token"`
- Response parsed via `auth0UserinfoSchema` (requires `sub`, `email`)
- `auth0Result.data.email !== input.email` → throws `"email mismatch"`

This confirms the JWT is live, unexpired, and belongs to the claimed email address.

### 5. Supabase — user lookup

`GET /rest/v1/users?auth0_id=eq.{sub}` (service role key)

Returns the Supabase user UUID for the Auth0 `sub`.

### 6. Supabase — team org upsert

`POST /rest/v1/organizations` with `{ domain, name, type: 'team', current_plan: tier }`

- **201** → new org created, use returned `id`
- **409 + code `23505`** (unique violation on partial index `organizations(domain) WHERE type = 'team'`) → org already exists, fetch by domain and use existing `id`
- Any other error → propagates

### 7. Supabase — org membership

`POST /rest/v1/organization_memberships` with `{ organization_id, user_id }`

- **204** → member added
- **409 + code `23505`** → user already a member, silently continue
- Any other error → propagates

### 8. Edge function — API key creation

`POST {SUPABASE_URL}/functions/v1/api-keys-create` with `{ name, tier, domain }` and `Authorization: Bearer {jwt}`

Response validated via `provisionApiKeyResultSchema`:

```typescript
{
  token: z.string().regex(/^obtk_[0-9a-f]{64}$/),
  keyId: z.string().min(1),
  prefix: z.string().min(1),
  tier: z.enum(['starter', 'growth', 'enterprise']),
}
```

### 9. Response

The edge function returns `{ ok: true, token, keyId, prefix, tier }` to the receiver, which forwards it unchanged to the sender worker.

The sender worker is a **transparent proxy** for this step (`handleSend`, `index.ts:139-144`):

- Reads the receiver response body as text (`receiverRes.text()`)
- Re-emits it with the same HTTP status code and `content-type` header
- Does **not** parse, validate, or transform the body
- If the receiver returns an error status (4xx/5xx), that status is forwarded as-is

**CORS**: if the request `origin` header matches `ALLOWED_ORIGINS_JSON` (or the hardcoded fallback `integritystudio.ai`/`www.integritystudio.ai`), the sender adds `access-control-allow-origin: <origin>` to the response. Requests from disallowed origins are rejected with 403 before `handleSend` is called.

**Error paths in `handleSend`**:

| Condition | Response |
|---|---|
| `RECEIVER_WORKER_URL` not a valid URL | 500 `INTERNAL_ERROR` |
| `SHARED_SECRET` missing | 500 `INTERNAL_ERROR` |
| `SendRequestSchema` validation fails | 400/401 with field-specific error code |
| `fetch` throws `TypeError` (network) | 502 `INTERNAL_ERROR` `"receiver-worker unreachable"` |
| Any other thrown error | 500 `INTERNAL_ERROR` `"send failed"` |

The Flutter client receives either the receiver's success payload or one of the above error shapes.

---

## Key schemas

| Schema | Location | Purpose |
|---|---|---|
| `inboxPayloadSchema` | `src/lib/validation/api-schemas.ts` | Discriminated union, validates full request payload |
| `actionSchema` | `src/lib/validation/api-schemas.ts` | `z.enum(['provision_api_key', 'sign_in'])` |
| `auth0UserinfoSchema` | `src/lib/validation/api-schemas.ts` | Validates Auth0 `/userinfo` response |
| `provisionApiKeyResultSchema` | `src/lib/validation/api-schemas.ts` | Validates edge fn response including token regex |
| `apiKeyTierSchema` | `src/lib/validation/api-schemas.ts` | `z.enum(['starter', 'growth', 'enterprise'])` |
| `emailToRegistrableDomainSchema` | `src/lib/validation/api-schemas.ts` | Email → registrable domain via tldts + pipe |

---

## Environment variables

| Variable | Service | Purpose |
|---|---|---|
| `SHARED_SECRET` | Receiver | HMAC-SHA256 signing key shared with sender |
| `AUTH0_DOMAIN` | Receiver | Auth0 tenant domain for `/userinfo` calls |
| `SUPABASE_URL` | Receiver | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Receiver | Service role key for org/membership writes |

---

## Related docs

- [Security architecture & threat model](../api-provisioning-security.md)
- [Flutter client contract](../api-provisioning-flutter-contract.md)
