# ROCOps Skills

Paperclip-importable skills used by the [ROCOps](https://github.com/TheArchitectAI) paperclip company agents.

Each skill is a self-contained `SKILL.md` describing how the agent should use a specific capability. Imported into paperclip via:

```sh
curl -X POST "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/skills/import" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -d '{"source":"https://github.com/TheArchitectAI/rocops-skills/<skill-name>"}'
```

## Skills

| Skill | Purpose |
|---|---|
| [rocops-rag-search](./rocops-rag-search) | Semantic search over the architect-os vault |
| [rocops-sf-soql-read](./rocops-sf-soql-read) | Read-only SOQL queries against prod_pipeline Salesforce |
| [rocops-github-pr-read](./rocops-github-pr-read) | Read GitHub PRs across TheArchitectAI org |

Internal use; safe for public repo (no secrets — skills wrap API usage patterns).
