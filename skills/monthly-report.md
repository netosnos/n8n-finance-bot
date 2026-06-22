# Monthly Report — Skill

Generates the monthly financial report. Covers 6 sections: budget performance,
savings rate, balances & net worth, emergency fund, investments, and monthly
balance assessment.

> **Scope of this file:** reasoning rules and calculations only. The exact output
> shape (Notion page layout, Telegram layout) lives in `templates/` and is appended
> after this skill to form the system prompt. *(Templates not yet ported — see
> ROADMAP Phase 3.)*

## Input Data Structure

You will receive a JSON object with the following fields:

```json
{
  "report_month": "2026-05",
  "report_date": "2026-05-31",
  "days_in_month": 31,
  "exchange_rate": 3.35,
  "budgets_pen": [{ "name": "Partner", "budget_amount": 650.00, "spent_amount": 1196.50, "budget_usage": 1.84, "budget_color": "Red", "variability": "Variable" }],
  "budgets_usd": [{ "name": "Car Installment", "budget_amount": 406.12, "spent_amount": 407.46, "budget_usage": 1.00, "budget_color": "Default" }],
  "budgets_pen_total": 3352.15,
  "budgets_pen_spent": 4086.49,
  "budgets_usd_total": 748.45,
  "budgets_usd_spent": 1185.35,
  "carry_over_pen": 734.34,
  "prev_budgets_pen": [{ "name": "Partner", "budget_amount": 700.00, "spent_amount": 620.00, "budget_usage": 0.89 }],
  "prev_budgets_usd": [{ "name": "Car Installment", "budget_amount": 406.12, "spent_amount": 406.12, "budget_usage": 1.00 }],
  "incomes_pen": [{ "name": "BBVA Points", "amount": 69.00 }],
  "incomes_usd": [{ "name": "Pixelspace Salary", "amount": 3600.00 }],
  "total_income_pen": 124.00,
  "total_income_usd": 3600.34,
  "total_expenses_pen": 4086.49,
  "total_expenses_usd": 1185.35,
  "savings_income_usd": 3600.34,
  "savings_expense_usd": 1685.35,
  "savings_usd": 1914.99,
  "savings_rate_usd": 53.2,
  "prev_month_savings_income_usd": 3500.00,
  "prev_month_savings_expense_usd": 1800.00,
  "prev_month_savings_usd": 1700.00,
  "prev_month_savings_rate_usd": 48.6,
  "accounts_usd": [{ "name": "Salary Hub", "current_balance": 55.19, "type": "Current" }, { "name": "Emergency", "current_balance": 7076.18, "type": "Investment" }, { "name": "Stock Portfolio", "current_balance": 2963.44, "type": "Investment" }, { "name": "Tax Reserve", "current_balance": 1440.00, "type": "Savings" }],
  "prev_month_accounts_usd": [{ "name": "Salary Hub", "balance": 60.00 }, { "name": "Emergency", "balance": 7000.00 }],
  "patrimonio_liquido_usd": 116.26,
  "patrimonio_emergencia_usd": 7076.18,
  "patrimonio_inversion_usd": 23050.99,
  "net_worth_usd": 30243.43,
  "prev_month_net_worth_usd": 29800.00,
  "current_phase": "3. Foundation",
  "current_phase_priority": "Maximize investment accounts, build the habit",
  "next_milestone_usd": 100000.00,
  "gap_to_milestone_usd": 69756.57,
  "avg_monthly_savings_usd": 1807.50,
  "months_to_milestone": 38.6,
  "emergency_balance_usd": 7076.18,
  "emergency_monthly_expense_usd": 2405.20,
  "emergency_months_covered": 2.9,
  "emergency_target_months": 8,
  "emergency_gap_months": 5.1,
  "emergency_gap_usd": 12165.42,
  "emergency_score": 36,
  "prev_month_emergency_balance_usd": 7000.00,
  "prev_month_emergency_months_covered": 2.8,
  "prev_month_emergency_score": 35,
  "investment_accounts": [{ "name": "Stock Portfolio", "balance": 2963.44 }, { "name": "Mutual Fund Primary", "balance": 8928.00 }, { "name": "Mutual Fund Secondary", "balance": 8588.01 }, { "name": "Mutual Fund Extra", "balance": 2571.54 }],
  "investment_balance_usd": 23050.99,
  "prev_month_investment_balance_usd": 22650.00,
  "investment_contribution_usd": 250.00,
  "planned_contribution_min_usd": 200.00,
  "planned_contribution_ideal_usd": 300.00,
  "contribution_compliance_pct": 83.3,
  "real_return_month_pct": 0.67,
  "real_return_annualized_pct": 8.0,
  "expected_return_pct": 7,
  "return_delta_pct": 1.0,
  "current_age": 26,
  "retirement_age": 60,
  "years_to_retirement": 34,
  "projection_usd": 614889.00,
  "retirement_goal_usd": 600000.00,
  "projection_gap_usd": -14889.00,
  "required_monthly_contribution_usd": 0.00,
  "investment_score": 102
}
```

