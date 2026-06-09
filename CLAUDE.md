# CLAUDE.md

Operational guidance for working in this project. Design lives in
[`workflow-architecture.md`](./workflow-architecture.md).

## Working conventions

- **Never replace a live workflow until its replacement is validated end-to-end.**
  v2 workflows are built in parallel and only cut over once fully tested.
- **Skills and templates live in GitHub**, not in Notion pages or hardcoded in nodes.
  The system prompt is assembled as `skill + template` fetched at runtime.
- **Deterministic logic goes in code; natural-language/judgment goes to the LLM + skill.**

## Gotchas

- **Timezone:** the n8n instance runs in **America/Lima**. Cron expressions are
  interpreted in Lima time, not UTC. Notion stores timestamps in UTC, so date filtering
  must convert to Lima and pad the query range by ±1 day (see `Get Financial Data` in
  the architecture doc).
- **n8n MCP stale draft:** `n8n_get_workflow` with mode `full`/`structure` can return a
  stale cached draft. Always verify the live graph with **`mode:active`**. Prefer
  `n8n_update_full_workflow` for structural changes (it ignores cached state). On this
  instance, partial updates auto-publish (activeVersionId advances).
- **`n8n_update_full_workflow` requires the `name` field** or it 400s.
- **Telegram `sendMessage`** returns the full API response — `message_id` is nested under
  `result`, not top-level (`$json.result.message_id`).

## Project reference

Internal IDs (workflows, credentials, databases, host) live in
[`project-reference.local.md`](./project-reference.local.md) — gitignored, not pushed
to the public repo.
