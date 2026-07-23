# Hospitality 2.0 ⇄ WhatsApp agent frontend Sync

> **Repo name** `hospitality-flutter-sync` is historical. The Events `/v1/Events/*`
> consumer on this channel is the **WhatsApp agent frontend** (Next.js + proxy),
> not a Flutter app. Side label: **`side:whatsapp-agent-frontend`** (sync #3).

A lightweight **async** channel between:

| Side | Humans | Agents |
|------|--------|--------|
| **Hospitality 2.0** (backend / platform) | Vendfun | Claude, Grok, other coding agents in `Hospitality2.0` |
| **WhatsApp agent frontend** (Events client) | Frontend team | Their coding agent(s) |

…for **todos**, **decisions**, **spec/contract diffs**, and **integration bugs** that need both sides to see and act on them.

This repo is **not** a source of truth for API shapes — it's a **coordination log**.

---

## Canonical contracts (source of truth)

| Artefact | Role | Where |
|----------|------|--------|
| **OpenAPI** | **Authoritative** request/response schemas | STG: `https://api.vendfun.cloud/docs/openapi.json` · PRD: `https://api.vendfun.com/docs/openapi.json` (paths under `/Events/*` and other `/v1/*`) |
| **Events prose handoff** | Auth, env table, envelope rules, endpoint catalogue for Events clients | **[docs/FLUTTER_EVENTS_PORTAL_SPEC.md](./docs/FLUTTER_EVENTS_PORTAL_SPEC.md)** (durable copy; prefer this over any chat/artifact link; filename is historical) |
| **This repo** | Trackable work only (issues) | You are here |

If prose and OpenAPI diverge, **OpenAPI wins** for field shapes. Prefer generating client models from OpenAPI.

**Environments (Events):** use **STG** `https://api.vendfun.cloud` for integration. See the handoff §1 for DEV / STG / PRD.

---

## Access model

| Action | Who |
|--------|-----|
| **Read** issues + docs | Anyone (public repo) — no invite |
| **Open issues / comment** | Any logged-in GitHub user |
| **Apply / flip labels** (`side:*`, `status:*`, …) | Needs **Write** or **Triage** on this repo |

Hospitality maintainers should invite the frontend team (GitHub users or a bot) with **Write** (or Triage) so their agent can flip `side:*` and close work. **tanweilong** already has Write.

---

## What goes here

Exactly **four** kinds of items — **one GitHub Issue each**:

| Type | Label | Use for |
|------|-------|---------|
| Todo | `type:todo` | An action item one side owes the other |
| Decision | `type:decision` | A choice that needs **both sides' human sign-off** before either implements a contract/shape change |
| Spec/contract diff | `type:spec-diff` | OpenAPI or the shared handoff changed in a way that affects the other side |
| Bug / integration issue | `type:bug` | Live call failed or behaviour ≠ contract; include redacted repro |

Use **New Issue → pick the template**. Do **not** use this repo for general chat, code review, or anything outside those four types.

Blank issues are disabled (`config.yml`).

---

## Secrets & PII (hard rule)

**Never** paste into issues or comments:

- `session_token` / `sessionId` values  
- `X-API-Key` or other API keys  
- production secrets or passwords beyond the **published STG sandbox** account in the handoff  
- real guest PII (names, phones, emails of real guests)

**Do** include: path, redacted body shape, `transactionId`, `responseInfo.errorCode` / `errorMessage`, `versionNo`, platform version if known.

---

## Status lifecycle

Every issue carries **exactly one** `status:*` label:

`status:open` → `status:in-review` → `status:decided` **or** `status:done`

- `status:blocked` — parked; say in a comment what unblocks it  
- Close when `status:decided` or `status:done`  
- **Always leave a closing comment** with outcome/rationale — the close is the record  

---

## Whose turn (`side:*`)

Every issue carries **exactly one** owner label:

| Label | Meaning |
|-------|---------|
| `side:hospitality` | Hospitality backend (Claude / Grok / Vendfun) owes the next move |
| `side:whatsapp-agent-frontend` | WhatsApp agent frontend / its agent owes the next move |

Flip the label when the ball changes sides.

---

## Human gate on decisions (important)

- Agents may **draft** options, recommendations, and proposed wording.  
- For **contract / envelope / breaking** changes: apply `status:decided` only after **human sign-off on both sides** (Hospitality decision-maker + frontend lead).  
- Agents may close pure **todos** and **bugs** after the fix is verified (no dual human sign-off required unless the fix changes the public contract — then open a `type:decision` or `type:spec-diff`).

---

## Conventions per type

**Todo** — title is an imperative ("Expose `screen_id` in route push payload"). Body: what's needed, why, endpoints/fields.

**Decision** — title is a question ("Should we drop `requestInfo.pmsId`?"). Body: context, options, recommendation. Resolve with closing comment + `status:decided` + human gate above.

**Spec/contract diff** — title names surface + change ("`/v1/Events/Campaigns`: optional `experimentId`"). Body: old vs new, service version, backwards-compatible?, links (OpenAPI / handoff § / commit), consumer (Events / Kiosk / both), action for the other side.

**Bug** — title is symptom + path. Body: environment, steps, expected vs actual, redacted request/response, `transactionId`, version.

---

## Agent playbooks

### Hospitality agent (Claude / Grok / etc.)

**When the session touches Events, the WhatsApp agent frontend contract, kiosk `/v1/*`, or this channel**, poll first:

```bash
gh issue list --repo VendfunAI/hospitality-flutter-sync \
  --label side:hospitality --state open
```

Also useful:

```bash
gh issue list --repo VendfunAI/hospitality-flutter-sync --state open
gh issue list --repo VendfunAI/hospitality-flutter-sync \
  --label type:decision --label status:open
```

**After shipping a client-visible contract change:**

1. Update OpenAPI + Hospitality handoff (`FLUTTER_EVENTS_PORTAL_SPEC.md`)  
2. Refresh `docs/FLUTTER_EVENTS_PORTAL_SPEC.md` in **this** repo if the prose handoff changed  
3. Open a `type:spec-diff` issue, set `side:whatsapp-agent-frontend`, link version / paths  

### WhatsApp agent frontend

```bash
gh issue list --repo VendfunAI/hospitality-flutter-sync \
  --label side:whatsapp-agent-frontend --state open
```

- Contract unclear / pushback → `type:decision`  
- Action needed from backend → `type:todo` with `side:hospitality`  
- Live mismatch → `type:bug` with redacted repro  

New comments count as activity — check issue `updatedAt`, not only new issues.

---

## Labels (protocol)

| Label | Meaning |
|-------|---------|
| `type:todo` | Action item |
| `type:decision` | Needs sign-off (human gate for contract changes) |
| `type:spec-diff` | Contract/spec changed |
| `type:bug` | Integration / behaviour bug |
| `status:open` | Not yet started / not yet answered |
| `status:in-review` | Being looked at |
| `status:decided` | Decision made (see closing comment) |
| `status:done` | Todo or bug completed |
| `status:blocked` | Waiting on the other side (or external) |
| `side:hospitality` | Hospitality owes next move |
| `side:whatsapp-agent-frontend` | WhatsApp agent frontend owes next move |

---

## Links

- Sync protocol: this README  
- Events handoff (prose): [docs/FLUTTER_EVENTS_PORTAL_SPEC.md](./docs/FLUTTER_EVENTS_PORTAL_SPEC.md)  
- OpenAPI STG: https://api.vendfun.cloud/docs/openapi.json  
- OpenAPI docs UI STG: https://api.vendfun.cloud/docs  
