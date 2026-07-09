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
- [x] Commit + push (AI Agent fetches them from raw GitHub URLs — done, verified HTTP 200 in Phase 4).
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

## Phase 5 — Modularize Add Expense / Add Income / Router 🔨

Modularize the rest of the monolith into reusable sub-workflows, same role as
`Get Financial Data` / `Send Telegram` — build once, reuse.

- [x] **Add Expense** sub-workflow: given a complete expense (merchant/amount/currency/
  date/account/category) → create the Notion Expense; return `{page_id, url}`. Validate
  in isolation.
- [ ] **Add Income** sub-workflow: analogous, for incomes.
- [ ] **Router** modularized to dispatch to the Add Expense / Add Income blocks.

> **Add Expense doubles as the shared block for the Gmail ingestion feature** — Phase 8's
> confirm button calls it, and the manual bot (Router → Add Expense) calls it too. So build
> **Add Expense first** within this phase; it's a dependency of Phase 8. (Build order ≠
> runtime order: at runtime the write fires last, but the button needs the block to exist.)
> Add Income and Router aren't needed by the ingestion feature but stay in Phase 5's scope.
>
> **✅ Add Expense v2 built (`1eOnQek7DHqealBa`, inactive) — 2026-07-07.** Pure write block,
> 4 nodes: Execute Workflow Trigger (`passthrough`) → Code: Build Notion Body → HTTP: Create
> Expense → Code: Return. **Input contract:** `{name, amount, currency:'PEN'|'USD', date
> (ISO, optional→defaults to now Lima), account_id, category_id}`. **Output:** `{page_id,
> url}`. No Gemini, no Telegram — the NL-extraction + confirmation UX stay in the caller
> (Router v2 / Phase 8). Lifts the v1 create body verbatim (icon `arrow-down_gray`, PEN DB
> `273868c7-8890-801f-b858-c15148f3e7fc` / USD DB `273868c7-8890-8090-8400-f85e05c55cd6`,
> Notion cred `wxBq7Vmt6DFteDxh`). Validated end-to-end (test expense created with all
> fields correct, then trashed). v1 Add Expense (`Q1yg4XgqUUH6alxO`) left untouched.

## Phase 6 — Cutover ✅
- [x] Activate the Monthly v2, deactivate the monolithic Monthly v1 (done 2026-06-30).
- [x] (Daily v1 left untouched — Daily dropped from v2.)

