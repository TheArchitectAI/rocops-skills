---
name: rocops-ghl-read
description: Read-only GoHighLevel (GHL) API queries via `https://services.leadconnectorhq.com`. Use when you need pre-contract data — new leads, contact records, SMS/email conversation history, pipeline stages, opportunity status, tag membership, drip campaign enrollment. GHL is canonical for pre-contract; SF is canonical post-contract. Read-only — never mutates.
---

# GoHighLevel (GHL) read via Connection Inc white-label API

Live read-only access to the team's GHL workspace. GHL = nurture engine for **pre-contract** leads (intake, drip, SMS, calls). SF = source of truth for **post-contract** active loans.

## API

- **Base URL:** `https://services.leadconnectorhq.com` (the GHL v2 API; the `app.connectionincorporated.com` URL is the UI white-label, NOT the API)
- **Auth:** `Authorization: Bearer ${GHL_API_KEY}` (from GCP Secret Manager `GHL_API_KEY`)
- **Location:** `Location-Id: ${GHL_LOCATION_ID}` header required on most v2 endpoints (from GCP Secret Manager `GHL_LOCATION_ID`)
- **Version:** `Version: 2021-07-28` header required on v2 endpoints
- **User-Agent required** per [[feedback_ghl_api_user_agent_required]] — without it Cloudflare returns 403/1010. Use anything sensible like `User-Agent: rocops-agent/1.0`.

## Source-of-truth boundaries (CRITICAL — know what lives where)

| Data type | Canonical source | Why |
|---|---|---|
| **New leads (last 7d, lead-stage)** | GHL | Lead-form webhooks land in GHL first; SF Lead query returns 0 by design (`[[feedback_new_leads_in_ghl]]`) |
| **Pre-contract SMS history / drip enrollment / call logs** | GHL | n8n + GHL workflows write here; not synced to SF |
| **Conversation drafts (AI-suggested replies before send)** | GHL Conversations + monorepo HITL queue | Drafts in GHL Conv, AI suggestions in HITL `pending_actions` |
| **Tag membership (cold, refi-candidate, etc.)** | GHL | Tag inventory + state changes |
| **Pipeline stage (opportunity)** | GHL pre-CTC, SF post-CTC | Crossover happens at contract execution |
| **Active loan status / AE assignment** | SF | Post-contract canonical |
| **Realtor attribution** | SF only | `Referred_By__c` / `Realtor_for_Co_Branding__c` — never infer from GHL tags per [[feedback_realtor_attribution_sf_only]] |
| **Paid-ad cohort identity** | Meta Graph API only | Never GHL tags for cohort definition per [[feedback_meta_canonical_no_ghl_crossref]] |

**Consolidation rule:** for any cross-system audit (e.g., "find all 30-day-stale new leads"), match in BOTH systems via phone + email parallel, union the contact IDs, then resolve duplicates by `lastModifiedDate` priority.

## Common queries

### Search contacts by phone / email / name
```bash
curl -fsS -X POST 'https://services.leadconnectorhq.com/contacts/search' \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Location-Id: $GHL_LOCATION_ID" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0" \
  -H "Content-Type: application/json" \
  -d '{
    "locationId": "'"$GHL_LOCATION_ID"'",
    "page": 1,
    "pageLimit": 20,
    "filters": [
      { "field": "phone", "operator": "eq", "value": "+16175551234" }
    ]
  }'
```

### Get a contact's full record (tags, conversations, drips)
```bash
curl -fsS "https://services.leadconnectorhq.com/contacts/$CONTACT_ID" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0"
```

### List conversations for a contact (SMS/email threads)
```bash
curl -fsS "https://services.leadconnectorhq.com/conversations/search?locationId=$GHL_LOCATION_ID&contactId=$CONTACT_ID" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0"
```

### Get messages in a conversation (find first-touch + drift)
```bash
curl -fsS "https://services.leadconnectorhq.com/conversations/$CONV_ID/messages" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0"
```

### Pipeline opportunities (pre-contract stage drift)
```bash
curl -fsS "https://services.leadconnectorhq.com/opportunities/search?locationId=$GHL_LOCATION_ID&pipelineId=2IHsPWsP6qgiUjrqHcY2&limit=50" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0"
```

`GHL_COLD_LEADS_PIPELINE_ID=2IHsPWsP6qgiUjrqHcY2` is the active cold-leads pipeline (per Cloud Run env). `GHL_NEW_LEAD_STAGE_ID=e56b3acd-ec75-4bb2-bbad-63c41a305f57` is the new-lead entry stage.

