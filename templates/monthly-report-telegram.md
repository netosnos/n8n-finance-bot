# Monthly Report — Telegram Template

Output shape for the short monthly summary sent via Telegram. Appended after
`skills/monthly-report.md` (alongside the Notion template) to form the system prompt.

## Rules

Short monthly summary sent via Telegram. Use `*bold*` (single asterisks) for bold,
`` `backticks` `` for amounts. Score color emoji: 🟢 ≥ 75, 🟡 50–74, 🔴 < 50.
Gemini returns a JSON object with keys `"notion"` (full report) and `"telegram"`
(this short message). The "View full report" link is appended automatically by the
workflow.

## Layout

```text
📊 *Monthly Report — [Month YYYY]*
Overall: [🟢/🟡/🔴] *XX/100 ([grade])*

• Budget: *XX/100* [↑/↓]
• Savings: *XX%* [label]
• Net Worth: 🇺🇸 XX.XX USD ([↑/↓] W%)
• Emergency: *X.X mo* / N target
• Investments: *XX/100* toward retirement

🎯 *Next month:* [the lever action]
```
