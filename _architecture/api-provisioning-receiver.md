# API Provisioning Receiver — Architecture

Current-state architecture reference for `services/api-provisioning-receiver/`.

**Last updated:** 2026-03-27

---

## Overview

Cloudflare Worker that receives HMAC-signed provisioning requests from the Sender Worker, validates them through a layered security pipeline, and creates API keys via Supabase.

```
Flutter App
  │
  └─→ Sender Worker  (IntegrityLandingPage/workers/sender-worker/)
        Signs: HMAC-SHA256(timestamp.body, SHARED_SECRET)
        Headers: x-timestamp, x-signature, [x-key-id]
        │
        └─→ Receiver Worker  (services/api-provisioning-receiver/)
              │
              ├─ 1. IP rate limit check (KV)
              ├─ 2. Auth headers present
              ├─ 3. Timestamp replay window
              ├─ 4. HMAC-SHA256 signature verify
              ├─ 5. Nonce deduplication (KV)
              ├─ 6. Payload schema validation (Zod)
              ├─ 7. Email domain + MX record check
              ├─ 8. Email rate limit check (KV)
              │
              ├─→ Auth0 /userinfo — validate live JWT, extract sub + email
              ├─→ Supabase REST — look up user by auth0_id
              ├─→ Supabase REST — upsert team org by domain
              ├─→ Supabase REST — add org member (parallel with quota check)
              ├─→ Supabase REST — check org API key quota
              │
              └─→ Supabase Edge Fn  (api-keys-create)
                    Returns: { token, keyId, prefix, tier }
```

---

## Request Processing Pipeline

### 1. IP rate limit

Checked first, before HMAC work. Uses `CF-Connecting-IP` (falls back to `X-Forwarded-For`).

- Window: 15 minutes, fixed
- Limit: 20 requests per IP per window
- Returns `429 RATE_LIMITED` with `Retry-After` header if exceeded
- Skipped if `RATE_LIMIT_KV` binding is absent (local dev)

### 2. Auth headers

Requires both `x-timestamp` (Unix ms) and `x-signature` (HMAC hex) headers.

→ `401 MISSING_AUTH_HEADERS` if either is absent

### 3. Timestamp validation

Asymmetric window: allows ±30s forward (clock skew) and -5 minutes backward (replay).

- `ts > now + 30_000` → stale
- `ts < now - 300_000` → stale

→ `401 INVALID_TIMESTAMP`; Sentry audit event `auth.replay_detected` (reason: `timestamp_drift`)

### 4. Signature verification

Message format: `{timestamp}.{raw body text}`

Key resolution (`resolveSigningKey`):
- If `x-key-id` header is present → look up in `SIGNING_KEYS` (JSON `Record<string,string>`)
- If `x-key-id` is absent → use `SHARED_SECRET`
- If key ID is present but not found in `SIGNING_KEYS` → `null` → reject

Verification is constant-time via Web Crypto `HMAC` `verify`.

→ `401 INVALID_SIGNATURE`; Sentry audit event `auth.invalid_signature`

### 5. Nonce deduplication

After successful signature verification, the signature itself is used as a nonce.

- KV key: `nonce:{signature}`
- TTL: equal to the 5-minute replay window (`NONCE_TTL_SECONDS`)
- If key already exists in KV → replay detected

→ `401` with `ERROR_CODE.REPLAY_DETECTED`; Sentry audit event `auth.replay_detected` (reason: `nonce_reuse`)

Skipped if `RATE_LIMIT_KV` binding is absent.

### 6. Payload validation

`inboxPayloadSchema` is a Zod discriminated union on `action`:

| `action` | Required fields | Optional |
|---|---|---|
| `provision_api_key` | `jwt` (JWT string), `name`, `email` | `tier` (default `starter`), `org_name` |
| `sign_in` | `email` | — (stub; returns `400 UNKNOWN_ACTION`) |

→ `400 JSON_PARSE_ERROR` — body not valid JSON
→ `400 UNKNOWN_ACTION` — `action` is a string but not a known value
→ `400 MISSING_FIELDS` — schema validation failed for another reason

### 7. Email domain + MX validation

