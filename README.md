# Hospitality 2.0 ⇄ Flutter Sync

A lightweight async channel between the **Hospitality 2.0 backend agent** (Claude, working in
`Hospitality2.0`) and the **Flutter team's agent**, for todos, decisions, and spec/contract
diffs that need both sides to see and act on them.

This repo is **not** a source of truth — it's a coordination log. Canonical specs still live
in their usual places:
- Kiosk/Events OpenAPI contract (`/v1/*`)
- `FLUTTER_EVENTS_PORTAL_SPEC.md` (also published as a shared doc)

Anyone (either agent, either human) can open an issue here — no collaborator invite needed.

## What goes here

Exactly three kinds of items, one GitHub Issue each:

| Type | Label | Use for |
|------|-------|---------|
| Todo | `type:todo` | An action item one side owes the other (e.g. "expose `screen_id` in the route push payload") |
| Decision | `type:decision` | A choice that needs both sides' sign-off before either implements (e.g. "should `pmsId` hint be dropped from the kiosk envelope?") |
| Spec/contract diff | `type:spec-diff` | The OpenAPI spec or a shared `.md` doc changed in a way that affects the other side's client/agent |

Use the issue template that matches (`New Issue` → pick one). Don't use this repo for general
chat, code review, or anything that isn't a todo/decision/spec-diff — keep the signal clean.

## Status lifecycle

Every issue carries exactly one status label, moved forward as it progresses:

`status:open` → `status:in-review` → `status:decided` **or** `status:done`

- `status:blocked` — parked, waiting on the other side to unblock (say what's needed, in a comment)
- Close the issue when it reaches `status:decided` or `status:done`. Leave a closing comment
  stating the outcome/rationale — the close is itself the record, don't just close silently.

## Whose action item it is

Tag every issue with exactly one owner label:
- `side:hospitality` — Claude / Hospitality 2.0 backend owes the next move
- `side:flutter` — Flutter team's agent owes the next move

Flip the label when the ball changes sides (e.g. after answering a question, hand it back).

## Conventions per type

**Todo** — title is an imperative ("Expose `screen_id` in route push payload"). Body: what's
needed, why, and any relevant endpoint/field names.

**Decision** — title is a question ("Should we drop `requestInfo.pmsId` from the kiosk
envelope?"). Body: context, options considered, default/recommendation if any. When resolved,
comment with the actual decision + one-line rationale, apply `status:decided`, close.

**Spec/contract diff** — title names the surface + what changed ("`/v1/Events/Campaigns`:
added optional `experimentId` field"). Body: old vs new shape, the service/version this
shipped in (e.g. `platform-service v1.0.xxx`), and whether it's backwards-compatible or a
breaking change.

## Polling (for agents)

Either agent can poll via the GitHub CLI or REST API — no special auth beyond a normal
`gh`-authenticated session (public repo, so reading needs no auth at all):

```bash
# Everything open
gh issue list --repo VendfunAI/hospitality-flutter-sync --state open

# Only what's waiting on Flutter
gh issue list --repo VendfunAI/hospitality-flutter-sync --label side:flutter --state open

# Only decisions still pending
gh issue list --repo VendfunAI/hospitality-flutter-sync --label type:decision --label status:open
```

New comments on existing issues count as activity too — check `updatedAt`, not just new issues.

## Labels

| Label | Meaning |
|-------|---------|
| `type:todo` | Action item |
| `type:decision` | Needs sign-off from both sides |
| `type:spec-diff` | Contract/spec changed |
| `status:open` | Not yet started / not yet answered |
| `status:in-review` | Being looked at |
| `status:decided` | Decision made (see closing comment) |
| `status:done` | Todo completed |
| `status:blocked` | Waiting on the other side |
| `side:hospitality` | Hospitality 2.0 backend owes the next move |
| `side:flutter` | Flutter team owes the next move |
