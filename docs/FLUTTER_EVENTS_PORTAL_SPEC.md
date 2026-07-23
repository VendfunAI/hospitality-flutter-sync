# Vendfun Events API — Contract for Flutter clients

> **What this document is:** Vendfun’s external API contract for hotel staff
> features under `/v1/Events/*` (auth, audience CRM, blasts, A/B comparison,
> human inbox, dashboard, profile, webhooks).  
> **What it is not:** Flutter product scope, UX, or PRD — that remains with the
> Flutter team.  
> **Status:** Backend **STG-live** (platform-service ≥ v1.0.375). **Not on PRD**
> until promoted (code is on the PRD box, but the Events DB migrations are not
> applied there yet — see §1 note).  
> **Machine-readable contract:** `GET {API_BASE}/docs/openapi.json` (paths under
> `/Events/*`) — **authoritative** if this prose and OpenAPI diverge.  
> **Version:** 1.3.1 (durable copy for Flutter sync repo; §9 protocol expanded
> — todos/decisions/spec-diffs/**bugs**, secrets + human gate) — 2026-07-23
> (v1.3 added §9 coordination channel; v1.2 Experiments; v1.1 sessionId +
> Dashboard/Profile/password/Campaigns Update/webhooks; v1.0 was 2026-07-20)

---

## 1. Base URL & transport

| Env | API base | Events status |
|-----|----------|----------------|
| **DEV** | `https://api.srv1404676.hstgr.cloud` | Code present; not the Events integration target |
| **STG** | `https://api.vendfun.cloud` | **Events live here — use this for integration testing** |
| **PRD** | `https://api.vendfun.com` | Not promoted yet (DB migrations pending) |

> **Naming change (2026-07-23):** the three environment names were re-tiered
> when a dedicated Lightsail box became the real PRD. If you bookmarked
> `api.srv1404676.hstgr.cloud` as "STG" from the 2026-07-20 version of this
> doc, that host is now called **DEV** and is not where Events is deployed —
> switch your sandbox testing to `api.vendfun.cloud` (below).

All Events endpoints:

```
POST {API_BASE}/v1/Events/<Area>/<Action>
Content-Type: application/json
```

Examples:

- `POST https://api.vendfun.cloud/v1/Events/Auth/Login`
- `POST https://api.vendfun.cloud/v1/Events/Customers/List`

OpenAPI (STG): `https://api.vendfun.cloud/docs/openapi.json`

**Brute-force / rate limiting (2026-07-22):**

- **App (P1):** `Auth/Login` counts failures per **email** (default **5** / 15 min) and per **client IP** (default **20** / 15 min). Over limit → Tier-1 `errorCode` **429**, message `Too many login attempts. Try again later.`, plus `Retry-After` when known. Counters clear on successful login.
- **Edge (P0, Lightsail):** Traefik middleware ~**10 req/min** on `/v1/Events/Auth/{Login,ForgotPassword,ResetPassword}` (priority above general `/v1`).
- **Config** (`sys_config` global): `max_login_attempts`, `login_ip_max_attempts`, `login_attempt_window_minutes`, `login_lockout_minutes`.



Edge Traefik may also return a bare HTTP 429 (outside the Tier-1 envelope) under flood; app lockout always uses HTTP 200 + `errorCode` 429.

---

## 2. Authentication (email + PIN / password)

Staff sign in with **email + password (or PIN)**. The server issues a
**`session_token`** used on subsequent calls.

| Item | Value |
|------|--------|
| Login credentials | `email` + `password` (alias: `pin`) |
| Required role | **`event_manager`** (hotel_admin / super_admin also allowed for ops) |
| Session | Opaque hex `session_token`, default ~**24 hours** (`expires_at` on login) |
| Hotel scope | `location_id` is taken from the **session user**. Normal users do not send
  `location_id`. Only `super_admin` may override with body `location_id`. |

### Login

```http
POST /v1/Events/Auth/Login
```

```json
{ "email": "user@hotel.com", "password": "123456" }
```

Success `data`:

```json
{
  "session_token": "…",
  "expires_at": "2026-07-21 12:00:00.000",
  "user": {
    "user_id": "uuid",
    "email": "user@hotel.com",
    "display_name": "…",
    "role": "event_manager",
    "location_id": "uuid",
    "location": {
      "location_id": "uuid",
      "name": "Hotel name",
      "logo_url": null,
      "timezone": "Asia/Kuala_Lumpur"
    }
  }
}
```

### Passing the token (every gated call) — updated 2026-07-22

**New canonical shape** — nests the session inside `requestInfo`, mirroring the
`requestInfo.secureId` envelope every other `/v1/*` surface uses (same wrapper,
different credential — a per-staff-member session instead of a per-hotel
device key):

```json
{
  "requestInfo": {
    "transactionId": "uuid",
    "clientId": "EVENTS_DASHBOARD",
    "sessionId": "<session_token from Login>"
  },
  "...other fields..."
}
```

`clientId` is accepted as sent and not validated server-side — use it to
identify your app/team. `transactionId` is optional and echoed back in
`responseInfo.transactionId`.

**Legacy shape still works (back-compat, no deadline to migrate off it):** the
original flat `session_token` body field, and `header.session_token` inside a
nested envelope, are both still accepted — this was an additive change, not a
breaking one. New integrations should use `requestInfo.sessionId`; existing
ones are not required to change.

`Auth/Login` itself stays flat `{email, password}` — there's no session yet to
wrap. `Auth/ForgotPassword` and `Auth/ResetPassword` (below) are also flat and
unauthenticated for the same reason.

### Session lifecycle

| Path | Body | Success `data` |
|------|------|----------------|
| `/v1/Events/Auth/Login` | `email`, `password` \| `pin` | `session_token`, `expires_at`, `user` |
| `/v1/Events/Auth/Verify` | `requestInfo.sessionId` | `valid: true`, `user` |
| `/v1/Events/Auth/Logout` | `requestInfo.sessionId` | `logged_out: true` |
| `/v1/Events/Auth/ChangePassword` | `requestInfo.sessionId`, `current_password`, `new_password` (min 8 chars) | `changed_at`, `sessions_revoked` (other active sessions killed; the calling session is kept) |
| `/v1/Events/Auth/ForgotPassword` | `email` (unauthenticated) | `sent: true` always — anti-enumeration, never reveals whether the email exists |
| `/v1/Events/Auth/ResetPassword` | `token` (emailed, single-use, 1h expiry), `new_password` (unauthenticated) | `reset: true`. Invalid and expired tokens both return the same generic `400` — never distinguishes garbage from stale-but-once-valid |

**No refresh endpoint.** There is no silent/background token renewal — a session is
valid until `expires_at` (~24h) or logout, and the only way to get a new one is
`Auth/Login` again. Client should call `Auth/Verify` on cold start / resume, and
on any `401` from a gated call, drop the local token and route to login.

### API key (`X-API-Key`)

Separate from `session_token`. Request an `X-API-Key` from Vendfun and send it
on **every** request, including `Auth/Login`. It identifies *your application*
(not the hotel, not the staff member) — this is what lets Vendfun disable one
integration's access without touching another team's key or any staff account.

| Item | Value |
|------|--------|
| Header | `X-API-Key: <your key>` |
| Today | **Optional** (soft-launch). Will become **required** — start sending it now. |
| Scope | Your key is tied to one hotel. Using it against a different hotel's session is rejected. |
| Once required, on failure | `401` missing/invalid key · `403` key not authorized for this hotel |

| Failure | `errorCode` | Meaning |
|---------|-------------|---------|
| Bad email/password | `401` | Invalid credentials |
| Too many failed logins (email or IP) | `429` | Temporary lockout — wait / Retry-After |
| Missing email/password | `400` | Validation |
| Wrong role (not event_manager / ops) | `403` | Insufficient permissions |
| Missing token on gated route | `400` | `session_token is required` |
| Expired / invalid token | `401` | Session invalid or expired |

---

## 3. Response envelope

**Always HTTP 200.** Business success/failure is in `responseInfo`.

```json
{
  "responseInfo": {
    "transactionId": "optional-echo",
    "rows": 1,
    "result": true,
    "errorCode": 0,
    "errorMessage": "",
    "returnMessage": "",
    "versionNo": "1.0.xxx"
  },
  "data": { }
}
```

| Field | Meaning |
|-------|---------|
| `result` | `true` success / `false` failure |
| `errorCode` | `0` = OK; otherwise business/error code |
| `errorMessage` | Human-readable; may include prefixes such as `WINDOW_CLOSED:…` |
| `rows` | Count of primary records in `data` (this page) |
| `total` | Present on list endpoints only — total rows matching the query, not just this page |
| `data` | Payload, or `null` on error |

### Error codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `400` | Bad request (missing fields, empty audience, template not approved, …) |
| `401` | Unauthorized (login or session) |
| `403` | Forbidden (role) |
| `404` | Not found |
| `409` | Conflict (e.g. free-form reply outside 24h window, campaign not draft, version mismatch) |
| `500` | Internal |
| `502` | Upstream send failed (e.g. WhatsApp provider) |

Optional: `transactionId` in body or `X-Transaction-Id` header — echoed for support.

---

## 4. Phone numbers

| Context | Format |
|---------|--------|
| Write / store / filters | **Digits-only international MSISDN**, no `+` — e.g. `60123132820` |
| Display | Client may format with `+` for UI; strip non-digits before write |

Bare national numbers are treated as Malaysia (+60) until multi-country dial
codes are configured.

---

## 5. Endpoint catalogue

All methods are **POST**. All routes except `Auth/Login`, `Auth/ForgotPassword`,
and `Auth/ResetPassword` require a session — see §2 for the current
`requestInfo.sessionId` shape (legacy flat `session_token` still accepted).

### 5.1 MVP — audience, blasts, human conversation

These surfaces support the first product slice: **audience management**,
**WhatsApp blasting**, and **conversation list / view / human reply**.

#### Customers (CRM / blast audience)

Hotel-scoped customers. System of record for campaign audiences. Scoped by the
session hotel (`location_id`).

| Path | Purpose |
|------|---------|
| `/v1/Events/Customers/List` | Paginated list: `segment` (`all` \| `guest` \| `prospect` \| `member`), `search`, `opt_in_only`, `tag`, `limit` (≤200), `offset`. Response `data` includes `rows[]` **and** `total` (total matching count, for page count / infinite scroll) |
| `/v1/Events/Customers/Get` | One row by `hotel_customer_id` |
| `/v1/Events/Customers/Upsert` | Create/merge by phone and/or email |
| `/v1/Events/Customers/Update` | Patch by `hotel_customer_id`; optional `version` → `409` on mismatch |
| `/v1/Events/Customers/Delete` | Soft-delete |
| `/v1/Events/Customers/Import` | Bulk upsert, **max 500 rows** per call |
| `/v1/Events/Customers/Tags` | Distinct tags for filtering |

**Typical customer fields:**  
`hotel_customer_id`, `guest_profile_id`, `phone`, `email`, `display_name`,
`is_guest`, `is_prospect`, `is_member`, `is_marketing_opt_in`, `tags_json`,
`notes`, `version`

**Upsert body (main fields):**  
`phone`, `email`, `display_name`, segment flags, `tags[]`, `notes`,
`marketing_opt_in`, `source`

Different blasts target different people via **tags / segment / opt-in** on this
table plus each campaign’s `audience_filter` — not separate audience tables.

#### Templates (read-only picker)

| Path | Purpose |
|------|---------|
| `/v1/Events/Templates/ApprovedList` | Approved WhatsApp templates ready for blasts / re-engage |

**Template fields:**  
`template_key`, `language`, `content_sid`, `category`, `var_order[]`,
`media_url`, `body_preview`

`var_order` lists named variables required for campaign create / `ReplyTemplate`.

#### Campaigns (blasts)

| Path | Purpose |
|------|---------|
| `/v1/Events/Campaigns/List` | `status` (`all` or draft/running/…), `limit`, `offset`. Response `data` includes `rows[]` **and** `total` |
| `/v1/Events/Campaigns/Get` | One campaign + template preview helpers |
| `/v1/Events/Campaigns/Create` | Create **draft** |
| `/v1/Events/Campaigns/PreviewAudience` | Audience count + sample rendered messages (no send) |
| `/v1/Events/Campaigns/Launch` | Queue sends for a **draft** only |
| `/v1/Events/Campaigns/Cancel` | Cancel |
| `/v1/Events/Campaigns/Update` | **New (2026-07-22).** Patch a **draft** campaign — `name`, `template_key`, `language`, `scheduled_at`, `variables`, `personalization`, `audience_filter`. Only the fields you send are changed. `409` if the campaign has already launched |
| `/v1/Events/Campaigns/Recipients` | Per-recipient status. Response `data` includes `rows[]` **and** `total` |
| `/v1/Events/Campaigns/Stats` | Delivery counters |

`Campaigns/Update` replaces the earlier "no update, cancel + recreate" note
from v1.0 of this doc — a draft's name/template/audience can now be edited in
place as long as it hasn't launched.

**Create body (main fields):**

```json
{
  "requestInfo": { "sessionId": "…" },
  "name": "Summer Jazz blast",
  "template_key": "whatsapp_event_invite",
  "language": "en",
  "channel": "whatsapp",
  "agent_id": "<optional uuid>",
  "variables": { "hotel_name": "…" },
  "personalization": { "guest_name": "display_name" },
  "audience_filter": {
    "segment": "prospect",
    "opt_in_only": true,
    "tag": "summer-jazz"
  }
}
```

| Field | Notes |
|-------|--------|
| `channel` | Optional; default **`whatsapp`**. v1 transport is WhatsApp only. |
| `agent_id` | Optional for MVP blasts; recommended when AI reply routing is used later |
| `audience_filter` | Applied at **Launch** against hotel CRM; empty match → `400` |
| Status | `draft` → `running` / `completed` → `cancelled` / `failed` |
| Launch | Only from **`draft`** (`409` otherwise). Send is **async** — use Recipients / Stats |
| Max audience | **2000** matching customers per campaign (`PreviewAudience` returns `count`, `capped`, `max` so the UI can warn before Launch) |

**Launch is not fully idempotent under a duplicate/retried call.** A network
timeout followed by a client retry, while the first call is still processing,
can produce two recipient-insert passes for the same campaign — a per-phone
uniqueness check on the recipient table blocks most duplicates but this is not
a hard guarantee. If a Launch call is ambiguous (timeout, connection drop),
call `Campaigns/Get` to check the resulting `status` before deciding whether to
retry, rather than re-sending Launch blind.

#### Inbox (conversations — human)

| Path | Purpose |
|------|---------|
| `/v1/Events/Inbox/List` | Threads; optional `owner` (`ai` \| `human` \| `escalated`), `search`, `since` |
| `/v1/Events/Inbox/Count` | Open escalation count |
| `/v1/Events/Inbox/Thread` | Messages + session window for one `conversation_id` |
| `/v1/Events/Inbox/Reply` | Free-form staff text (requires open 24h window; claims thread) |
| `/v1/Events/Inbox/ReplyTemplate` | Approved template into thread (allowed when window closed) |
| `/v1/Events/Inbox/Claim` | Take ownership (`owner=human`) |
| `/v1/Events/Inbox/Resolve` | End thread, hand back to AI (`owner=ai`, `status=ended`) |
| `/v1/Events/Inbox/Handback` | Hand back to AI while keeping active |

**Thread / list fields include:**  
`conversation_id`, `owner`, `status`, `guest_phone` / `user_identifier`,
`last_message`, `last_message_at`, plus session window fields (below).

Message roles: guest = `user`; staff/AI outbound = `assistant`.

---

### 5.2 WhatsApp 24-hour session window (API behaviour)

Provider rule: free-form outbound is only allowed if the guest messaged within
the last **24 hours**.

| Field | Meaning |
|-------|---------|
| `session_open` | `true` if free-form `Reply` is allowed |
| `session_expires_at` | When the window ends |
| `session_remaining_seconds` | Seconds left (or `0` / null if closed) |
| `last_inbound_at` | Last guest inbound timestamp |

Returned on **Inbox/List** (per thread) and **Inbox/Thread**.

| Call | When window closed |
|------|---------------------|
| `Inbox/Reply` | `errorCode` **409**, `errorMessage` starts with **`WINDOW_CLOSED:`** |
| `Inbox/ReplyTemplate` | Allowed (template re-engage). Success still leaves `session_open: false` until the guest messages again |

`List` supports `since` (ISO datetime) as a polling cursor for updated threads.

---

### 5.3 Post-MVP — AI agent assist (not required for first slice)

After MVP (audience + blast + human inbox), AI assist can use:

| Path | Purpose |
|------|---------|
| `/v1/Events/Agents/List` | Hotel’s staff-configured AI agents |
| `/v1/Events/Agents/ConfigureKnowledge` | Replace (swap) an agent’s **event-layer** knowledge |

**ConfigureKnowledge** body:

```json
{
  "session_token": "…",
  "agent_id": "uuid",
  "entries": [
    {
      "knowledge_type": "other",
      "language": "en",
      "content": "Event details the agent should ground replies on…"
    }
  ]
}
```

- One call **replaces** that agent’s event-layer knowledge (hotel-wide KB unchanged).  
- Empty `entries` clears event-layer knowledge.  
- Agent create/edit (roles, tools, channels) is **not** on this API (admin portal).

Vendfun does **not** store an “event” entity; event CMS stays outside this API.

---

### 5.4 Dashboard (analytics rollups) — new 2026-07-22

| Path | Purpose |
|------|---------|
| `/v1/Events/Dashboard/Summary` | Workspace-wide totals for a time window: `range` (`7d` \| `30d` \| `90d`, default `30d`) |
| `/v1/Events/Dashboard/Trend` | Time-series for one metric: `metric` (`ctr` \| `delivered` \| `read` \| `clicked`, default `ctr`), `range` (e.g. `"14d"`, default), `granularity` (`day` \| `hour`, default `day`, capped at 168 buckets for `hour`) |

**Summary `data`:**

```json
{
  "messages_sent": 1200,
  "active_campaigns": 3,
  "conversations_active": 42,
  "conversations_awaiting_reply": 5,
  "ctr_percent": 0,
  "ctr_delta_points": 0,
  "funnel": { "sent": 1200, "delivered": 1180, "read": 900, "clicked": 0 }
}
```

**Trend `data`:**

```json
{
  "metric": "ctr",
  "points": [
    { "at": "2026-07-09T00:00:00.000Z", "value": 0 },
    { "at": "2026-07-10T00:00:00.000Z", "value": 0 }
  ]
}
```

**Honest caveat:** `funnel.clicked`, `ctr_percent`, and `ctr_delta_points` are
**always 0** — there is no link-tracking resource yet. The formula is real and
wired correctly; it will start returning live numbers the moment link
tracking ships. `points` never omits a gap bucket (returns `value: 0` instead)
so a chart axis stays evenly spaced.

---

### 5.5 Profile (staff self-service) — new 2026-07-22

| Path | Purpose |
|------|---------|
| `/v1/Events/Profile/Get` | Own profile + notification preferences |
| `/v1/Events/Profile/Update` | Patch `name` / `email` / `phone` / `timezone` / `language` (only these — `role`/`company`/`plan` are read-only). `409` if `email` is already taken by another user |
| `/v1/Events/Profile/Notifications` | Set notification preferences — **partial merge**, send only the key(s) you're changing |

**Profile `data`:**

```json
{
  "name": "Jane Tan",
  "email": "jane@hotel.com",
  "phone": "60123132820",
  "timezone": "Asia/Kuala_Lumpur",
  "language": "en",
  "role": "event_manager",
  "company": "VF Hotel",
  "plan": null,
  "whatsapp_business_number": "601...",
  "member_since": "2026-01-15",
  "notification_preferences": {
    "campaign_completed": true,
    "new_reply_received": true,
    "weekly_performance_report": false
  }
}
```

`plan` is always `null` — there is no plan/tier concept in this system yet.
`role`, `company`, and `whatsapp_business_number` are read-only (ignored if
sent to `Profile/Update`). `notification_preferences` is a keyed object, not
an array, so a future preference can be added without changing the shape —
unknown keys sent by the caller are ignored, not rejected.

---

### 5.6 Outbound webhooks (server-to-server)

**Register your own callback URL — new 2026-07-22.** Previously ops had to
hand-seed `event_webhook_url` after Flutter emailed a URL over; now the
Flutter/dashboard team can self-register it:

| Path | Purpose |
|------|---------|
| `/v1/Events/Webhook/Register` | Set (or update) this hotel's `webhook_url`. **Must start with `https://`** |
| `/v1/Events/Webhook/Test` | Fire one real signed `test_ping` at the registered URL and report the delivery outcome synchronously |

**Register request:**

```json
{ "requestInfo": { "sessionId": "…" }, "webhook_url": "https://your-backend.example.com/vendfun-events" }
```

**Register response `data`:**

```json
{
  "webhook_url": "https://your-backend.example.com/vendfun-events",
  "secret": "…"
}
```

The HMAC secret is **auto-generated on first registration** and **left
untouched on re-registration** — re-POSTing the same or a different
`webhook_url` does not rotate the secret. Save it: it's what you check the
`X-Signature` header against on every event/test you receive (see below).
There is currently no separate "rotate secret" call — re-registering does not
do it.

**Test response `data`:** `{"delivered": true|false, "http_status": 200, "error": null}` —
delivery success/failure is reported as data, not as an API error;
`errorCode` on `Webhook/Test` itself is `0` unless no webhook is registered
yet (`400`).

If ops configures per-hotel `event_webhook_url` (+ optional `event_webhook_secret`)
— whether via the self-service calls above or manually — the platform may POST:

```
POST <event_webhook_url>
Content-Type: application/json
X-Signature: base64(HMAC-SHA1(secret, rawBody))   // when secret is set
```

```json
{
  "eventType": "wa_inbound" | "wa_agent_reply" | "wa_delivery",
  "locationId": "<uuid>",
  "timestamp": "ISO-8601",
  "payload": { }
}
```

| eventType | Main payload fields |
|-----------|---------------------|
| `wa_inbound` | `from`, `text`, `mediaUrl`, `mediaType`, `mimeType` |
| `wa_agent_reply` | `conversationId`, `agentId`, `to`, `text`, `success` |
| `wa_delivery` | `providerMessageId`, delivery status fields, `recipient`, … |

Unconfigured hotels: no-op. Webhooks are best-effort; clients may still poll
`Inbox/List` and `Campaigns/Recipients` / `Stats`.

**Note on real-time delivery to the Flutter app:** this webhook is
server-to-server — Vendfun POSTs to a URL a hotel/ops configures
(`event_webhook_url`), it is not a push channel into an open Flutter session.
If Flutter is a webapp (no FCM/APNs), the intended shape is: Twilio/WhatsApp
event → this webhook → your backend logs it to your DB → your own mechanism
(polling, SSE, WebSocket, etc.) delivers it to the open browser tab. Today
`Inbox/List`'s `since` cursor is the only thing our side directly exposes for
"what's new" polling. Worth a short design conversation before Flutter builds
against one assumption — we don't want to prescribe your delivery mechanism,
just flag that nothing on our side pushes into an open tab today.

---

### 5.7 Experiments (A/B comparison) — new 2026-07-23

**Model: a "variant" is an ordinary campaign.** There is no audience-splitting
automation on our side — no data source exists for CTR (`funnel.clicked` is
always 0, see §5.4), so an automatic "pick the winner" isn't trustworthy
today. Instead: **you** create 2+ normal campaigns via `Campaigns/Create`,
each with its own `audience_filter`/tag (e.g. `"test-a"` / `"test-b"`), launch
them however you like, and this API just groups those campaign_ids under one
experiment for side-by-side reporting — then a human records which one won.
**Nothing here auto-launches a campaign.**

| Path | Purpose |
|------|---------|
| `/v1/Events/Experiments/Create` | Create a draft experiment: `name`, `decision_metric` (`read_rate` \| `delivered_rate`, informational only — see caveat above) |
| `/v1/Events/Experiments/AddVariant` | Attach an existing `campaign_id` (from `Campaigns/Create`) with a `label` (e.g. `"A"`). `404` if the campaign doesn't exist at this hotel, `409` if it's already attached to another experiment, `409` if the experiment isn't a draft |
| `/v1/Events/Experiments/List` | `status` (`all` \| `draft` \| `completed` \| `cancelled`), `limit`, `offset`. Response `data` includes `rows[]` **and** `total` |
| `/v1/Events/Experiments/Get` | One experiment + `variants[]`, each variant enriched with live `campaign_sent_count`/`delivered_count`/`read_count`/`delivered_rate`/`read_rate` (null until the variant's campaign has sent anything) |
| `/v1/Events/Experiments/PromoteWinner` | `winner_variant_id` — **a manual staff decision**, not computed by the API. `400` if the variant doesn't belong to this experiment, `409` if already completed/cancelled. Sets `status: completed`. Does **not** send anything |
| `/v1/Events/Experiments/Cancel` | Abandon a draft experiment (`409` once completed) |

**Get `data` shape:**

```json
{
  "experiment_id": "…", "location_id": "…", "name": "Subject line test",
  "decision_metric": "read_rate", "status": "completed",
  "winner_variant_id": "…", "promoted_at": "2026-07-23T…", "promoted_by": "staff@hotel.com",
  "variants": [
    {
      "experiment_variant_id": "…", "campaign_id": "…", "label": "A",
      "campaign_name": "…", "campaign_status": "completed",
      "campaign_sent_count": 40, "delivered_count": 39, "read_count": 30,
      "delivered_rate": 0.975, "read_rate": 0.75
    },
    { "experiment_variant_id": "…", "campaign_id": "…", "label": "B", "...": "..." }
  ]
}
```

**No `clicked`/CTR field on a variant** — same reason `funnel.clicked` in §5.4
is always 0. If you want to send the winning message to a larger remaining
audience after `PromoteWinner`, that's a normal `Campaigns/Create` +
`Campaigns/Launch` call using the winning variant's `template_key` — this API
doesn't do that for you.

---

## 6. Full path list

```
# Auth
POST /v1/Events/Auth/Login
POST /v1/Events/Auth/Verify
POST /v1/Events/Auth/Logout
POST /v1/Events/Auth/ChangePassword
POST /v1/Events/Auth/ForgotPassword
POST /v1/Events/Auth/ResetPassword

# MVP — audience
POST /v1/Events/Customers/List
POST /v1/Events/Customers/Get
POST /v1/Events/Customers/Upsert
POST /v1/Events/Customers/Update
POST /v1/Events/Customers/Delete
POST /v1/Events/Customers/Import
POST /v1/Events/Customers/Tags

# MVP — templates + blasts
POST /v1/Events/Templates/ApprovedList
POST /v1/Events/Campaigns/List
POST /v1/Events/Campaigns/Get
POST /v1/Events/Campaigns/Create
POST /v1/Events/Campaigns/PreviewAudience
POST /v1/Events/Campaigns/Launch
POST /v1/Events/Campaigns/Cancel
POST /v1/Events/Campaigns/Update
POST /v1/Events/Campaigns/Recipients
POST /v1/Events/Campaigns/Stats

# Experiments (A/B comparison)
POST /v1/Events/Experiments/Create
POST /v1/Events/Experiments/AddVariant
POST /v1/Events/Experiments/List
POST /v1/Events/Experiments/Get
POST /v1/Events/Experiments/PromoteWinner
POST /v1/Events/Experiments/Cancel

# MVP — human inbox
POST /v1/Events/Inbox/List
POST /v1/Events/Inbox/Count
POST /v1/Events/Inbox/Thread
POST /v1/Events/Inbox/Reply
POST /v1/Events/Inbox/ReplyTemplate
POST /v1/Events/Inbox/Claim
POST /v1/Events/Inbox/Resolve
POST /v1/Events/Inbox/Handback

# Dashboard
POST /v1/Events/Dashboard/Summary
POST /v1/Events/Dashboard/Trend

# Profile
POST /v1/Events/Profile/Get
POST /v1/Events/Profile/Update
POST /v1/Events/Profile/Notifications

# Outbound webhook self-registration
POST /v1/Events/Webhook/Register
POST /v1/Events/Webhook/Test

# Post-MVP — AI agent assist
POST /v1/Events/Agents/List
POST /v1/Events/Agents/ConfigureKnowledge
```

---

## 7. STG sandbox (integration testing)

> **STG only** — `https://api.vendfun.cloud` (see §1 naming note if you had
> `api.srv1404676.hstgr.cloud` bookmarked as STG). Tenant: VF Hotel  
> `location_id = 11111111-1234-6789-3255-111111111111`

| Field | Value |
|-------|--------|
| Email | `events.test@vendfun.com` |
| PIN | `48162501` (corrected 2026-07-23 — an earlier version of this doc had a stale 6-digit PIN) |
| Role | `event_manager` |

### Login

```bash
# $APIKEY = your issued X-API-Key (optional for now, send it anyway — see §2)
curl -sS -X POST "https://api.vendfun.cloud/v1/Events/Auth/Login" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $APIKEY" \
  -d "{\"email\":\"events.test@vendfun.com\",\"password\":\"48162501\"}"
```

### Authenticated call pattern

```bash
# $TOKEN = data.session_token from Login
curl -sS -X POST "https://api.vendfun.cloud/v1/Events/Customers/List" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $APIKEY" \
  -d "{\"requestInfo\":{\"sessionId\":\"$TOKEN\"},\"segment\":\"all\",\"limit\":50,\"offset\":0}"
```

### Other useful samples

```bash
# Templates
curl -sS -X POST "https://api.vendfun.cloud/v1/Events/Templates/ApprovedList" \
  -H "Content-Type: application/json" \
  -d "{\"requestInfo\":{\"sessionId\":\"$TOKEN\"}}"

# Inbox
curl -sS -X POST "https://api.vendfun.cloud/v1/Events/Inbox/List" \
  -H "Content-Type: application/json" \
  -d "{\"requestInfo\":{\"sessionId\":\"$TOKEN\"}}"

# Campaigns
curl -sS -X POST "https://api.vendfun.cloud/v1/Events/Campaigns/List" \
  -H "Content-Type: application/json" \
  -d "{\"requestInfo\":{\"sessionId\":\"$TOKEN\"},\"status\":\"all\",\"limit\":20}"

# Dashboard
curl -sS -X POST "https://api.vendfun.cloud/v1/Events/Dashboard/Summary" \
  -H "Content-Type: application/json" \
  -d "{\"requestInfo\":{\"sessionId\":\"$TOKEN\"},\"range\":\"30d\"}"

# Logout
curl -sS -X POST "https://api.vendfun.cloud/v1/Events/Auth/Logout" \
  -H "Content-Type: application/json" \
  -d "{\"requestInfo\":{\"sessionId\":\"$TOKEN\"}}"
```

---

## 8. Contract ownership

| Artefact | Role |
|----------|------|
| OpenAPI `…/docs/openapi.json` | **Authoritative** request/response schemas |
| This document | Auth + endpoint summary for integration |
| Vendfun internal | `_ai/context/WIP_FLUTTER_EVENTS_API.md`, `ENDPOINTS_REFERENCE.md` §2.10 |

When the surface changes, Vendfun updates OpenAPI (and this summary) in the same
release train. Prefer generating client models from OpenAPI.

---

## 9. Coordination channel (todos / decisions / spec diffs / bugs)

For anything that needs **both sides to track and act on** — not a one-off question —
use the shared sync repo instead of ad hoc chat/email:

**[github.com/VendfunAI/hospitality-flutter-sync](https://github.com/VendfunAI/hospitality-flutter-sync)**

| | |
|--|--|
| **Protocol** | [README.md](https://github.com/VendfunAI/hospitality-flutter-sync/blob/main/README.md) |
| **This handoff (durable)** | [docs/FLUTTER_EVENTS_PORTAL_SPEC.md](https://github.com/VendfunAI/hospitality-flutter-sync/blob/main/docs/FLUTTER_EVENTS_PORTAL_SPEC.md) |
| **OpenAPI (authoritative schemas)** | `GET {API_BASE}/docs/openapi.json` |

- One GitHub Issue per item: `type:todo` · `type:decision` · `type:spec-diff` · `type:bug`
- Status: `status:open` → `status:in-review` → `status:decided` / `status:done` (or `status:blocked`)
- Owner: `side:hospitality` / `side:flutter` — whoever owes the next move
- **Read** is public; **label flips** need Write/Triage on the repo (invite Flutter team)
- **Never** paste session tokens, API keys, or real guest PII into issues
- Contract/shape **decisions** need **human sign-off on both sides** before `status:decided`

Use it like this:
- **`type:spec-diff`** — this doc or OpenAPI changed in a client-visible way
- **`type:decision`** — pushback / choice before either side implements
- **`type:todo`** — action items owed by either side
- **`type:bug`** — live call ≠ contract (include redacted repro + `transactionId`)

Flutter agent poll:

```bash
gh issue list --repo VendfunAI/hospitality-flutter-sync --label side:flutter --state open
```

Hospitality agent poll:

```bash
gh issue list --repo VendfunAI/hospitality-flutter-sync --label side:hospitality --state open
```

---

*Ad hoc one-off questions → Vendfun backend directly. Include path, redacted body,
and `responseInfo` / `transactionId`. For anything trackable, open an issue in the
sync repo above (§9).*
