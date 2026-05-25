---
name: rocops-ghl-write
description: Scoped GoHighLevel (GHL) write skill for Lead Operations. Performs ONLY (a) contact tag add/remove from a fixed allowlist (`under-contract`, `spanish-cohort`, `handoff-*`, `ae-*`, `lo-*`, `docs-*`, `dpa-*`, `queue:*`, `escalate`, `escalation:leadership`, `do-not-contact`) and (b) opportunity status terminal transitions (`won`, `lost`, `abandoned`). Every write is appended to a JSON-lines audit log under `architect-os/logs/ghl-writes/`. Refuses free-form pipeline stage moves, message sends, contact creation, and custom-field writes — those are out of scope by policy and route through n8n or other skills. NPI is redacted from all log/echo output. GHL sunsets mid-2026; keep dependencies minimal.
---

# GoHighLevel (GHL) write — Lead-Ops scoped mutations

Live, narrowly-scoped write access for the team's GHL workspace. Companion to `rocops-ghl-read` (use that for any lookup before mutating). All other GHL writes (messages, contact creation, free stage moves, custom fields) remain out of scope and stay on n8n.

## API

- **Base URL:** `https://services.leadconnectorhq.com`
- **Auth:** `Authorization: Bearer ${GHL_API_KEY}` (GCP Secret Manager `GHL_API_KEY`)
- **Location:** `Location-Id: ${GHL_LOCATION_ID}` header on v2 endpoints (GCP Secret Manager `GHL_LOCATION_ID`)
- **Version:** `Version: 2021-07-28`
- **User-Agent required** — without it Cloudflare returns 403/1010. Use `User-Agent: rocops-agent/1.0`.

## What this skill writes (v1 scope)

| Operation | HTTP | Endpoint | Allowed values |
|---|---|---|---|
| Add contact tags | `POST` | `/contacts/{contactId}/tags` | tags from allowlist (see below) |
| Remove contact tags | `DELETE` | `/contacts/{contactId}/tags` | tags from allowlist (see below) |
| Set opportunity status | `PUT` | `/opportunities/{opportunityId}/status` | `won`, `lost`, `abandoned` |

**Nothing else.** If the user asks for anything outside this table, refuse and link them to the n8n write-path or the appropriate read skill.

## Tag allowlist (enforced client-side — refuse anything else)

Exact matches:
- `under-contract`
- `spanish-cohort`
- `escalate`
- `escalation:leadership`
- `do-not-contact`

Prefix patterns (the suffix is free-form but the prefix is mandatory):
- `handoff-*`   (e.g. `handoff-ae`, `handoff-uw`, `handoff-docs`)
- `ae-*`        (AE assignment, e.g. `ae-mikec`)
- `lo-*`        (LO assignment, e.g. `lo-yousra`)
- `docs-*`      (docs state, e.g. `docs-received`, `docs-missing`)
- `dpa-*`       (DPA program enrollment, e.g. `dpa-masshousing-25k`, `dpa-one-plus`, `dpa-city-*`)
- `queue:*`     (queue assignment, e.g. `queue:chris-leroux`, `queue:grettel-perez`, `queue:mike-simpson`, `queue:gerard`, `queue:zunaira-asghar`)

Reject (with a clear error) any tag that doesn't match. Do NOT bypass the allowlist with `--force` or similar — if a tag isn't on the list, that's a CTO conversation, not a one-off override.

## Out of scope in v1 — refuse these explicitly

- **Free-form pipeline stage moves** — high blast radius; pipeline stage encodes UW progression. If a stage move is needed, escalate to CTO with the contact + opportunity + intended stage.
- **Conversation / message sends** — handled via n8n + HITL queue.
- **New contact creation** — use the intake webhook.
- **Custom field writes** — separate ticket if needed.
- **Bulk operations >25 contacts** — chunk into batches and require explicit confirmation per batch; never silently fan out a single command to thousands of contacts.

## Required: pre-mutation read

Before any write, look up the contact / opportunity first via `rocops-ghl-read`:

1. Confirm the contact / opportunity exists in the target location.
2. Confirm current tag set / current status — so the write is a real change and not a no-op or accidental overwrite.
3. Capture the "before" state in the audit log (see Audit log section).

If the read returns 404 or empty, do not write. Surface the miss to the user.

## Audit log (mandatory)

Every write — successful or failed — appends one JSON line to:

```
architect-os/logs/ghl-writes/YYYY-MM-DD.jsonl
```

(date is the UTC date of the call). Line shape:

```json
{
  "ts": "2026-05-23T17:14:09.221Z",
  "actor_agent_id": "<PAPERCLIP_AGENT_ID>",
  "run_id": "<PAPERCLIP_RUN_ID>",
  "issue_id": "<PAPERCLIP_TASK_ID-or-null>",
  "operation": "add_tags" | "remove_tags" | "set_opportunity_status",
  "target": {
    "contact_id": "abc123",
    "opportunity_id": null,
    "redacted_ref": "Y***-***-1234"
  },
  "before": { "tags": ["cold","..."] }      | { "status": "open" },
  "request":  { "tags": ["under-contract"] } | { "status": "won" },
  "after":    { "tags": ["cold","under-contract","..."] } | { "status": "won" },
  "response_status": 200,
  "response_error": null
}
```

