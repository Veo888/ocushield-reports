# Ocushield · Xero API Reporting Protocol
**Last updated:** 15 May 2026  
**Purpose:** Ensure accurate comparison periods in all monthly P&L variance reports

---

## The Problem

Xero's P&L API auto-calculates comparison periods when not explicitly set. This can return **truncated or incorrect date ranges** — confirmed in May 2026 when the March 2026 comparison period auto-calculated as 03-03-26 → 31-03-26 instead of 01-03-26 → 31-03-26, causing Amazon ROW to be understated by ~£14,900 and total revenue by ~£13,100.

**Never rely on Xero's auto-calculated comparison periods. Always pass explicit dates.**

---

## Required API Calls for Monthly Reporting

For a report on month **M of year Y**, make the following **5 explicit calls**:

### 1. Current Month (actuals)
```
start_date: YYYY-MM-01
end_date:   YYYY-MM-[last day]
comparison_start_date: [prior month first day]
comparison_end_date:   [prior month last day]
```

### 2. Prior Month (MoM — explicit, to verify)
```
start_date: [prior month first day]
end_date:   [prior month last day]
comparison_start_date: [month before that, first day]
comparison_end_date:   [month before that, last day]
```
> Always pull prior month explicitly with its own dedicated call. Never rely on the comparison period returned alongside the current month pull.

### 3. Same Month Prior Year (YoY)
```
start_date: [YYYY-1]-MM-01
end_date:   [YYYY-1]-MM-[last day]
comparison_start_date: [two months prior, first day]  ← dummy; not used
comparison_end_date:   [two months prior, last day]   ← dummy; not used
```
> Pull prior year month explicitly. Verify the `current_pnl.period` dates in the response match what was requested before using any figures.

### 4. YTD Current Year
```
start_date: YYYY-01-01
end_date:   YYYY-MM-[last day]
comparison_start_date: [YYYY-1]-01-01
comparison_end_date:   [YYYY-1]-MM-[last day]
```
> Always set both YTD periods explicitly. Xero's auto-comparison for YTD defaults to the prior equivalent *duration* from an arbitrary anchor, not Jan–Apr of the prior year.

### 5. YTD Prior Year (explicit verification pull)
```
start_date: [YYYY-1]-01-01
end_date:   [YYYY-1]-MM-[last day]
```
> Pull separately and cross-check totals against what call #4 returned as its comparison period.

---

## May 2026 Report · Correct Call Pattern

```
Call 1 — May 2026 actuals + April comparison
  start_date:            2026-05-01
  end_date:              2026-05-31
  comparison_start_date: 2026-04-01
  comparison_end_date:   2026-04-30

Call 2 — April 2026 explicit (verify MoM baseline)
  start_date:            2026-04-01
  end_date:              2026-04-30
  comparison_start_date: 2026-03-01
  comparison_end_date:   2026-03-31

Call 3 — May 2025 explicit (YoY)
  start_date:            2025-05-01
  end_date:              2025-05-31
  comparison_start_date: 2025-04-01
  comparison_end_date:   2025-04-30

Call 4 — YTD Jan–May 2026
  start_date:            2026-01-01
  end_date:              2026-05-31
  comparison_start_date: 2025-01-01
  comparison_end_date:   2025-05-31

Call 5 — YTD Jan–May 2025 (explicit verify)
  start_date:            2025-01-01
  end_date:              2025-05-31
```

---

## Validation Checks (run after every pull)

After each API call, verify the following before using any figures:

| Check | What to look for |
|-------|-----------------|
| **Period start date** | `current_pnl.period.start_date` must equal the requested `start_date` exactly |
| **Period end date** | `current_pnl.period.end_date` must equal the requested `end_date` exactly |
| **Comparison start date** | `comparison_pnl.period.start_date` must equal the requested `comparison_start_date` |
| **Comparison end date** | `comparison_pnl.period.end_date` must equal the requested `comparison_end_date` |
| **Revenue sanity check** | Total revenue should be within ±20% of the prior month unless a known spike exists |
| **Cross-call reconciliation** | Prior month total revenue from Call 1 comparison must match Call 2 current period total revenue to the penny |

If any check fails: **re-pull with fully explicit dates before using the data.**

---

## Known Xero API Behaviours to Watch

| Behaviour | Detail |
|-----------|--------|
| Auto-comparison truncation | When comparison period not set, Xero may offset start date by 1–3 days, likely related to week-based period logic |
| YTD auto-comparison | Defaults to prior equivalent duration from an internal anchor, not calendar Jan–prior month end |
| `last_refreshed` timestamp | Reflects Xero cache refresh time, not the period end date — do not confuse these |
| April 2025 anomaly | Despite the comparison label showing `2025-03-02`, the underlying April 2025 figures returned correctly — suggests the label can be misleading even when data is right. Always verify via explicit pull. |

---

## Quick Reference · Month-End Dates

| Month | Last Day |
|-------|----------|
| January | 31 |
| February | 28 (29 in leap year — 2024, 2028) |
| March | 31 |
| April | 30 |
| May | 31 |
| June | 30 |
| July | 31 |
| August | 31 |
| September | 30 |
| October | 31 |
| November | 30 |
| December | 31 |

---

*Protocol established following April 2026 reporting cycle. Review if Xero MCP tool version changes.*