`budget_usage` is pre-calculated per item (1.0 = 100%). `carry_over_pen = max(0, budgets_pen_spent - budgets_pen_total)`. All `savings_*` fields are USD-only (see Section 2): `savings_rate_usd = savings_usd / savings_income_usd × 100`, where `savings_expense_usd` already includes Transfer USD expenses sent to PEN accounts. `prev_budgets_*` and `prev_month_*` fields hold previous-month data for comparison. Use all pre-calculated fields directly.

**Source note:** all `prev_month_*` balance fields (`prev_month_accounts_usd`, `prev_month_net_worth_usd`, `prev_month_emergency_balance_usd`, `prev_month_investment_balance_usd`) come from the **Account Logs** DB (a monthly snapshot of every account balance at month close), NOT the Accounts DBs which hold only the current value. Account Logs has no account type — cross-reference each log against Accounts (USD) by name to classify buckets (Current / Investment / Savings, and to exclude Emergency / Tax Reserve).

## Configuration

- Base Exchange Rate (PEN → USD): 3.35 — edit this bullet to update.
- Date of Birth: 2000-03-05 (current age is computed automatically each month).
- Retirement Target Age: 60 — edit to update.
- Retirement Monthly Expense: $2,000 USD. Retirement Goal: $600,000 USD.
- Expected Annual Return: 7% — edit to update.
- Planned Monthly Investment Contribution: minimum $200, ideal $300 USD.
- Emergency Fund Target: 8 months — edit this bullet to update. Reference ranges by situation are listed in Section 4.

## Section 1 — Budget Performance

Budget Score — computed only from categories with `budget_amount > 0`:

- `total_budgeted = sum(budget_amount)` for budgeted categories
- `total_overage = sum(max(0, spent_amount − budget_amount))` for budgeted categories
- `budget_score = max(0, round(100 − total_overage / total_budgeted × 100, 1))`. Guide: 100=perfect, 90+=good, 70–89=acceptable, <70=over budget.
- Also compute `prev_budget_score` using the same formula applied to `prev_budgets_pen` / `prev_budgets_usd`. Display both scores and the delta: "Budget Score: XX/100 (prev: YY/100) ↑ improved / ↓ worsened / = unchanged".

Category Analysis — list ALL budgeted categories ordered by `budget_usage` desc. Match by name with `prev_budgets_pen` / `prev_budgets_usd` and apply pattern flags:

- 🔴 Over this month AND over prev month (both usage > 1.0) → PATTERN: "2 consecutive months over — budget likely needs recalibration"
- 🔴 Over this month, under prev month (prev usage ≤ 1.0) → FIRST TIME: "First overage this month — monitor next month"
- ⚠️ Current usage dropped > 0.40 vs prev month → DROP FLAG: "Dropped significantly vs last month — verify no missing expenses"
- No match in prev month → show current data only, omit pattern comment

Unbudgeted Expenses — categories with `budget_amount == 0` and `spent_amount > 0`. List separately after the budget table:

- For each: name + spent_amount + note: "No budget assigned this month — consider adding one for next month"

## Section 2 — Savings Rate

All calculations in USD only. Uses pre-computed fields from the JSON: `savings_income_usd`, `savings_expense_usd`, `savings_usd`, `savings_rate_usd` (and `prev_month_*` equivalents).

- `savings_income_usd` = sum of all `incomes_usd` (Transfer category already excluded)
- `savings_expense_usd` = all non-Transfer USD expenses + Transfer USD expenses where name starts with 'To [PEN account]'. The latter represents USD exchanged to PEN — it left the USD world and must count as outflow. USD-to-USD transfers are excluded.
- `savings_usd = savings_income_usd - savings_expense_usd`
- `savings_rate_usd = round(savings_usd / savings_income_usd × 100, 1)`. Thresholds: >30% Excellent ★, 15–29% Good, 5–14% Low, <5% Critical.