> **Monthly v2 (`N9AIhMG9agY3zPQP`) is LIVE.** Cron `0 5 1 * *` (1st, **5am Lima** — moved
> from v1's 8am at user request). v1 is now inactive. First production run: 1 Jul → June
> report vs May. Its sub-workflows
> (Get Financial Data / Render Notion Page / Send Telegram) are active. First run depends on
> June-close Account Logs snapshots existing (same dependency v1 had).

## Gmail Expense Ingestion — the big picture (Phases 5, 7, 8)

Auto-detect bank notification emails, identify the expense, and — **only after the user
confirms via Telegram** — add it to Notion. Three pillars:

- **Identify** (Phase 7) — read the email, produce a structured expense. Pure read.
- **Add** (Phase 5 — Add Expense, shared block) — write the expense to Notion. Pure write.
- **Telegram** (Phase 8) — the interactive glue: present → ask/confirm → call Add.

**Build order = 5 → 7 → 8.** Add Expense (5) and Identify (7) are independent (neither
depends on the other); both must precede Telegram (8), which orchestrates them. Start with
**Phase 5**: it has no blockers (builds against the known Notion schema), while Phase 7 is
waiting on the card→account map + an Interbank sample. (Runtime order differs: the write
fires last, after the Telegram confirm.)

**Flow model — confirm-before-write** (replaces the old "Flow C" auto-save). Nothing enters
Notion without the user's OK. The bot presents the identified expense and either asks for
what's missing (usually the category — context-dependent) or, if it knows everything, just
asks to confirm. One adaptive flow with a smart branch.

## Phase 7 — Identify expenses (email parsing) 🔨 (resumed 2026-07-06)

Read a bank email → output a structured expense `{merchant_raw, amount, currency, date,
time, card_last4}`. No Notion writes here.

**Blockers — RESOLVED 2026-07-07** (card→account map + Interbank in `project-reference.local.md`):
- [x] **Card→account map** (PEN only): BBVA `*4008` & `*1218` → **Variable Expenses**; BBVA
      **PLIN transfers + QR payments** also → Variable Expenses; Interbank Amex `*087`
      (`servicioalcliente@netinterbank.com.pe`) → **Fixed Expenses**.
- [x] **Interbank** sender + card + account known; **email sample still pending** to write
      its parser (build it once a real consumption email is captured).

**Build steps (BBVA card consumption first, then the rest):**
- [ ] Filter Gmail to the wanted email types by **subject** — `from:procesos@bbva.com.pe`
      alone is NOT enough (16+ types, see finding). Start with `Has realizado un consumo con
      tu tarjeta BBVA`.
- [x] Parse BBVA consumption fields from the body (`Comercio / Monto / Moneda / Fecha /
      Hora / *NNNN`). **Built + validated 2026-07-08** against 6 real emails — outputs
      `{ok, bank, type, merchant_raw, amount, currency, date (ISO Lima), card_last4,
      account_id, unmapped_card}`. Parses from `snippet` (BBVA emails are HTML-only, no
      text/plain part; the Gmail Trigger's `simple:true` snippet holds every field).
- [ ] Create the **Merchant Map** Notion DB (raw bank string → friendly name). Needed
      because BBVA truncates the merchant to ~18 chars ("BOBOCHA OPEN PLAZ").
- [ ] Store the card→account map (see local ref) — decide runtime home (Notion config vs
      code constant); leaning Notion so it's editable without touching n8n.
- [ ] Map card last4 → Notion account; look up merchant in Merchant Map (flag if unknown).
- [ ] **Add BBVA PLIN + QR-payment parsers** (user wants these logged → Variable Expenses).
      ⚠️ Nuance: some PLINs are to himself (self-transfer, e.g. "a Ernesto A Angulo J") vs to
      others — decide handling (category is chosen in Telegram/Phase 8, so a self-transfer
      would just get the `Transfer` category; confirm before building this parser).
- [ ] Write the Interbank parser (once a sample email is captured).

> **SCOPE (updated 2026-07-07):** in scope for BBVA = card consumption **+ PLIN transfers
> + QR payments** (all → Variable Expenses, per user). USD is **out of scope for now** —
> **extend to USD once the whole PEN flow works** (planned follow-up).
>
> **FINDING (2026-07-07) — BBVA sends 16+ email types.** Surveyed 300 emails (7 May–7 Jul)
> from `procesos@bbva.com.pe`: 130 "Has realizado un consumo con tu tarjeta BBVA", Transf.
> Interbancaria (46), PLIN (37 + 19 QR), Pago Tarjetas propias (21), T-Cambio (14), pago a
> comercios con QR (7), transf. ctas propias/terceros, Apartados, estados de cuenta, etc.
> **⇒ MUST filter by subject, not just sender.** Build parsers per wanted type.
>
> **Still deferred:** **"La compra… ha sido anulada"** — a reversal; if the expense was
> already added, it should be reverted. For now just notify, don't auto-delete.
>
> **Gotcha:** the Gmail OAuth token (`Gmail account`, `4rFJWwULha1qyy86`) expires/revokes
> and needs periodic manual **Reconnect** in n8n (hit this 2026-07-07). Redirect URI must be
> `https://n8n.netosnos.dev/rest/oauth2-credential/callback`.

## Phase 8 — Telegram bot (interactive: notify / ask / confirm) 🔮

The interactive layer. Presents the identified expense, collects confirmation + any missing
field (category), and on confirm **calls Add Expense (Phase 5)**. Also learns new merchants.

- [ ] **Spike — learn Telegram bot mechanics first** (isolated experiment, don't touch the
      Router): send a message with an inline keyboard, receive the `callback_query`, edit
      the message in place (`editMessageText`), answer with `answerCallbackQuery`.
- [ ] Design the notification: "💳 Consumo detectado: `<merchant>` · S/`<amt>` · `<account>`"
      with buttons — category suggestions (Gemini picks 3–4 of the 44 + "Ver todas") +
      `[✅ Confirmar]` / `[🗑 Descartar]`.
- [ ] **Category suggestion skill** (LLM judgment, skill + template in GitHub). Context-
      dependent categories (`Family`/`Friends`/`Partner`/`Special Ocations`) must be asked;
      some (Fuel→`Fuel`, Spotify→`Spotify`) are inferable → suggest with confidence.
- [ ] **Pending state:** hold the identified expense between notify and tap (callback
      arrives as a separate event). Map `message_id → pending expense` via a small store
      (`callback_data` has a 64-byte limit).
- [ ] On confirm → call **Add Expense**; edit the message to "✅ Categorizado como `<cat>`".
- [ ] Branch the existing bot's Telegram Trigger on `callback_query` vs text (NOT a 2nd
      bot — one webhook per bot).
- [ ] **New-merchant learning:** if the merchant was unknown, ask its friendly name → save
      to the Merchant Map so it's recognized next time.

> **Existing workflow:** `Finance Bot - Email Expenses` (`nZxRev5elRCHJIy7`, inactive) —
> currently just **Gmail Trigger1** → a Set placeholder. This becomes the Phase 7 identify
> workflow; the Telegram callback handling integrates into the existing bot (Phase 8).
