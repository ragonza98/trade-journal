# Uptrendy — Trade Journal

## Overview
Single-file HTML app (`index.html`). No build step, no framework, no npm. Open the file directly in a browser.

**Libraries loaded via CDN:**
- Tailwind CSS (utility styling)
- Chart.js (equity curve and stats charts)
- jsPDF + jsPDF-autotable (PDF export)

**Live deployment:** https://tradepalx.netlify.app/
Netlify auto-deploys on every push to `master` (GitHub: `ragonza98/trade-journal`).

---

## Architecture

### Rendering
The entire UI is re-rendered on every state change. `render()` → `renderApp()` builds HTML strings and injects them into `<div id="app">`. There is no virtual DOM or diffing — just full re-renders.

Navigation is handled by `navigate(view, opts)` which sets `state.view` and calls `render()`.

### Views
`dashboard`, `newTrade`, `history`, `statistics`, `setups`, `settings`, `tradeDetail`, `premarket`, `premarketDetail`, `premarketForm`

### State
One global `state` object holds everything in memory. Persisted to storage on mutations.

```js
state = {
  journals, trades, plans, personalization,
  currentJournalId,
  view, editTradeId,
  historyFilter, statsTab, settingsTab,
  draftTrade,            // autosaved New Trade form
  selectedSetup, selectedPlanId,
  premarketFormDate, premarketListMonth
}
```

---

## Storage

### localStorage (via `DB` object)
Keys are prefixed with `tj_`. All values JSON-serialized.
- `tj_journals` — array of journal objects
- `tj_trades` — array of all trades across all journals
- `tj_plans` — array of premarket plans
- `tj_personalization` — global setups/reasons config
- `tj_currentJournalId` — active journal ID

### IndexedDB (via `ImageDB`)
Stores trade screenshot images by ID (avoids localStorage size limits).

---

## Data Schemas

### Trade
```js
{
  id,           // uid()
  journalId,    // links to a journal
  date,         // 'YYYY-MM-DD'
  setup,        // string from personalization.setups
  direction,    // 'Long' | 'Short'
  outcome,      // 'Win' | 'Loss' | 'Missed' | 'Flat'
  dollarAmount, // always positive; sign inferred from outcome
  realizedR,    // R-multiple (positive number; sign inferred from outcome)
  riskAmount,   // dollar risk
  outcomeReasons, // string[]
  duration,     // minutes (int, computed from tradeTime/endTime)
  tradeTime,    // entry time 'HH:MM'
  endTime,      // exit time 'HH:MM'
  ticker, confidence, entryType, entryTriggers,
  premarketPlanId, hypotheticalOutcome, hypotheticalOutcomeReasons,
  rr, dailyBias, htfPdArray, marketEnvironment,
  executions,   // [{ id, title }] — image refs stored in IndexedDB
  htfImages,    // [{ id, title }]
  emotions, createdAt
}
```

**Outcome rules:**
- `Win` / `Loss` — affect account balance and all P&L stats
- `Missed` — excluded from P&L; tracked separately for `potentialR`
- `Flat` — excluded from P&L and all stats; represents a day with no trading activity

**When `Flat` is selected in the New Trade form**, these sections are hidden (not relevant):
- Direction, Setup, $ Amount, Entry Type, RR, Entry Triggers, Realized R
- Entry Time, Exit Time, Duration
- Executions card
- Outcome Reasons

### Journal
```js
{ id, name, accountSize, initialAccountSize, currency, fixedR, ... }
```
`accountSize` is kept in sync as trades are added/edited/deleted.
`fixedR` is the dollar value of 1R (used to auto-calculate Realized R).

### Personalization (global, shared across all journals)
```js
{
  setups, entryTypes, entryTriggers,
  winReasons, lossReasons, missedReasons, biasReasons
}
```

### Premarket Plan
```js
{ id, journalId, date, bias, dailyRiskLimit, planOutcome,
  analysis, intradayCommentary, emotions, pledgeSigned, ... }
```
`bias`: `'Bullish' | 'Bearish' | 'SnD' | 'None'`
`planOutcome`: `'Followed' | 'Adapted' | "Didn't Follow"`

---

## New Trade Form — Field Order

Date is **outside and above** the Trade Analysis card.

Inside the Trade Analysis card, top to bottom:
1. Outcome + Direction (first row — Outcome drives what else is shown)
2. Outcome Reasons (hidden if Flat)
3. Missed Fields — Hypothetical Outcome (shown only if Missed)
4. Trade Specific Fields — Setup, $Amount, Entry Type, RR, Triggers, Realized R (hidden if Flat)
5. Timing Fields — Entry Time, Exit Time, Duration (hidden if Flat)
6. Ticker / Symbol + Setup Confidence (always visible)

Then: Context card → Executions card (hidden if Flat) → HTF View → Emotions & Notes

---

## Premarket Form — Section Order

1. Session Setup (date, bias, daily risk limit)
2. Key Levels
3. Analysis
4. Chart Screenshots
5. Pre-Session Emotions
6. Intraday Commentary + Intraday Screenshots
7. Plan Outcome
8. Trader's Pledge

---

## Market Environment Options
`Trending` | `Ranging` | `Search and Destroy`

---

## Key Helpers

| Function | Purpose |
|---|---|
| `uid()` | Generates unique IDs |
| `escHtml(s)` | XSS prevention — use for all user content in HTML strings |
| `fmt$(n)` | Formats as `+$1,234.56` |
| `fmtR(n)` | Formats as `+1.23R` |
| `calcStats(trades)` | Computes win rate, P&L, avg R, profit factor, etc. |
| `journalTrades(jid?)` | Returns trades for current (or given) journal |
| `currentJournal()` | Returns active journal object |
| `persistTrades()` | Saves `state.trades` to localStorage |
| `updateMissedFields(outcome)` | Toggles form sections based on selected outcome |

---

## Patterns & Conventions

- **HTML injection:** Build strings, inject into DOM. Never use `innerHTML +=` in loops — build the full string first.
- **XSS:** Always wrap user-supplied strings in `escHtml()` before interpolating into HTML.
- **No `null` coalescing issues:** Use `|| 0` or `|| []` defensively when reading trade fields.
- **Dollar amounts are always positive.** Use `outcome` to determine sign in display and calculations.
- **`draftTrade`** in state autosaves the New Trade form so it survives navigation away and back.
- **New journals start blank** — no default setups or reasons are injected. Personalization is global.
- **Export/Import:** Full data backup via JSON export/import (trades + journals + plans + personalization).
- **Date picker icon:** styled white via `input[type="date"]::-webkit-calendar-picker-indicator { filter: invert(1) }` — needed because app uses a dark background.
- **Native `<select>` dropdowns cannot be styled** — the open dropdown list is OS-rendered. To get custom hover styles, selects must be replaced with custom div-based components.

---

## Badge & Color Reference

| Outcome | Badge class | Color |
|---|---|---|
| Win | `win-badge` | `#10b981` (green) |
| Loss | `loss-badge` | `#ef4444` (red) |
| Missed | `missed-badge` | `#f59e0b` (amber) |
| Flat | `flat-badge` | `#64748b` (gray) |

Primary green (nav active, buttons): `#10b981`
Background: `#0a0f1e` / Card: `#0d1424` / Border: `#1e293b`
