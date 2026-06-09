# Finance Bot — Architecture

> Reference document for the modular architecture of the Finance Bot.
> Defines reusable building blocks, their contracts, and the build plan.

---

## 1. Design principles

1. **Deterministic logic → code. Natural language / judgment → LLM + skill.**
   Any value that must be exact and consistent is computed in code, never left to
   the LLM to derive.

2. **Modularize only what is IDENTICAL, not what is similar.**
   - Output side (Notion render, Telegram send) is identical across reports → shared sub-workflow.
   - Calculation logic differs per report → lives in each orchestrator.
   - Raw data fetching is the same pattern, parameterized → agnostic sub-workflow.

3. **One source of truth per concept.**
   - Reasoning rules (thresholds, what to list, priority logic) → `skills/*.md` in GitHub.
   - Output shape (Notion page format, Telegram format) → `templates/*.md` in GitHub.
   - Deterministic calculations → code (Build Data).
   - Date correctness → a single place (`Get Financial Data`).

4. **Orchestrators think in Lima calendar days, never UTC.**
   All timezone complexity is encapsulated inside `Get Financial Data`.

5. **This document describes structure, not business rules.**
   Specific rules (color thresholds, amount formats, output templates) live in
   `skills/` and `templates/`, never in this file.

---

## 2. Target architecture

Three layers: a GitHub repo (rules + output shape), shared sub-workflows
(reusable building blocks), and lightweight orchestrators that wire them together.

### Layer 1 — GitHub repo (source of truth for rules + output shape)

| Folder | Purpose | Files |
|---|---|---|
| `skills/` | Reasoning rules (thresholds, what to list, priority logic) | `daily-report.md`, `monthly-report.md`, … |
| `templates/` | Output shape (Notion format, Telegram format) | `daily-report-notion.md`, `daily-report-telegram.md`, … |

> System prompt = skill + template, assembled by the orchestrator.

### Layer 2 — Shared sub-workflows (Execute Workflow, no own trigger)

| Sub-workflow | Input | Output / Role |
|---|---|---|
| **Get Financial Data** | `{from, to}` (Lima days) | Normalized raw data — **owner of the date-correctness logic** |
| **Render Notion Page** | Markdown text | Creates page + nested children (two-step) |
| **Send Telegram** | Text + link | Sanitizes and sends |

### Layer 3 — Orchestrator workflows (lightweight)

| Workflow | Pipeline |
|---|---|
| **Daily Report** | Set Dates → Get Financial Data → Build Data (Daily) → AI Agent (daily skill + template) → Render Notion + Send Telegram |
| **Monthly Report** | Set Dates → Get Financial Data ×2 (target + prev) → Build Data (Monthly) → AI Agent (monthly skill + template) → Render Notion + Send Telegram |
| **Add Expense / Add Income / Router** | Later phases |

---

## 3. Sub-workflow contracts

### 3.1 `Get Financial Data`

**Responsibility:** raw data-fetch layer, agnostic of report type. Does NOT compute
metrics or interpret. It fetches data from Notion for a date range and returns it
normalized and timezone-correct (Lima).

**Input:**
```json
{
  "from": "2026-06-01",   // Lima calendar day (NOT a UTC timestamp)
  "to":   "2026-06-07"    // Lima calendar day
}
```
*(Possible future parameter: `include: ["expenses","incomes","budgets","accounts","account_logs"]`
to fetch only what is needed. Initially fetches everything — cost is trivial.)*

**Output:**
```json
{
  "accounts_pen": [ { "name", "current_balance", "type" } ],
  "accounts_usd": [ { "name", "current_balance", "type" } ],
  "expenses":     [ { "name", "amount", "categoryId", "date", "lima_date": "2026-06-07" } ],
  "incomes":      [ { "name", "amount", "categoryId", "date", "lima_date" } ],
  "budgets":      [ { "name", "category_id", "budget_amount", "period_start", "period_end", ... } ]
}
```

**Date-correctness logic (encapsulated inside):**

Notion stores timestamps in **UTC**. Lima is UTC-5. An expense made on Jun 7 at 9 PM
Lima is stored as Jun 8, 2 AM UTC. This causes two chained problems, each with a fix:

1. **PADDING** — Notion's filter compares in UTC, so boundary rows leak in or out.
   Fix: query Notion with a range **widened by ±1 day** to guarantee no boundary row
   is dropped.
   ```
   Want:       from=2026-06-01, to=2026-06-07  (Lima)
   Ask Notion: 2026-05-31  →  2026-06-08        (+1 day each side)
   ```

2. **CONVERSION** — the true date is the Lima date, not the UTC query date.
   Fix: stamp each row with its real `lima_date`.
   ```javascript
   const limaDate = (utc) => new Intl.DateTimeFormat('en-CA', {
     timeZone: 'America/Lima', year:'numeric', month:'2-digit', day:'2-digit'
   }).format(new Date(utc));
   ```

3. **PRECISE FILTER** — with all rows in hand, keep only those inside the real range.
   ```javascript
   rows = rows
     .map(r => ({ ...r, lima_date: limaDate(r.date) }))
     .filter(r => r.lima_date >= from && r.lima_date <= to);
   ```

**Internal order:** `padding → conversion → filter`.
Padding prevents LOSING boundary rows. Conversion + filter prevents INCLUDING rows
from another day. Together: the list is exactly the movements in that Lima range.

