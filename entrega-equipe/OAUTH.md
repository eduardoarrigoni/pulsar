# Provider OAuth — Implementation Reference

**Status:** Reference / draft
**Date:** 2026-08-24
**Scope:** Connect an athlete's external account once, keep it connected indefinitely, and ingest activities without polling. Strava is the worked example; the model is provider-agnostic by design.

> **Read [ANALYSIS.md](ANALYSIS.md) first.** A valid token does **not** unlock coach visibility or long-term storage — those limits attach to the data, not the authentication. See [§8 Compliance](#8-compliance-obligations) for what this token may lawfully serve.

---

## 1. The goal

The athlete authorizes **once**. After that:

```
ONE TIME
  Athlete clicks "Connect Strava"
    → consent screen on strava.com
    → redirect back with ?code=...
    → exchange code for tokens
    → store

FOREVER AFTER (no athlete involvement)
  access_token expired?  → refresh it
  new activity?          → webhook tells you
  athlete revoked?       → webhook tells you
```

The connection lives until the athlete revokes it. The only thing that can break it accidentally is mishandled token rotation — see [§4](#4-the-bug-that-breaks-authorize-once).

---

## 2. Authorization

### 2.1 Send the athlete to the consent screen

```
GET https://www.strava.com/oauth/authorize
  ?client_id=<CLIENT_ID>
  &redirect_uri=https://app.pulsar.example/oauth/strava/callback
  &response_type=code
  &approval_prompt=auto
  &scope=read,activity:read_all
  &state=<CSRF_TOKEN>
```

| Param | Notes |
|---|---|
| `approval_prompt` | `auto` — a returning athlete who already approved is redirected straight through. `force` re-prompts every time. **Use `auto`** — this is what makes it feel like "authorize once". |
| `state` | Random, single-use, bound to the session. **Required** — without it the callback is CSRF-open. Verify on return. |
| `redirect_uri` | Must match the callback domain registered in your API settings. |

### 2.2 Scopes

| Scope | Grants |
|---|---|
| `read` | Public profile |
| `profile:read_all` | Full profile, including private |
| `activity:read` | Activities set to *Everyone* / *Followers* |
| `activity:read_all` | **All** activities, including *Only You* |
| `activity:write` | Create/modify activities |

**Pulsar needs `read,activity:read_all`.** Without `_all`, any activity the athlete marked private is invisible — which for a coaching context is most of the interesting ones.

### 2.3 Handling the callback

Success:
```
GET /oauth/strava/callback?state=<CSRF>&code=<CODE>&scope=read,activity:read_all
```

Denied:
```
GET /oauth/strava/callback?state=<CSRF>&error=access_denied
```

⚠️ **The athlete can uncheck individual scopes on the consent screen.** The `scope` parameter on the callback reports what was *actually* granted, which is not necessarily what you asked for. Always verify:

```js
const granted = new Set((query.scope ?? '').split(','))
if (!granted.has('activity:read_all')) {
  // Degraded connection — tell the athlete what won't work,
  // don't silently store a connection that can't see their runs.
}
```

---

## 3. Tokens

### 3.1 Exchange the code

```http
POST https://www.strava.com/oauth/token
Content-Type: application/x-www-form-urlencoded

client_id=<CLIENT_ID>
&client_secret=<CLIENT_SECRET>
&code=<CODE>
&grant_type=authorization_code
```

Response:
```json
{
  "token_type": "Bearer",
  "expires_at": 1785000000,
  "expires_in": 21600,
  "refresh_token": "e5n567567...",
  "access_token": "a4b945687g...",
  "athlete": { "id": 12345, "firstname": "...", "lastname": "..." }
}
```

Keep `athlete.id` — it's the `owner_id` on every webhook event, and it's how you route an incoming event to a Pulsar athlete.

### 3.2 Lifetimes

| Token | Lifetime | Behaviour |
|---|---|---|
| `access_token` | 6 hours (`expires_in: 21600`) | Bearer token on API calls |
| `refresh_token` | No expiry | Valid until revoked — **but rotates**, see §4 |

### 3.3 Refresh

```http
POST https://www.strava.com/oauth/token

client_id=<CLIENT_ID>
&client_secret=<CLIENT_SECRET>
&grant_type=refresh_token
&refresh_token=<REFRESH_TOKEN>
```

Returns a fresh `access_token`, a new `expires_at`, **and a `refresh_token` that may differ from the one you sent**.

Refresh proactively — don't wait for a 401:

```js
const SKEW = 300 // 5 min
if (conn.expires_at - now() < SKEW) await refresh(conn)
```

### 3.4 Using the token

```http
GET https://www.strava.com/api/v3/athlete/activities?after=<EPOCH>&per_page=100
Authorization: Bearer <ACCESS_TOKEN>
```

---

## 4. The bug that breaks "authorize once"

Two failure modes, both of which force the athlete to re-authorize. This is the single most important section of this document.

### 4.1 Dropping the rotated refresh token

Every refresh response may carry a **new** `refresh_token`. Keep using the old one and the connection dies.

```js
const res = await refresh(conn.refresh_token)

await db.updateConnection(conn.id, {
  access_token:  res.access_token,
  refresh_token: res.refresh_token,   // ← ALWAYS write this back
  expires_at:    res.expires_at,
})
```

Never conditionally skip that write. Write it every time, unchanged or not.

### 4.2 The concurrent-refresh race

Two workers refresh the same connection at the same time. Both receive new tokens; the second write wins; the first worker's token is already invalid — and depending on ordering, the *stored* token may be the dead one.

```
worker A ──refresh(T0)──> T1      stores T1
worker B ──refresh(T0)──> T2      stores T2   (T1 now dead, T2 stored — survivable)
                                              (reverse order → T1 stored, dead — connection lost)
```

Serialize per connection. Any of these works:

- `SELECT ... FOR UPDATE` on the connection row for the duration of the refresh
- A distributed lock keyed by `connection:{id}`
- A single dedicated refresh worker; everyone else reads the stored token

Pick one and apply it consistently — this failure is intermittent, load-dependent, and miserable to debug in production.

### 4.3 Treat 401 as revocation

A `401` on a call made with a freshly refreshed token means the athlete revoked access on Strava's side. Mark the connection revoked and start the deletion clock (§8.2) — don't retry in a loop.

---

## 5. Webhooks (push subscriptions)

Polling every athlete is wasteful and will hit rate limits. Subscribe instead. **One subscription per application**, not per athlete — it covers every connected athlete.

### 5.1 Create the subscription

```http
POST https://www.strava.com/api/v3/push_subscriptions

client_id=<CLIENT_ID>
&client_secret=<CLIENT_SECRET>
&callback_url=https://api.pulsar.example/webhooks/strava
&verify_token=<YOUR_RANDOM_STRING>
```

### 5.2 Answer the validation handshake

Immediately after the request above, Strava calls your endpoint:

```
GET /webhooks/strava
  ?hub.mode=subscribe
  &hub.challenge=15f7d1a91c1f40f8a748fd134752feb3
  &hub.verify_token=<YOUR_RANDOM_STRING>
```

You must verify the token and echo the challenge:

```js
app.get('/webhooks/strava', (req, res) => {
  if (req.query['hub.verify_token'] !== process.env.STRAVA_VERIFY_TOKEN) {
    return res.sendStatus(403)
  }
  res.json({ 'hub.challenge': req.query['hub.challenge'] })
})
```

The response body key is literally `hub.challenge`. The subscription is not created until this succeeds.

### 5.3 Receiving events

```json
{
  "object_type": "activity",
  "object_id": 1360128428,
  "aspect_type": "create",
  "updates": {},
  "owner_id": 134815,
  "subscription_id": 120475,
  "event_time": 1516126040
}
```

| Field | Meaning |
|---|---|
| `object_type` | `activity` or `athlete` |
| `aspect_type` | `create` / `update` / `delete` |
| `object_id` | Activity id, or athlete id |
| `owner_id` | The Strava athlete — your routing key |

⚠️ **Respond `200` within 2 seconds.** Do not fetch the activity inline. Acknowledge, enqueue, process asynchronously:

```js
app.post('/webhooks/strava', async (req, res) => {
  res.sendStatus(200)              // acknowledge FIRST
  await queue.push(req.body)       // then work
})
```

The event carries no activity data — only an id. Your worker fetches the detail with that athlete's token.

---

## 6. Deauthorization

### 6.1 Athlete revokes on Strava

Arrives as a webhook:

```json
{
  "object_type": "athlete",
  "aspect_type": "update",
  "object_id": 134815,
  "owner_id": 134815,
  "updates": { "authorized": "false" }
}
```

Note `"false"` is a **string**, not a boolean.

```js
if (e.object_type === 'athlete' && e.updates?.authorized === 'false') {
  await markRevoked(e.owner_id)   // starts the §8.2 deletion clock
}
```

### 6.2 Disconnecting from inside Pulsar

```http
POST https://www.strava.com/oauth/deauthorize
Authorization: Bearer <ACCESS_TOKEN>
```

Then delete the stored tokens. Give athletes this control in the UI — it's both good practice and an LGPD expectation.

---

## 7. Rate limits

Read the live values from the response headers rather than hardcoding them:

```
X-RateLimit-Limit: 200,2000        // 15-min, daily
X-RateLimit-Usage: 30,300
```

Approximate Standard Tier ceilings: **200 req/15 min and 2,000/day (read)**, 400/15 min and 4,000/day overall. Sources vary and tiers change — **confirm against your app's API dashboard.**

Limits are **per application**, not per athlete, so they are shared across every connected athlete. Practical consequences:

- Webhooks over polling (§5) is the single biggest saving.
- On `429`, back off exponentially and requeue — don't drop the event.
- Backfilling an athlete's history on first connect is the spikiest operation you have. Throttle it, run it in the background, and page through with `per_page=100`.

---

## 8. Compliance obligations

Authentication is not authorization to keep or share. These are enforced by [ANALYSIS.md](ANALYSIS.md) Finding 1.

### 8.1 What this token may serve

| Use | Allowed |
|---|---|
| Show the athlete their own data, fetched live | ✅ |
| Retain beyond 7 days | ❌ §6.2, §5.5 |
| Display to their coach | ❌ §2.3, §6.1 |
| Feed ML / embeddings / RAG | ❌ §5.3 |

Everything a coach sees, and everything retained long-term, must arrive via athlete file upload or manual entry — **not through this token.**

### 8.2 Deletion on revocation

On revocation (webhook §6.1, or a persistent `401` per §4.3), you must **permanently delete all Strava Data and all data derived from it within 30 days** (§7.4). Derived metrics are explicitly included — a load score computed from Strava HR is still covered.

Design consequence: **tag every record with its provenance at write time.** Retrofitting "which of these rows came from Strava?" onto an existing schema is painful, and you cannot comply without it.

```
activity
  id
  athlete_id
  source          'strava' | 'file_upload' | 'manual'
  source_ref      provider activity id, or uploaded file id
  ingested_at
  ...
```

Data from `file_upload` and `manual` is unaffected by revocation — which is the whole point of the [ANALYSIS.md](ANALYSIS.md) recommendation.

---

## 9. Provider-agnostic model

Garmin, COROS and Polar are all OAuth 2.0 with refresh tokens. Build the connection layer once.

```sql
provider_connection
  id
  athlete_id           FK
  provider             'strava' | 'garmin' | 'coros' | 'polar'
  provider_user_id     -- webhook routing key
  access_token         -- ENCRYPTED AT REST
  refresh_token        -- ENCRYPTED AT REST
  expires_at
  scopes
  connected_at
  revoked_at           -- nullable; set → deletion workflow
  UNIQUE (provider, provider_user_id)
```

One `TokenManager` handles refresh-before-expiry and rotation for every provider; per-provider adapters handle endpoints, scopes and webhook payload shapes.

**Encrypt both tokens at rest.** They grant access to personal data, and under LGPD that data includes health information.

When Garmin's program reopens, adding it is: an adapter, a webhook handler, a config entry. Note that Garmin's Activity API pushes **`.FIT` files**, which land in the same parser as athlete uploads — see [ANALYSIS.md](ANALYSIS.md) Finding 3.

---

## 10. Implementation checklist

**Authorization**
- [ ] `state` generated, stored in session, verified on callback
- [ ] `approval_prompt=auto`
- [ ] Granted scopes verified against required scopes
- [ ] `error=access_denied` handled with a real message

**Tokens**
- [ ] Rotated `refresh_token` written back on **every** refresh
- [ ] Per-connection lock around refresh
- [ ] Proactive refresh with ~5 min skew
- [ ] Persistent `401` → mark revoked, start deletion
- [ ] Both tokens encrypted at rest

**Webhooks**
- [ ] Validation handshake echoes `hub.challenge`
- [ ] `verify_token` checked
- [ ] `200` returned before any processing
- [ ] Events queued, processed async, idempotent by `object_id`
- [ ] Deauthorization (`updates.authorized === 'false'`) handled

**Compliance**
- [ ] `source` recorded on every ingested record
- [ ] Deletion job for revoked connections, ≤30 days, incl. derived data
- [ ] Retention rule enforced for Strava-sourced data
- [ ] Athlete-facing disconnect button

**Operations**
- [ ] Rate-limit headers read and respected
- [ ] `429` → exponential backoff + requeue
- [ ] Initial backfill throttled and backgrounded

---

## 11. Scheduled platform changes

| Date | Change |
|---|---|
| 2026-09-01 | Club endpoints and Segment Explore deprecated |
| 2027-06-01 | New API base URL; authorization method changes |

The 2027 change affects both the base URL and the auth mechanism. Keep provider endpoints in configuration, not scattered through the codebase.

---

## Sources

Verified 2026-08-24. Strava's developer documentation is the authority; re-check before implementing.

- [Strava — Getting started / authentication](https://developers.strava.com/docs/getting-started/)
- [Strava — Webhook events](https://developers.strava.com/docs/webhooks/)
- [Strava — Rate limits](https://developers.strava.com/docs/rate-limits/)
- [Strava API Policy (2026)](https://www.strava.com/legal/api_policy)
- [An Update To Our Developer Program](https://communityhub.strava.com/insider-journal-9/an-update-to-our-developer-program-13428)
