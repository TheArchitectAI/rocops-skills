---
name: rocops-sf-soql-read
description: Read-only SOQL queries against the prod_pipeline Salesforce org using the sf CLI. Use when you need pipeline data, Contact/Loan/Transaction_Property records, AE-assignment audits, partner attribution lookups, or any other live SF read. Never mutates. Multi-key parallel match is mandatory for dedup/attribution.
---

# Salesforce SOQL read via sf CLI

Live read-only access to the prod_pipeline org. All AA Mortgage canonical data lives here.

## Org

- **Org alias:** `prod_pipeline`
- **Org ID:** `00D8V0000028EhjUAE`
- **Auth:** JWT via `SF_JWT_*` secrets (already wired into the sf CLI on this machine — no need to log in)
- **Single human user:** `roc@ccm.com` (SysAdmin). All API queries run as this user.

## Health probe (optional)

```bash
sf org display -o prod_pipeline 2>&1 | head -10
```

Should show Status `Connected`. If not, `sf` auth has drifted — escalate to CTO (don't try to re-auth from within a skill).

## Query pattern

```bash
sf data query \
  --target-org prod_pipeline \
  --query "SELECT Id, Name, Stage_Status__c FROM Loan__c WHERE Active_Loan_Indicator__c = true LIMIT 10" \
  --result-format human
```

Or JSON for parsing:
```bash
sf data query --target-org prod_pipeline \
  --query "SELECT COUNT() FROM Contact WHERE Account_Executive__c = '003PX00000TOYYCYA5'" \
  --json
```

## CRITICAL: describe before querying custom objects

The ROC org uses non-standard field names on custom objects. Per [[feedback_describe_before_query]], always run describe first:

```bash
sf sobject describe --target-org prod_pipeline --sobject Transaction_Property__c \
  --json | jq '.result.fields[] | {name, type, custom}' | head -30
```

Specifically: `Transaction_Property__c` uses `Borrower_Name__c`, NOT `ContactId`.

## CRITICAL: LO assignment field

LO assignment is **NEVER `Owner`** (per [[feedback_sf_account_executive_canonical]]). Owner is always the team queue "The ROC Mortgage Group". Use `Account_Executive__c`:

| Team member | Contact ID |
|---|---|
| Ivan Duarte | `003PX00000TOYYCYA5` |
| Michael Simpson | `003PX00000WeYtfYAF` (OOO 5/15-5/29) |
| Yauvan Kumar | `0038V00002gyAi3QAE` |
| Zunaira Asghar | `003PX00000IhSrsYAF` |

## CRITICAL: multi-key parallel matching for dedup

Per [[feedback_multi_key_matching]] — single-key loses 5-10% of dupes. Match in parallel on:
- Phone (4 fields): `Phone`, `MobilePhone`, `HomePhone`, `OtherPhone`
- Email
- Name fuzzy (first-initial-last-3)

Run each as a separate query, union the Contact IDs, then verify.

## Realtor attribution

Per [[feedback_realtor_attribution_sf_only]], use ID comparison not name:
- `Referred_By__c` — referring partner (Contact lookup)
- `Realtor_for_Co_Branding__c` — co-marketing partner
- `Buyer's_Agent__c` — buyer's agent on the deal

Empty `Referred_By__c` = no attribution = safe to include in cohort.

## Known field traps

| Field | Trap |
|---|---|
| `Total_PITI__c` | **Formula field.** Do NOT include in PATCH bodies (rolls back the whole write). Per [[feedback_total_piti_is_formula_field_2026-05-16]]. |
| `Last_Touch__c` | Capped at 140 chars on Loan, 255 on Contact. |
| `Stage__c` on `Transaction_Property__c` | Legacy values exist in records but not in picklist — `/start` filters under-count by ~5%. Per [[feedback_jungo_sf_sync_gap_late_stages_2026-05-16]]. |

## NPI redaction in your output

Always sanitize in comments/outputs:
- Names → first-initial-last-3 (e.g., `M. Garc`)
- SSN → `[REDACTED]`
- Loan numbers → `LOAN-{last4}`
- Addresses → `[REDACTED-ADDR]`

The raw SOQL result has full data — redact when you write it back to issue comments or memory.

## When to use this skill

- Pipeline audits (loan counts, stage distribution, SLA breaches)
- AE assignment verification
- Partner attribution audits (Researcher's primary use)
- Contact lookups by phone/email for multi-key match
- Dedup probes
- Loan-detail pulls for specific cases

## When NOT to use this skill

- Writes — this skill is read-only. Writes go through n8n (which uses the same JWT user but is the canonical write path).
- GHL data — GHL is downstream, lags, conflates sources. SF is canonical (per [[feedback_meta_canonical_no_ghl_crossref]]).
- Late-stage loan truth — Jungo→SF sync stops at est_close. Use Cube via a different skill for late-stage reads.

## Safety constraints

- **NEVER `sf data create/update/upsert/delete`.** This skill exists for SOQL only. If you need a write, escalate to a write-capable surface.
- **`sf describe` BEFORE SOQL on custom objects.** Field names matter.
- **Source of truth hierarchy:** Meta Graph API → SF (you) → Blend live. Never use GHL for cohort definition.
- **NPI in outputs is a P0 violation.** Always redact.

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| `sf data query` fails with auth error | JWT expired or secret rotated | Don't retry; escalate to CTO |
| Field not found | Wrong field API name | Run `sf sobject describe` and correct the query |
| Empty result on a known-populated query | Hit org-level row limit, or filter wrong | Re-check WHERE clause; check picklist values for stage drift |
| Slow query (>30s) | Selecting too many fields, no indexed WHERE | Trim SELECT; add indexed filter (e.g., `IsActive=true`) |

## Reference

- SF troubleshooting doc: `/home/dwizy/Workspace/mortgagearchitect-ai/docs/SALESFORCE_TROUBLESHOOTING.md`
- Single human user: `[[feedback_sf_org_single_human_user_2026-05-17]]`
- Pipeline Deep Audit (canonical pipeline state): `[[../wiki/Pipeline-Deep-Audit-2026-05-16]]`
- LOA Daily Protocol v2 (SLA windows): `[[reference_loa_daily_protocol_2026-05-16]]`