**Downstream benefit:** every row carries a precomputed `lima_date`. Build Data nodes
never touch timezone logic again — they just compare strings:
```javascript
// Daily — "today" subset:
const expensesToday = data.expenses.filter(e => e.lima_date === today);
// Monthly — "this month":
const thisMonth = data.expenses.filter(e => e.lima_date.slice(0,7) === reportMonth);
```

---

### 3.2 `Render Notion Page`

**Responsibility:** convert the LLM's markdown text into a complete Notion page,
including nested children (which the Notion API silently drops if passed inline).

**Input:**
```json
{
  "database_id": "36f868c7-...",
  "title_parts": [ ... ],        // title with date mention
  "properties":  { ... },        // DB properties (numbers, dates, etc.)
  "icon_url":    "https://www.notion.so/icons/document_gray.svg",
  "markdown":    "# Today's Expenses\n---\n..."   // LLM report
}
```

**Output:** `{ "page_id", "url" }`

**Internal logic:**
- `parseInlineRichText` — splits lines into `` `code` `` and `**bold**` segments.
- `parseNotionBlocks` — converts lines into Notion blocks; `↳` lines are mapped as
  children of the parent bullet (`childrenMap`).
- **Two-step:** `POST /v1/pages` with flat blocks → `GET` page blocks →
  `PATCH` each parent block with its children (Notion drops children passed inline).

---

### 3.3 `Send Telegram`

**Responsibility:** send the message to Telegram with correct formatting and a link
to the Notion page.

**Input:**
```json
{
  "chat_id": 1219416663,
  "text":    "<LLM message in Telegram format>",
  "notion_url": "https://www.notion.so/..."
}
```

**Output:** `{ "ok": true, "message_id": ... }`

**Internal logic:**
- `sanitize()` — strips orphan `[text]` (without `(url)`), escapes `_` to avoid
  legacy Markdown parse errors.
- Appends `\n\n📖 [View full report](url)`.
- Sends with `parse_mode: Markdown`.

**Open decision — Telegram message generation:** instead of the LLM generating
telegram and notion separately (risk of inconsistency), generate **Notion only** and
derive Telegram from that content. Two variants:
- **A. Code step** — mechanically extract relevant sections from the Notion text.
  Deterministic, no extra LLM call. Requires converting backticks/`**bold**` to
  Telegram's `*bold*`. *(Tentative recommendation.)*
- **B. Second AI Agent call** — "summarize this Notion into Telegram format". More
  flexible, one extra call.

> To be decided when building Daily Report. Note: option A makes the Telegram format a
> code concern (no `templates/*-telegram.md` LLM template needed); option B keeps the
> Telegram template as an LLM artifact. This choice determines whether telegram
> templates exist in `templates/`.

---

## 4. Skills & Templates (GitHub)

Two kinds of artifact, in two folders inside the single project repo
(`n8n-finance-bot`, public). The same repo holds the docs (`CLAUDE.md`,
`workflow-architecture.md`, `ROADMAP.md`); `project-reference.local.md` is gitignored.

```
github.com/netosnos/n8n-finance-bot/
├── CLAUDE.md  workflow-architecture.md  ROADMAP.md
├── skills/                       # how to reason / decide
│   ├── daily-report.md
│   ├── monthly-report.md
│   ├── add-expense.md            # future
│   └── add-income.md             # future
└── templates/                    # exact output shape
    ├── daily-report-notion.md
    ├── daily-report-telegram.md
    ├── monthly-report-notion.md
    └── monthly-report-telegram.md
```

**skill** = reasoning rules: color thresholds, what to list, priority/action logic,
amount formatting rules, role/context.
**template** = the exact output structure the LLM must fill (Notion blocks, Telegram layout).

Separating them lets output shape change without touching decision logic, and lets
templates be reused.

**Access from n8n:** HTTP GET to the raw URLs:
```
https://raw.githubusercontent.com/netosnos/n8n-finance-bot/main/skills/daily-report.md
https://raw.githubusercontent.com/netosnos/n8n-finance-bot/main/templates/daily-report-notion.md
```

**System prompt assembly (by the orchestrator):**
```
[ skill ]              ← role + rules + priority logic
[ template(s) ]        ← exact output shape
———
(financial data is injected as the USER message, never in the system prompt)
```
Rules and templates come fully BEFORE the output request; data is a separate message.

---

## 5. AI Agent node

The **AI Agent** node handles the system/user separation internally.

```
AI Agent
├── Chat Model: Google Gemini Chat Model (credential ocxdzGp0AXzKatU8)
├── Memory:  (empty — report is one-shot, no conversation)
├── Tools:   (empty for now — future: get_financial_data as a tool)
├── System prompt: {{ skill .md }} + {{ template .md }} (fetched from GitHub)
└── User message:  FINANCIAL DATA FOR {date}:\n{{ financialDataJson }}
```

**API key note:** the Google AI Studio key is authentication only; it is not tied to
any system prompt. The system instruction travels in each request → it lives in the
GitHub `.md` file, one source of truth, same key for all workflows.

**Future:** the Agent can be given a `get_financial_data` tool (a sub-workflow as a
Custom Tool) so it decides when to fetch data instead of pre-fetching. Enables
ad-hoc cases like "give me the report for day X" from the bot.

---

## 6. References

- **Build plan / phases** → [`ROADMAP.md`](./ROADMAP.md)
- **Project reference** (workflow / credential / database IDs, host, timezone) →
  [`CLAUDE.md`](./CLAUDE.md)
- **Business rules** (color thresholds, what to list, amount formats, output templates) →
  `skills/` and `templates/` in the GitHub repo
