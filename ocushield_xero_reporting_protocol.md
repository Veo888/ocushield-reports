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

---

## Issue 2 · FM Restatement / Prior Period Adjustment Contamination
**Identified:** 18 May 2026  
**Affected report:** April 2026

### What happened

The April 2026 API pull returned Eye Screening revenue of £130,824. This appeared plausible given the growing Eye Screening pipeline and was used in the initial report. The FM subsequently identified it as incorrect — the correct April figure per Xero's native P&L was £7,401. The £123,424 discrepancy appears to have been caused by a prior period adjustment or reallocation posted by the FM that was temporarily reflected in the API response before being reversed.

**Impact:** April revenue overstated by £123,424. Net P&L swung from an apparent +£58,305 profit to a confirmed −£65,118 loss.

### Root cause

When the FM makes journal entries, credit note adjustments, or prior period corrections, the API can temporarily return figures that include those in-flight adjustments. The API snapshot reflects the ledger state at the moment of the call — if a correction is mid-posting, the figure may be inflated or deflated relative to the finalised position.

### The fix — mandatory FM sign-off cross-check

**Before publishing any report, cross-check the API `total_income` figure against Xero's own native P&L report.** These should match to the penny. If they don't, the FM has likely made or is making adjustments — wait for the ledger to settle and re-pull.

The Xero native report link is returned in every API response as `xero_report_link`. Always open it and verify total revenue before building the report.

---

## Mandatory Cross-Check · API vs Native Xero Report

Add this as a final step after all 5 API calls:

| Check | Method | Pass condition |
|-------|--------|----------------|
| **Total revenue match** | Open `xero_report_link`; compare `total_income` to native P&L | Match to the penny |
| **Top account sanity** | Compare top 3–5 income accounts from API to native report | No account variance >£100 |
| **FM confirmation** | If FM has made adjustments this month, confirm ledger is closed before pulling | FM sign-off before report build |
| **Unusual spike check** | Any account >2x MoM — verify against native Xero before using | Flag and hold if unexplained |

---

## Issue 3 · Prepayment vs Revenue Misclassification Risk
**Identified:** 18 May 2026

### Context

The £123k Medicash payment (6-month contract May–Oct 2026, paid upfront 1 May) initially appeared in the API data as April Eye Screening revenue. Correct accruals treatment:

- **Cash:** received May, sits as deferred income on balance sheet
- **Revenue:** ~£20,500/month recognised over 6-month term (May–Oct)
- **April P&L:** £0 — contract had not started

### Rule for future reports

When a large Eye Screening or B2B payment appears in the API, always confirm with the FM:
1. Is this a single-month delivery or a multi-month prepayment?
2. Has it been coded as revenue or deferred income in Xero?
3. Does the amount match the native Xero P&L?

### Medicash revenue recognition schedule

| Month | Revenue | Cumulative |
|-------|---------|------------|
| May 2026 | ~£20,500 | £20,500 |
| Jun 2026 | ~£20,500 | £41,000 |
| Jul 2026 | ~£20,500 | £61,500 |
| Aug 2026 | ~£20,500 | £82,000 |
| Sep 2026 | ~£20,500 | £102,500 |
| Oct 2026 | ~£20,500 | £123,000 |

*Exact monthly figure subject to FM confirmation of contract value and start date.*

---

*Protocol last updated 18 May 2026. Issues 1–3 all identified during the April 2026 reporting cycle.*
