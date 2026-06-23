# Monthly Report — Notion Page Template

Output shape for the full monthly report rendered as a Notion page. Appended after
`skills/monthly-report.md` to form the system prompt.

## Rules

Each section is a `heading_1` followed by a divider, then content; use `heading_2`
for subsections. No emojis in titles. List EVERY item in every data array (ALL
budget categories in `budgets_pen` / `budgets_usd`, ALL accounts in `accounts_usd`,
ALL unbudgeted) — NEVER summarize, condense, or omit items. For each budget category
whose `pattern` field is non-empty, you MUST add an indented child sub-item (a new
line starting with `↳`) directly below that category bullet, containing exactly the
pattern text — never inline it on the bullet, and never skip a non-empty pattern.
Categories with an empty pattern get no sub-item. Format all amounts with thousands
separators and 2 decimals (e.g. `🇺🇸 1,145.00 USD`). Use backtick inline code for
amounts: `🇵🇪 XX.XX PEN` or `🇺🇸 XX.XX USD`. Use **bold** for key figures and
↑/↓/= for month-over-month direction. Score color emoji (only for scores):
🟢 ≥ 75, 🟡 50–74, 🔴 < 50.

**Placeholders:** square brackets `[...]` in the layout below mark a value to fill in.
Output the value ONLY — never the brackets themselves (write `↓`, not `[↓]`; write `🔴`,
not `[🔴]`; write `B`, not `[B]`). For a `[↑/↓/=]` marker emit exactly ONE symbol:
`↑` if the metric improved vs last month, `↓` if it worsened, `=` if unchanged. For a
`[🟢/🟡/🔴]` marker emit exactly one emoji per the score thresholds.

## Layout

```text
Monthly financial report for [Month YYYY]. Overall score: **XX/100 ([grade])**.

# Budget Performance
---
Budget Score: **XX/100** (prev: YY/100) [↑/↓/=]
## PEN Budget
[List EVERY item in budgets_pen as a bullet. When the item pattern field is non-empty, add exactly ONE indented child line starting with ↳ directly below that bullet, containing only the pattern text. Items whose pattern is empty get NO child line:]
• [name] — `🇵🇪 [spent] PEN` of `🇵🇪 [budget] PEN` ([usage]%)
↳ [pattern text — present this child line ONLY when the item pattern is non-empty]
## USD Budget
[Same for budgets_usd:]
• [name] — `🇺🇸 [spent] USD` of `🇺🇸 [budget] USD` ([usage]%)
↳ [pattern text, only when non-empty]
## Unbudgeted Expenses
[List EVERY item in unbudgeted_pen and unbudgeted_usd:]
• [name] — `🇵🇪/🇺🇸 [spent] PEN/USD` spent, no budget assigned
Next month: set the Extra Expense budget to `🇵🇪 [carry_over_pen] PEN` (this month PEN overage carried forward). [omit this line if carry_over_pen is 0]

# Savings Rate
---
Savings Rate: **XX%** [label] (prev: YY%) [↑/↓/=]
• Income: `🇺🇸 XX.XX USD`
• Expenses: `🇺🇸 XX.XX USD`
• Net saved: `🇺🇸 XX.XX USD`

# Balances & Net Worth
---
Net Worth: `🇺🇸 XX.XX USD` — [grew/shrank] `🇺🇸 ZZ.ZZ USD` (W%) vs last month
• Liquid: `🇺🇸 XX.XX USD`
• Emergency: `🇺🇸 XX.XX USD`
• Investments: `🇺🇸 XX.XX USD`
## Accounts
[List EVERY account in accounts_usd, do not omit any even if change is 0:]
• [name] — `🇺🇸 XX.XX USD` ([↑/↓] `🇺🇸 ZZ.ZZ USD`)
## Phase & Milestone
• Phase: [N. Name] — [priority]
• Next milestone: `🇺🇸 XX.XX USD` — gap `🇺🇸 YY.YY USD` — ~Z months

# Emergency Fund
---
Score: [🟢/🟡/🔴] **XX/100** ([status])
• Covered: X.X months (target: N)
• Monthly expense base: `🇺🇸 XX.XX USD`
• Gap: `🇺🇸 YY.YY USD` (Z months)
• vs last month: [closer/further] (YY → XX months)

# Investments
---
Score: [🟢/🟡/🔴] **XX/100** ([status])
• Balance: `🇺🇸 XX.XX USD`
• Contribution: `🇺🇸 XX.XX USD` (ZZ% of `🇺🇸 300.00 USD` ideal)
• Real return: XX% annualized vs 7% expected ([+/−]W% [✅/⚠️])
• Projection at 60: `🇺🇸 XX.XX USD` vs goal `🇺🇸 600,000.00 USD`
• To close gap: contribute `🇺🇸 YY.YY USD`/month  [omit if on track]

# Month Balance
---
Overall: [🟢/🟡/🔴] **XX/100 ([grade])**
## What went well
• [Section] (XX/100) — [context]
• [Section] (XX/100) — [context]
## What went wrong
• [Section] (XX/100) — [context]
• [Section] (XX/100) — [context]
## Most important adjustment
**[One concrete action on the lever section, tied to real numbers]**
```
