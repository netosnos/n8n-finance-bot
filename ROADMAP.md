# Roadmap — Finance Bot v2

Build plan for the modular architecture. Design lives in
[`workflow-architecture.md`](./workflow-architecture.md).

> Each phase is validated before moving to the next. Nothing replaces a live workflow
> until its replacement is validated end-to-end.

## Phase 0 — Setup
- [ ] Create public GitHub repo `n8n-finance-bot`.
- [ ] Add `skills/` and `templates/` folders.
- [ ] Move existing docs (`CLAUDE.md`, `workflow-architecture.md`, `ROADMAP.md`) into it; keep `project-reference.local.md` gitignored.

## Phase 1 — `Get Financial Data`
- [ ] Sub-workflow with Execute Workflow Trigger (input `{from, to}`).
- [ ] Fetches with padding (±1 day) + pagination + collapse.
- [ ] `lima_date` conversion + precise filter.
- [ ] Normalize output (accounts/expenses/incomes/budgets).
- [ ] Validate against real data.

## Phase 2 — `Render Notion Page` + `Send Telegram`
- [ ] Build `Render Notion Page` sub-workflow (parser + two-step append).
- [ ] Build `Send Telegram` sub-workflow.
- [ ] Validate in isolation.

## Phase 3 — Skills in GitHub
- [ ] Write `skills/daily-report.md` + `templates/daily-report-notion.md`.
- [ ] Write `skills/monthly-report.md` + `templates/monthly-report-notion.md`.

## Phase 4 — Daily Report (first full orchestrator)
- [ ] Set Dates → Get Financial Data → Build Data (Daily) → AI Agent → Render + Send.
- [ ] Resolve the Telegram generation decision (architecture §3.3).
- [ ] Validate end-to-end.

## Phase 5 — Monthly Report
- [ ] Same pattern, with Get Financial Data called ×2 (target + previous month).
- [ ] Validate end-to-end.

## Phase 6 — Add Expense / Add Income / Router
- [ ] Modularize the rest reusing the built blocks.

## Phase 7 — Cutover
- [ ] Activate v2 workflows, deactivate the monolithic ones.
- [ ] Point the Router at the v2 sub-workflows.