### Tags (membership inventory)
```bash
curl -fsS "https://services.leadconnectorhq.com/locations/$GHL_LOCATION_ID/tags" \
  -H "Authorization: Bearer $GHL_API_KEY" \
  -H "Version: 2021-07-28" \
  -H "User-Agent: rocops-agent/1.0"
```

## Cross-system consolidation pattern (SF + GHL)

For audits like G13 first-touch, G9 cancel-reason, dedup probes, partner attribution:

1. **Pull GHL contacts** matching the cohort (e.g., last-7d new leads, or stage = Inactive)
2. **Pull SF contacts** matching the same cohort (phone/email/name multi-key parallel)
3. **Union by canonical key** — phone OR email OR (fuzzy name + same city)
4. **Resolve duplicates** — for fields that exist in both, prefer most-recent `LastModifiedDate`/`updatedAt`
5. **Report per-row provenance** — sanitized name + GHL_id + SF_id + which system is canonical for the metric being measured

This is the right pattern for ALL cross-system audits going forward.

## Hard Rules

1. **Read-only.** Never `POST /contacts`, `PUT /contacts/:id`, `POST /conversations/messages`, or any mutating endpoint. Writes happen via n8n workflows.
2. **User-Agent header MANDATORY** (per [[feedback_ghl_api_user_agent_required]]). Without it Cloudflare returns 403/1010.
3. **NPI redaction.** GHL contacts include phone + email + full name. Sanitize in your output: names → first-initial-last-3, phone → `+1***-***-{last4}`, email → `{first-char}***@{domain}`.
4. **Tagging is downstream** (per [[feedback_meta_canonical_no_ghl_crossref]]). Do NOT use GHL tags to define paid-ad cohorts — those come from Meta Graph API. GHL tags are for nurture state only.
5. **GHL is sunsetting Mid-2026** per [[Rearchitect-2026-05-15]] KILL list. Don't build long-term dependencies on GHL data; prefer SF + Meta where the data lives there too.
6. **Realtor attribution NEVER via GHL** (per [[feedback_realtor_attribution_sf_only]]). SF lookups only.
7. **Conversation history requires the conversation ID first** — `GET /conversations/search?contactId=X` to find the conv, then `GET /conversations/{id}/messages` for the body.

## When to use this skill

- "Where did this lead come from?" — check GHL conversation history + source
- "Has this contact been SMS'd recently?" — GHL conversations
- "What pipeline stage is this lead at?" — GHL opportunities
- "Cross-system dedup probe" — combine with rocops-sf-soql-read
- "First-touch automation trace" (G13) — GHL is where the SMS/email actually went out
- "Pre-contract stage drift" — GHL opportunities, not SF stages

## When NOT to use this skill

- Active loan data → SF (`rocops-sf-soql-read`)
- AE assignment → SF
- Closed deal history → SF
- Paid-ad cohort identity → Meta Graph API
- Late-stage loan truth → Cube (per G12 guardrail)
- Writes of any kind → n8n workflow (out of scope for this read-only skill)

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| `403` or Cloudflare 1010 | Missing User-Agent header | Add `User-Agent: rocops-agent/1.0` |
| `401 Unauthorized` | API key rotated or invalid | Don't retry; escalate to CTO |
| `404` on contact lookup | Contact doesn't exist OR lives in different location | Verify `GHL_LOCATION_ID`; not all team contacts are in the same GHL location |
| Empty results on a known-populated query | Filter syntax wrong (GHL v2 uses `filters` array, not flat query params) | Use the `POST /contacts/search` body pattern shown above |
| Slow query (>10s) | GHL rate-limit or pagination | Page through with `page` + `pageLimit` ≤ 100 |

## Reference

- Existing monorepo helpers to model after: `~/Workspace/mortgagearchitect-ai/server/routers/ghlSpeedToLead.ts`, `server/_core/leadIntakeV2.ts`, `server/cron/ghlSlaEnforcement.ts`
- GHL UI (white-label by Connection Inc): `https://app.connectionincorporated.com` (NOT the API; per [[reference_ghl_url_loan_officer_ai_crm]])
- Sunset plan: [[project_ghl_sunset_plan]]
- New leads in GHL: [[feedback_new_leads_in_ghl]]
- Partner CRM architecture: [[reference_partner_crm_sf_plus_ghl]]