Rules:
- Create the file's parent dir if missing (`mkdir -p architect-os/logs/ghl-writes`).
- One JSON object per line (jsonl), no trailing comma.
- Even on `4xx` / `5xx`, append a line with `response_status` and `response_error` populated and `after` left null.
- Never write a name, email, or full phone in the log — only `redacted_ref` (see NPI redaction below).

## NPI redaction (echo and logs)

When showing the operator (or writing to the audit log) a name / phone / email tied to a contact:
- name → first initial + last 3 of surname (`Yousra Bekhti` → `Y***-***hti`)
- phone → `+1***-***-{last4}`
- email → `{first-char}***@{domain}`

Same rules as `rocops-ghl-read`. Never echo full NPI back to the operator — they can re-look it up if they need it.

## Recipes

### Add a single allowlisted tag to a contact

```bash
# 1) Confirm contact exists + capture current tags (via rocops-ghl-read pattern)
BEFORE=$(curl -fsS "https://services.leadconnectorhq.com/contacts/$CONTACT_ID" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0" \
  | jq '.contact.tags')

# 2) Mutate
RESP=$(curl -fsS -w "\n%{http_code}" -X POST \
  "https://services.leadconnectorhq.com/contacts/$CONTACT_ID/tags" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Location-Id: $GHL_LOCATION_ID" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0" \
  -H "Content-Type: application/json" \
  -d '{"tags":["under-contract"]}')

# 3) Append audit line (see schema in Audit log section)
```

### Remove an allowlisted tag

```bash
curl -fsS -X DELETE \
  "https://services.leadconnectorhq.com/contacts/$CONTACT_ID/tags" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Location-Id: $GHL_LOCATION_ID" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0" \
  -H "Content-Type: application/json" \
  -d '{"tags":["handoff-ae"]}'
```

### Set opportunity terminal status (won / lost / abandoned)

```bash
curl -fsS -X PUT \
  "https://services.leadconnectorhq.com/opportunities/$OPP_ID/status" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0" \
  -H "Content-Type: application/json" \
  -d '{"status":"won"}'
```

Only `won`, `lost`, `abandoned` are allowed via this skill. `open` reactivation and any `pipelineStageId` change is a CTO conversation.

## Hard rules

1. **Allowlist is final.** If a tag isn't in the allowlist, refuse — no `--force`, no overrides.
2. **User-Agent header MANDATORY** (per `feedback_ghl_api_user_agent_required`).
3. **Read before write.** Always look up the target first so the audit log captures `before` state.
4. **One audit line per call** — successful and failed alike.
5. **NPI redaction always.** Never echo full name/phone/email in operator output or audit log.
6. **No free stage moves.** `PUT /opportunities/{id}` (which can change `pipelineStageId`) is OUT of scope — only the status sub-route is in.
7. **No bulk fan-out.** Anything >25 targets must be split, with the operator confirming each batch.
8. **GHL is sunsetting mid-2026** — don't expand this skill's scope without checking the sunset plan first.
9. **Source-of-truth boundary holds** — for post-contract canonical status changes, write to SF, not GHL.

## When to use this skill

- Operator says "tag X as under-contract" — yes (tag is on allowlist).
- Operator says "handoff to AE Mike on this contact" — yes (`handoff-ae`, `ae-mikec`).
- Operator says "mark this opp won/lost/abandoned" — yes.
- Operator says "remove the `escalate` tag now that it's resolved" — yes.

## When NOT to use this skill

- "Move this opp from `Nurture` to `App Submitted`" — pipeline stage move → CTO.
- "Send this contact an SMS" — n8n write-path (HITL queue).
- "Create a new contact for…" — intake webhook.
- "Update their loan amount custom field" — separate ticket; not in v1.
- "Tag everyone matching X" (bulk) — chunk and confirm per batch.
- "Add a brand new tag we just invented" — propose it to CTO first; do not add it ad hoc.

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| `403` / Cloudflare 1010 | Missing User-Agent | Add `User-Agent: rocops-agent/1.0` |
| `401 Unauthorized` | API key rotated/invalid | Don't retry; escalate to CTO |
| `404` on contact / opp | Wrong location, deleted record, or typo | Verify `GHL_LOCATION_ID`; re-look-up via `rocops-ghl-read` |
| `422` on opportunity status | Already in target status, or transition not allowed | Read the current status; if already correct, treat as no-op and still log it |
| Rate limit | Burst of writes | Sleep + retry once; if it repeats, stop and escalate |

## Reference

- Read companion: `rocops-ghl-read` (always start here)
- Sunset plan: `project_ghl_sunset_plan`
- Source-of-truth boundary: see `rocops-ghl-read` SKILL.md "Source-of-truth boundaries" table
- Gap that prompted v1: ROC-153 (Yousra Bekhti under-contract handoff), ROC-168 (this skill)
- Audit log root: `architect-os/logs/ghl-writes/`