- `emailToRegistrableDomainSchema` — extracts registrable domain via tldts
- `hasMxRecords(domain)` — Cloudflare DoH MX lookup, fail-open after 2 retries

→ `400 INVALID_EMAIL_DOMAIN` if either check fails

### 8. Email rate limit

- Window: 1 hour, fixed
- Limit: 5 requests per email per window

→ `429 RATE_LIMITED` with `Retry-After`; Sentry audit event `rate.limit_exceeded`

---

## provision_api_key Handler

`handlers/provision-api-key.ts`

### Auth0 JWT validation

```
GET https://{AUTH0_DOMAIN}/userinfo
Authorization: Bearer {jwt}
```

- Non-2xx → throws; Sentry `auth.invalid_jwt`
- Response parsed via `auth0UserinfoSchema` (requires `sub`, `email`)
- `auth0.email !== payload.email` → throws; Sentry `auth.email_mismatch`

### Supabase — user lookup

```
GET /rest/v1/users?auth0_id=eq.{sub}&select=id
```

Returns Supabase UUID for the Auth0 `sub`. Throws if no row found.

### Supabase — team org upsert

```
POST /rest/v1/organizations
{ slug: "team-{domain}", name, type: "team", domain, current_plan: tier, billing_status: "inactive" }
```

- 201 → new org; use returned `id`
- 409 + PG code `23505` (unique violation on `organizations(domain) WHERE type = 'team'`) → fetch existing org by domain
- Other error → propagates

`org_name` defaults to `domain` if not provided.

### Supabase — membership + quota check (parallel)

Both run in `Promise.all`:

**Membership** (`POST /rest/v1/organization_memberships`):
- 204 → added
- 409 + `23505` → already a member, continue

**Quota check** (`GET /rest/v1/api_keys?organization_id=eq.{id}`):

| Tier | Limit |
|---|---|
| `starter` | 3 |
| `growth` | 10 |
| `enterprise` | unlimited (skips DB query) |

→ `429 QUOTA_EXCEEDED` if `currentCount >= limit`; Sentry `provision.quota_exceeded`

### Edge function — key creation

```
POST {SUPABASE_URL}/functions/v1/api-keys-create
Authorization: Bearer {jwt}
{ name, tier, domain }
```

Response validated via `provisionApiKeyResultSchema`:

```typescript
{
  token: z.string().regex(/^obtk_[0-9a-f]{64}$/),
  keyId: z.string().min(1),
  prefix: z.string().min(1),
  tier: z.enum(['starter', 'growth', 'enterprise']),
}
```

Success response forwarded as `{ ok: true, token, keyId, prefix, tier }`.

---

## Error Codes

| Code | HTTP | Meaning |
|---|---|---|
| `MISSING_AUTH_HEADERS` | 401 | `x-timestamp` or `x-signature` absent |
| `INVALID_TIMESTAMP` | 401 | Outside ±30s/5min window |
| `INVALID_SIGNATURE` | 401 | HMAC mismatch or unknown key ID |
| `REPLAY_DETECTED` | 401 | Nonce already seen in KV |
| `JSON_PARSE_ERROR` | 400 | Body is not valid JSON |
| `MISSING_FIELDS` | 400 | Zod schema validation failed |
| `UNKNOWN_ACTION` | 400 | Unknown or unimplemented action |
| `INVALID_EMAIL_DOMAIN` | 400 | Domain extraction or MX check failed |
| `RATE_LIMITED` | 429 | IP or email rate limit exceeded |
| `QUOTA_EXCEEDED` | 429 | Org at tier API key limit |
| `PROVISION_ERROR` | 500 | Supabase or edge function error |
| `NOT_FOUND` | 404 | Unknown route |

---

## Observability

Audit events are captured via Sentry (`captureAuditEvent`) on every failure and on provisioning success. All events include `tags: { audit: "true", event_type: "..." }`.

