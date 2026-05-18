---
name: rocops-rag-search
description: Semantic search over the architect-os vault (memory + wiki) via the RAG service at http://100.127.26.77:8765 (Tailscale-only). Use when you need to find canonical answers to "what did we decide about X", "where did we capture Y", "find the rule on Z" — anything likely written in the vault but not in your loaded system prompt. Returns ranked chunks with file paths so you can Read the full source.
---

# RAG search over architect-os vault

When you need to find canonical answers that aren't in your loaded system prompt, query the RAG service. It's faster + more accurate than guessing or grepping.

## Service

- **Endpoint:** `http://100.127.26.77:8765` (Tailscale-only — accessible from Surface + roclaw + Mac via tailnet)
- **Index:** `/home/dwizy/architect-os` vault, ~23,000 chunks across ~1,800 files (memory + wiki + protocols)
- **Embedding:** bge-m3 via Ollama on Neo's Hive
- **Reindex:** every git commit to architect-os triggers a post-commit hook → POST `/rag/reindex` → Mac-commit-to-Mac-queryable latency ~15 sec

## Health probe (use first)

```bash
curl -fsS -m 5 http://100.127.26.77:8765/rag/health 2>&1 | head -3
```

Healthy response includes `"queue_depth":0,"ollama_up":true`. If unhealthy, fall back to grepping `~/architect-os/vault/` directly.

## Query

```bash
curl -fsS -X POST http://100.127.26.77:8765/rag/query \
  -H "Content-Type: application/json" \
  -d '{"q": "your question", "k": 5}'
```

Returns top-k chunks ranked by semantic similarity:
```json
{
  "results": [
    { "path": "vault/memory/feedback_xyz.md", "score": 0.82, "text": "...excerpt..." },
    ...
  ]
}
```

## Workflow

1. **Probe health.** If unhealthy, grep instead.
2. **Query with a clear question.** Not keywords — phrase it like you'd ask a colleague.
3. **Read top results' full paths via Read tool.** RAG excerpts are previews, not the whole file. Always Read the full source before citing.
4. **Cite using `[[file-basename]]` syntax** in your output comments so future agents can re-find the same source.

## When to use this skill

- "Where did we decide X?" / "What's our rule on Y?" — RAG is faster than reading the whole index
- Cross-domain questions touching multiple memory files at once
- Finding feedback memories that constrain your action (e.g., "is there a rule about Discord?" → RAG finds `feedback_messaging_surfaces.md`)

## When NOT to use this skill

- Looking up a specific file by name → use `Read` directly with the full path
- Listing all files in a directory → use `ls` or `find`
- Counting/aggregating across files → use grep/awk; RAG returns chunks, not full files
- Querying current state (live SF data, live pipeline counts) → RAG is a frozen snapshot; for live data use the appropriate API skill

## Safety constraints

- **Read-only.** This skill never writes to the vault. To add new memory, use a separate write skill.
- **No NPI.** Don't query containing borrower names, SSNs, full addresses. If you need to look up a borrower, do it via SF SOQL.
- **Tailnet only.** The endpoint is not publicly reachable. If you're running outside the tailnet, this skill won't work.

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| `Could not resolve host` | Tailscale down or roclaw offline | Skip skill; grep vault directly |
| `queue_depth > 0` for long | Reindex lagging after big commit | Wait 60s, retry; if persists, grep directly |
| Empty `results: []` | Query too narrow or off-topic | Broaden the query; try synonyms |
| `ollama_up: false` | Hive offline | Skip skill; grep vault directly |

## Reference

- Full service docs: `~/architect-os/vault/memory/reference_rag_service.md`
- Wiki search alternative when RAG is down: `~/architect-os/vault/wiki/INDEX.md`
- RAG service code: `~/architect-os/services/rag/`
