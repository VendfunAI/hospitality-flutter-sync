---
name: Bug / integration issue
about: Live call failed or behaviour does not match the contract
title: "[path] short symptom"
labels: ["type:bug", "status:open"]
---

<!-- Apply exactly one: side:hospitality OR side:whatsapp-agent-frontend (who should investigate / fix next). -->

**Owed by:** `side:hospitality` / `side:whatsapp-agent-frontend` (apply the matching label)

**Environment:** STG / PRD / other:  
**API base:**  
**When:** (date/time, approx)  
**Client:** WhatsApp agent frontend / Kiosk / curl / other:

**Path:** e.g. `POST /v1/Events/...`

**Steps to reproduce:**
1.
2.

**Expected:**


**Actual:**


**transactionId** (if any):


**responseInfo** (errorCode, errorMessage, versionNo — redact secrets):


**Request shape** (redacted — no session tokens / keys / PII):


**Platform / app version** (if known):


---
_Hard rule: never paste session_token, X-API-Key, production secrets, or real guest PII._