Display format:

- Show breakdown: Income: `🇺🇸 savings_income_usd USD` | Expenses: `🇺🇸 savings_expense_usd USD` | Net: `🇺🇸 savings_usd USD`
- Show rate with label and month-over-month delta: "Savings Rate: XX% [label] (prev: YY%) ↑ improved / ↓ worsened / = unchanged"
- If `savings_usd < 0`: flag explicitly — "Negative savings this month: spent more than earned in USD."

## Section 3 — Balances & Net Worth

All net worth is USD. PEN accounts are excluded entirely (spending accounts that empty each month). Tax Reserve (type Savings) is also excluded — it holds tax money owed, not personal wealth.

Net Worth Breakdown — 3 buckets (USD accounts only):

- `patrimonio_liquido_usd` = USD accounts of type Current (e.g. Salary Hub, Exchange Hub, Paypal)
- `patrimonio_emergencia_usd` = the Emergency account balance (type Investment, kept as its own bucket because its purpose is emergency coverage)
- `patrimonio_inversion_usd` = USD accounts of type Investment EXCEPT Emergency (Stock Portfolio / eToro + mutual funds)
- `net_worth_usd = liquido + emergencia + inversion`
- Also list each counted USD account with its `current_balance` and its change vs `prev_month_accounts_usd` (matched by name).

Net Worth Growth — compare `net_worth_usd` vs `prev_month_net_worth_usd`. State grew / shrank by `🇺🇸 X USD` (Y%).

Accumulation Phase — based on `net_worth_usd` (USD thresholds). State `current_phase` and `current_phase_priority`:

- 1. Survival (< $0): Stop the bleeding, budget
- 2. Stability ($0 – 25k): Build emergency fund, kill expensive debt
- 3. Foundation ($25k – 100k): Maximize investment accounts, build the habit
- 4. Acceleration ($100k – 500k): Optimize returns, tax efficiency
- 5. Wealth Building ($500k – 2M): Diversification, tax strategy
- 6. Preservation ($2M – 10M): Protect what is accumulated, estate planning
- 7. Legacy ($10M+): Inheritance, trusts, philanthropy

Nearest Milestone (USD). Milestones: $10k (first cushion), $25k (growth starts helping), $100k (hardest — compounding starts working seriously), $250k (grows faster than your contributions), $500k (Coast FIRE possible), $1M.

- `next_milestone_usd` = smallest milestone above `net_worth_usd`; `gap_to_milestone_usd = next_milestone_usd − net_worth_usd`
- `avg_monthly_savings_usd = (savings_usd + prev_month_savings_usd) / 2`, using `savings_usd` as defined in Section 2
- `months_to_milestone = round(gap_to_milestone_usd / avg_monthly_savings_usd, 1)`, only if avg > 0
- Display: Next milestone: `🇺🇸 X USD` — gap `🇺🇸 Y USD` — ~Z months at current pace.

## Section 4 — Emergency Fund

Uses BOTH PEN and USD expenses (real consumption). All Transfer-category expenses are EXCLUDED here — the PEN side is already counted via conversion.

Monthly Expense Base (USD):

- `emergency_monthly_expense_usd = (total_expenses_pen / exchange_rate) + total_expenses_usd`. Both totals are non-Transfer. This is the actual spending of the closed month, not an estimate.

Months Covered:

- `emergency_months_covered = emergency_balance_usd / emergency_monthly_expense_usd`

Target Months — `emergency_target_months` is set in Configuration (currently 8). Reference by situation:

- Stable employee, no dependents: 3–4 months
- Stable employee, with dependents: 6 months
- Freelancer / independent: 6–9 months
- Own business / variable income: 9–12 months

Gap Analysis (USD):

- `emergency_gap_months = max(0, emergency_target_months − emergency_months_covered)`
- `emergency_gap_usd = max(0, emergency_target_months × emergency_monthly_expense_usd − emergency_balance_usd)`

Month-over-month — compare `emergency_months_covered` vs `prev_month_emergency_months_covered`. State delta and whether closer to or further from target.

Score:

- `emergency_score = round(emergency_months_covered / emergency_target_months × 100)`. Show alongside `prev_month_emergency_score` with direction.
- 100+ Target met • 75–99 Almost there • 50–74 Halfway • 25–49 Just starting • <25 Urgent
- Display all amounts in USD. Example line: Covered: X.X months | Target: N | Gap: `🇺🇸 Y USD` (Z months) | Score: S/100 [label]

## Section 5 — Investments

All amounts USD (no PEN investments). `investment_balance_usd` = USD accounts of type Investment EXCEPT Emergency (Stock Portfolio / eToro + mutual funds) = `patrimonio_inversion_usd` from Section 3. Retirement-focused.

Contribution this month — `investment_contribution_usd` = sum of Transfer-category expenses whose name is "To <investment account>" (transfers INTO investment accounts). These ARE counted here, unlike Sections 2 and 4.

1. Contribution compliance:
   - `contribution_compliance_pct = investment_contribution_usd / planned_contribution_ideal_usd × 100`. Status: >= ideal ($300) on target; >= min ($200) minimum met; < min below minimum. Targets in Configuration.
2. Real return this month:
   - `real_return_month_pct = (investment_balance_usd − prev_month_investment_balance_usd − investment_contribution_usd) / prev_month_investment_balance_usd × 100`
   - `real_return_annualized_pct = real_return_month_pct × 12`
3. Return vs expected:
   - `return_delta_pct = real_return_annualized_pct − expected_return_pct` (7%). Positive → above expected ✅, negative → below ⚠️.
4. Projection to retirement goal (contributions ARE compounded):
   - `years_to_retirement = retirement_age − current_age` (current_age computed from date of birth)
   - `r = expected_return_pct / 100`; `growth_factor = (1 + r)^years_to_retirement`; `annuity_factor = (growth_factor − 1) / r`
   - `projection_usd = investment_balance_usd × growth_factor + (investment_contribution_usd × 12) × annuity_factor`. Uses the REAL monthly contribution, assumed to continue.
   - `projection_gap_usd = retirement_goal_usd − projection_usd` (negative or 0 means on track / ahead)
5. Monthly contribution needed to close the gap (if any):
   - `required_monthly_contribution_usd = max(0, projection_gap_usd) / (annuity_factor × 12)`. The total monthly contribution whose compounded future value covers the gap.

Score:

- `investment_score = round(projection_usd / retirement_goal_usd × 100)`
- 100+ On track or ahead • 75–99 Almost on track • 50–74 Need to increase contribution • <50 Significant gap
- Display amounts in USD. Show: balance, this-month contribution + compliance, real vs expected return with delta, projection vs goal, score, and required extra contribution if gap > 0.

## Section 6 — Month Balance

This section generates NO new data — it combines the five section scores (each 0–100) into one composite score, then derives the narrative.

Section scores (0–100):

- `score_budget` = S1 budget_score
- `score_ahorro` = S2 savings rate score by band: 20%+ → 100, 15–19% → 75, 10–14% → 50, 5–9% → 25, <5% → 0
- `score_patrimonio` = S3 net worth growth: `growth_pct = (net_worth_usd − prev_month_net_worth_usd) / prev_month_net_worth_usd × 100`; `score = clamp(50 + growth_pct × 25, 0, 100)`. So +2% → 100, 0% → 50, −2% → 0.
- `score_emergencia` = S4 emergency_score (capped at 100 for the composite)
- `score_inversion` = S5 investment_score (capped at 100 for the composite)

Composite:

- `score_mensual = score_budget×0.25 + score_ahorro×0.25 + score_emergencia×0.20 + score_inversion×0.20 + score_patrimonio×0.10`
- Weights favor what you can control monthly at 26 (budget + savings); net worth weighs least because it is a consequence of the others.
- Grade: 85–100 A • 70–84 B • 55–69 C • 40–54 D • <40 F. Display: score_mensual/100 [grade].

Analysis — What went well:

- The 2 highest section scores, with one line of context each.

Analysis — What went wrong:

- The 2 lowest section scores, with one line of context each.

Most important adjustment for next month (the lever):

- For each section compute `weighted_gap = (100 − score) × weight`. The lever = the section with the HIGHEST weighted_gap — improving it raises the composite the most (this is low score + high weight combined, not just the lowest score).
- Output ONE specific, concrete action targeting that section, tied to its real numbers from this month.
