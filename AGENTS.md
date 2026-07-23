# Agent instructions — hospitality-flutter-sync

This repository is an **async coordination channel**, not application source code.

## On session start (either side)

If your work involves the Events API, WhatsApp agent frontend integration, kiosk `/v1/*`, or this channel:

```bash
# Hospitality agents
gh issue list --repo VendfunAI/hospitality-flutter-sync --label side:hospitality --state open

# WhatsApp agent frontend agents
gh issue list --repo VendfunAI/hospitality-flutter-sync --label side:whatsapp-agent-frontend --state open
```

Act on open items before starting unrelated work when they block integration.

## Rules

1. Use issue **templates** only (`todo` / `decision` / `spec-diff` / `bug`).
2. Exactly one `type:*`, one `status:*`, one `side:*` label.
3. **No secrets** in issues (tokens, keys, real guest PII).
4. Contract **decisions** → human sign-off both sides before `status:decided`.
5. Prefer OpenAPI for field shapes; prose handoff for auth/env/catalogue:
   - `docs/FLUTTER_EVENTS_PORTAL_SPEC.md`
   - `https://api.vendfun.cloud/docs/openapi.json` (STG)

Full protocol: [README.md](./README.md).