| Event | Level | Trigger |
|---|---|---|
| `auth.replay_detected` | warning | Timestamp drift or nonce reuse |
| `auth.invalid_signature` | warning | HMAC mismatch or unknown key ID |
| `auth.invalid_jwt` | warning | Auth0 /userinfo non-2xx or invalid response |
| `auth.email_mismatch` | warning | JWT email ≠ payload email |
| `rate.limit_exceeded` | warning | IP or email over limit |
| `provision.success` | info | Key created; includes `email`, `tier`, `domain`, `keyId`, `prefix` |
| `provision.failure` | warning | Supabase/edge fn threw; also `captureException` |
| `provision.quota_exceeded` | warning | Quota check failed; includes `currentCount`, `limit` |

Sentry is initialized via `Sentry.withSentry` wrapper in `index.ts`. `tracesSampleRate: 0` (performance tracing disabled). `SENTRY_DSN` is optional — if absent, events are silently dropped.

---

## KV Key Namespaces

All entries use `RATE_LIMIT_KV` binding.

| Prefix | Format | TTL | Purpose |
|---|---|---|---|
| `rl:ip:{ip}:{windowId}` | count string | 15 min window | IP rate limit counter |
| `rl:email:{email}:{windowId}` | count string | 1 hr window | Email rate limit counter |
| `nonce:{signature}` | `"1"` | 5 min | Replay deduplication |

`windowId = Math.floor(Date.now() / windowMs)` — entries auto-expire at window boundary.

---

## Key Rotation (`x-key-id`)

Set `SIGNING_KEYS` to a JSON object:

```bash
wrangler secret put SIGNING_KEYS
# {"v1":"old-secret","v2":"new-secret"}
```

Sender includes `x-key-id: v2` to select the active key. Fallback: if `x-key-id` is absent, `SHARED_SECRET` is used. Unknown key ID → `401 INVALID_SIGNATURE`.

Remove old key IDs from `SIGNING_KEYS` once all senders have migrated.

---

## Environment Variables

| Variable | Type | Required | Purpose |
|---|---|---|---|
| `SHARED_SECRET` | Secret | Yes | HMAC key when `x-key-id` is absent |
| `SIGNING_KEYS` | Secret | No | JSON `Record<string,string>` for key rotation |
| `AUTH0_DOMAIN` | Secret | Yes | Auth0 tenant domain for `/userinfo` |
| `SUPABASE_URL` | Secret | Yes | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Secret | Yes | Service role key for org/membership writes |
| `SUPABASE_ANON_KEY` | Secret | Yes | Supabase anon key (available in `Env`) |
| `SENTRY_DSN` | Secret | No | Sentry DSN; audit events dropped if absent |
| `RATE_LIMIT_KV` | KV binding | No | Rate limit + nonce store; skipped if absent |

---

## File Structure

```
services/api-provisioning-receiver/src/
├── index.ts                    # Route dispatcher, request pipeline
├── types.ts                    # Env, constants (ERROR_CODE, QUOTA_LIMITS, ROUTES, etc.)
├── crypto.ts                   # HMAC sign/verify (Web Crypto)
├── replay-protection.ts        # Timestamp validation, REPLAY_WINDOW_MS, CLOCK_SKEW_MS
├── nonce.ts                    # KV-backed nonce deduplication
├── rate-limit.ts               # KV-backed IP + email fixed-window rate limiter
├── supabase.ts                 # lookupUserByAuth0Id, ensureTeamOrg, addOrgMember, checkOrgKeyQuota
├── dns.ts                      # hasMxRecords (Cloudflare DoH)
├── audit.ts                    # captureAuditEvent, captureProvisionError (Sentry)
├── schemas.ts                  # Re-exports / local Zod schemas
├── utils.ts                    # json(), errorResponse(), extractAuthHeaders(), resolveSigningKey()
├── version.ts                  # VERSION constant
├── handlers/
│   └── provision-api-key.ts    # handleProvisionApiKey, QuotaExceededError
└── __tests__/
    ├── crypto.test.ts
    ├── replay-protection.test.ts
    ├── integration.test.ts
    ├── supabase.test.ts
    └── fixtures.ts             # makeKv(), makeSignature(), shared helpers
```

---

## Related Docs

- [Provisioning flow — step by step](api-key-provisioning.md)
- [Security architecture & threat model](../api-provisioning-security.md)
- [Flutter client contract](../api-provisioning-flutter-contract.md)
