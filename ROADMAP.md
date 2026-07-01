# Roadmap — Finance Bot v2

Build plan for the modular architecture. Design lives in
[`workflow-architecture.md`](./workflow-architecture.md).

> Each phase is validated before moving to the next. Nothing replaces a live workflow
> until its replacement is validated end-to-end.

## Phase 0 — Setup ✅
- [x] Create public GitHub repo `n8n-finance-bot`.
- [x] Add `skills/` and `templates/` folders.
- [x] Move existing docs (`CLAUDE.md`, `workflow-architecture.md`, `ROADMAP.md`) into it; keep `project-reference.local.md` gitignored.

## Phase 1 — `Get Financial Data` ✅
- [x] Sub-workflow with Execute Workflow Trigger (input `{from, to}`).
- [x] Fetches with padding (±1 day) + pagination + collapse.
- [x] `lima_date` conversion + precise filter.
- [x] Normalize output (accounts/expenses/incomes/budgets).
- [x] Validate against real data.

> Built as workflow `Get Financial Data` (15 nodes, inactive). Validated against
> May 2026: 95 PEN expenses, PEN non-transfer total `4148.71` (exact match to the
> Monthly's validated `budgets_pen_spent`), no Lima-boundary leak. Transfers are
> INCLUDED (with parsed `dest`) — the agnostic layer does not interpret/exclude them.
> Each tx row carries `currency` + `lima_date`; budgets carry `currency` and are
> kept by range-overlap with `[from, to]`.

## Phase 2 — `Render Notion Page` + `Send Telegram` ✅
- [x] Build `Render Notion Page` sub-workflow (parser + two-step append).
- [x] Build `Send Telegram` sub-workflow.
- [x] Validate in isolation.

> `Render Notion Page` (10 nodes, inactive): input `{database_id, title_parts, properties,
> icon_url, markdown}` → `{page_id, url}`. Reuses the Daily v1 parser
> (`parseInlineRichText` + `parseNotionBlocks`); two-step create+append for nested `↳`
> children; an IF guarantees `{page_id,url}` is returned even with no children. Validated:
> page rendered with code/bold inline + the `↳` sub-items correctly nested (test page trashed).
>
> `Send Telegram` (6 nodes, inactive): input `{chat_id, text, notion_url}` → `{ok, message_id}`.
> Sanitizes (orphan `[brackets]`, escapes `_`), appends `📖 [View full report](url)`,
> sends `parse_mode: Markdown`. `message_id` read from `$json.result`. Validated (msg sent).

> **SCOPE DECISION (2026-06-22):** v2 ships the **Monthly report only**. The Daily report
> is dropped from v2 — in practice the user found little use for it on v1. The Daily v1
> workflow stays running as-is for now; the user may revisit integrating a Daily v2 later.
> So everywhere below, "report orchestrator" = Monthly.

## Phase 3 — Skills in GitHub ✅ (monthly only)
- [x] Write `skills/monthly-report.md` (rules/calculations S1–S6 only).
- [x] Extract `templates/monthly-report-notion.md` + `templates/monthly-report-telegram.md` from the Notion skill page.
- [ ] Commit + push (so the AI Agent can fetch them from raw GitHub URLs — prerequisite for Phase 4).
- ~~Daily skill/template~~ — dropped from v2 (see scope decision).

> **Monthly skill + templates DONE.** All three ported faithfully from the Notion
> "Monthly Report Skill" page (ID in `project-reference.local.md`): skill = intro +
> input structure + config + S1–S6 rules; the two output-format blocks became
> `templates/monthly-report-{notion,telegram}.md`. System prompt = skill + both templates.

## Phase 4 — Monthly Report orchestrator (v2) ✅
- [x] **Push skill + templates first** (raw URLs resolve, HTTP 200).
- [x] Set Dates (target + prev month, Lima) → Get Financial Data ×2 (target + prev) → Build Data (Monthly) → Gemini (skill + both templates from GitHub) → Render Notion Page + Send Telegram.
- [x] Reconcile Build Data with the new `Get Financial Data` output shape (currency/lima_date on tx, `account_logs`, range-overlap budgets).
- [x] Resolve §3.3 — kept v1's approach: ONE Gemini call returns `{notion, telegram}`.
- [x] Validate end-to-end against May 2026 — composite **83.8 B**, net worth **$30,248.43**, all 6 sections + nested `↳` patterns rendered, Telegram sent. Page properties all correct. Test page trashed.

> Built as workflow **Monthly Report v2** (16 nodes, INACTIVE pending cutover). Sub-workflows
> `Get Financial Data` / `Render Notion Page` / `Send Telegram` are ACTIVE (so the parent can
> reference them). Their triggers use `passthrough` so the orchestrator feeds items directly
> (no resourceMapper). Minor polish noted: Gemini renders the template's `[↑/↓/=]` literally
> with brackets — cosmetic, could refine the template wording later.

## Phase 5 — Add Expense / Add Income / Router
- [ ] Modularize the rest reusing the built blocks.

## Phase 6 — Cutover ✅
- [x] Activate the Monthly v2, deactivate the monolithic Monthly v1 (done 2026-06-30).
- [x] (Daily v1 left untouched — Daily dropped from v2.)

> **Monthly v2 (`N9AIhMG9agY3zPQP`) is LIVE.** Cron `0 5 1 * *` (1st, **5am Lima** — moved
> from v1's 8am at user request). v1 is now inactive. First production run: 1 Jul → June
> report vs May. Its sub-workflows
> (Get Financial Data / Render Notion Page / Send Telegram) are active. First run depends on
> June-close Account Logs snapshots existing (same dependency v1 had).
