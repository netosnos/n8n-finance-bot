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
- [x] Filter Gmail to the wanted email types by **subject** — `from:procesos@bbva.com.pe`
      alone is NOT enough (16+ types, see finding). **Built 2026-07-09:** broad Gmail Trigger
      → **`Route by Subject` Switch** (extensible, one output per type) → parser. First rule
      routes "Has realizado un consumo con tu tarjeta BBVA" → `Parse BBVA Consumption`; all
      other types fall through and are dropped. Validated: 20 mixed BBVA emails → exactly 9
      consumptions reached the parser (all `ok:true`), the rest dropped. Live flow now
      3 nodes: Gmail Trigger1 → Route by Subject → Parse BBVA Consumption (inactive).
- [x] Parse BBVA consumption fields from the body (`Comercio / Monto / Moneda / Fecha /
      Hora / *NNNN`). **Built + validated 2026-07-08** against 6 real emails — outputs
      `{ok, bank, type, merchant_raw, amount, currency, date (ISO Lima), card_last4,
      account_id, unmapped_card}`. Parses from `snippet` (BBVA emails are HTML-only, no
      text/plain part; the Gmail Trigger's `simple:true` snippet holds every field).
- [x] Create the **Merchant Map** Notion DB (raw bank string → friendly name). **Created
      2026-07-12** inside the Finance Bot page — schema `Raw` (title) + `Friendly`
      (rich_text). IDs in `project-reference.local.md`. (Chose a DB over a repo `.md` /
      table-block so the bot can query by `Raw` and write learned merchants as new rows.)
- [x] **Populated the Merchant Map — 53 rows** (2026-07-13). Harvested distinct
      `merchant_raw` from ~150 real consumption emails (BBVA + Interbank) via the fixed
      parsers, applied proposed friendly names, bulk-inserted with n8n HTTP. Keys derived
      from the fixed parser so they match runtime output exactly. The ~7 ambiguous ones
      (SCO HUANCAYO, RODRIGUEZ DIAZ D, INVERSIONES URBANISTICAS, etc.) were **intentionally
      dropped** by the user (couldn't identify them) → they correctly surface as
      `merchant_new` so Telegram asks + learns them later. **Also fixed an Interbank
      parser bug:** old emails (pre ~Jun 2026) prepend a stray `4` glued to the merchant
      (`4SMARTFIT`); parser now strips a leading `4` when followed by a letter.
- [x] Wire the **Merchant Lookup** — **DONE + validated end-to-end 2026-07-13.** Both parser
      branches converge on `Consolidate Parse Output` (single anchor) → `Query Merchant Map`
      (HTTP `POST /v1/databases/{id}/query`, `notionApi` cred, filter `Raw equals
      {{merchant_raw}}`) → `Apply Merchant Map` (Code: merges lookup back by index, adds
      `merchant_friendly` — Friendly or falls back to `merchant_raw` — and `merchant_new`
      bool). Consolidates both parser branches into one Phase-7 output. Validated via temp
      webhook+mock harness (exec 428): hit→friendly/new:false, miss→raw/new:true; real
      HTTP→Notion call succeeded. Harness removed; workflow back to inactive. Live flow now
      7 nodes: Gmail Trigger1 → Route by Subject → {Parse BBVA, Parse Interbank} →
      Consolidate Parse Output → Query Merchant Map → Apply Merchant Map.
- [x] Store the card→account map (see local ref) — **DONE as code constants** inside each
      parser (`CARD_TO_ACCOUNT`): BBVA `*4008`/`*1218` → Variable Expenses, Interbank
      `*087`/`*5069` → Fixed Expenses. Chose code over Notion config (few, stable cards);
      revisit → Notion only if card count grows or a USD card is added.
- [x] Map card last4 → Notion account (parsers' `CARD_TO_ACCOUNT` → `account_id`); look up
      merchant in Merchant Map + flag if unknown (**= the Merchant Lookup done 2026-07-13**).
- [ ] **Add BBVA PLIN + QR-payment parsers** (user wants these logged → Variable Expenses).
      ⚠️ Nuance: some PLINs are to himself (self-transfer, e.g. "a Ernesto A Angulo J") vs to
      others — decide handling (category is chosen in Telegram/Phase 8, so a self-transfer
      would just get the `Transfer` category; confirm before building this parser).
- [x] Write the Interbank parser. **Built + validated 2026-07-12** vs 5 real emails. Format
      differs from BBVA (`Tarjeta: ****NNN Comercio: X Monto: S/. Y Fecha: DD/MM/YYYY Hora:
      HH:MM AM/PM`) — parser strips `S/.` and converts AM/PM→24h. Cards `*087`/`*5069` →
      Fixed Expenses. Routed via a 2nd Switch rule (`From` contains `netinterbank.com.pe`
      AND `Subject` contains `Tarjeta` → covers Amex/Visa consumption + recurring, excludes
      bills/PLIN/transfers). Live flow now 4 nodes: Gmail Trigger1 → Route by Subject →
      {Parse BBVA Consumption (case 0), Parse Interbank Consumption (case 1)}.

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

- [x] ~~Spike — learn Telegram bot mechanics~~ **Skipped** — validated mechanics directly on
      the real bot (single-user, low risk), incrementally + reversibly instead.
- [x] Design the notification: card is **English** (bot speaks English) —
      "💳 Expense detected / `<merchant>` 🆕 · S/`<amt>` / `<account>` · `<bank>` / Which category?"
      with **5 category recommendations + `📋 View all` + `🗑 Dismiss`**. Built in `nZxRev5elRCHJIy7`
      notify side (Prepare Card → Telegram Send Card → Build Pending Body → HTTP Insert Pending).
      DONE + validated 2026-07-14. (Gemini replaces the 5 fixed picks later.)
- [x] **Category suggestion (Gemini)** — DONE 2026-07-15. `AI: Suggest Categories`
      (`gemini-3.1-flash-lite`) returns exactly 5 picks; the 5 card buttons are fixed slots with
      expression text/callback_data (solves the dynamic-keyboard limit). Prompt currently INLINE
      (TODO: externalize to a GitHub skill per convention).
- [x] **Pending state:** Notion **"Pending Expenses" DB** (n8n Data Tables API not exposed on the
      instance). `Message ID` (title) = lookup key. Row created on notify, archived on resolve.
      DONE 2026-07-13.
- [x] On confirm → call **Add Expense v2**; edit the message to "✅ Logged as `<cat>` — …".
      DONE via `Expense Callback` sub-workflow (`GjN4yoC2v9S5CxZU`). category + discard branches
      built; **`View all` (45-cat keyboard) branch still TODO.**
- [x] Branch the existing bot's Telegram Trigger on `callback_query` vs text (NOT a 2nd bot).
      Router (`2PsOEqcvG0J7mBGioAJJ_`) now listens to `callback_query` + `IF: Is Callback` delegates
      taps to `Execute: Expense Callback`; text path unchanged. **Full cycle validated end-to-end
      via a REAL user tap 2026-07-15 (exec 438): tap Gym → expense written → card edited → row archived.**
- [x] **`View all` / `View less` toggle** — DONE 2026-07-16. `View all` = flat list of ALL live
      categories; `View less` collapses back to the 5 Gemini picks (stored in Pending DB `Picks`).
      Emojis removed except 🗑 Dismiss; `appendAttribution:false` everywhere.
- [x] **Fully dynamic categories** — DONE + validated 2026-07-16. Categories fetched live from Notion
      at runtime (Gemini allowed-list, `name→id`, `Logged as` name, and the View-all keyboard). **Add a
      category in Notion → it appears in the bot, zero manual work.** View-all keyboard built via HTTP to
      the Telegram Bot API (native node can't do variable-length keyboards); bot token in a `Bot Config`
      Set node (**TODO: migrate to `$env.TELEGRAM_BOT_TOKEN` for better security**).
- [x] **New-merchant learning** — DONE + validated 2026-07-16 (Option C: log now, learn + rename on reply).
      On confirm with `merchant_new`, the bot asks for a friendly name via Telegram ForceReply; the reply
      (detected by the Router via `reply_to_message`) → `Finance Bot - Learn Merchant` (`ZYRfCg3nGoAxfYz3`)
      saves `Raw→Friendly` to the Merchant Map AND renames the just-created expense. Validated end-to-end.
- [ ] **Harden edge:** tapping a card whose pending row is already archived → Add Expense v2 throws
      on missing fields; add an IF-after-Build-Expense-Input guard that edits "expired" instead.
- [ ] **Externalize the Gemini prompt** to a GitHub skill (per project convention; currently inline).

> **Now ACTIVE (published):** Router, `Expense Callback` (`GjN4yoC2v9S5CxZU`), Add Expense v2
> (`1eOnQek7DHqealBa`). Identify workflow `nZxRev5elRCHJIy7` stays INACTIVE until we go live on
> real emails (its Gmail Trigger must not poll yet).

> **Existing workflow:** `Finance Bot - Email Expenses` (`nZxRev5elRCHJIy7`, inactive) —
> currently just **Gmail Trigger1** → a Set placeholder. This becomes the Phase 7 identify
> workflow; the Telegram callback handling integrates into the existing bot (Phase 8).
