---
name: Prediction Journal
description: Track open paper positions, settle resolved ones, compute running PnL — keeps the agent honest about its own calls
var: ""
tags: [crypto, prediction-markets]
---
> **${var}** — Optional market slug to settle just that one position. Empty = run the full daily journal: refresh open positions, settle resolved ones, recompute running PnL.

Read `memory/MEMORY.md` — specifically the `## Open positions` section, which is the source of truth for what we're tracking.
Read `memory/logs/` from the last 14 days to find the original thesis text for each open position (so the settlement entry can reference the case at entry).

## What this skill is for

This is the agent's accountability ledger. Every position the other skills open (`polymarket-thesis`, `polymarket-edge`, `polymarket-contrarian`) gets written to `memory/MEMORY.md` under `## Open positions`. This skill is the only one that closes them.

A position is "open" until:
- The underlying market resolved (we know YES or NO)
- The market closed without resolving (rare but happens — defer to manual)
- We explicitly close it (paper-trade equivalent of "I'm out, taking the loss")

The output is two things:
1. **Today's settlement journal** — what closed, what we got right, what we got wrong, the PnL on each
2. **Open positions update** — for each still-open position, current price vs entry, P/L unrealized

Paper-trade only. Sizes are notional — `$100` per position by convention from the source skills.

## Sandbox note

The sandbox may block outbound curl. Use **WebFetch** as a fallback for any URL fetch. Polymarket APIs are public.

## Steps

### 1. Read open positions

Parse `memory/MEMORY.md` for the `## Open positions` section. Each entry should look like:

```
- [skill] [DATE] — "[Market question]" ([market slug])
  side: [LONG/SHORT YES/NO] • entry: 0.XX • conviction: X/10 • size: $100
  thesis: [one-line summary of the call]
```

If the section is missing or empty, ship a `no open positions to settle, journal empty` notification and stop. Don't manufacture entries.

If `${var}` is set, filter to just that market slug.

### 2. For each open position, fetch current market state

```bash
curl -s "https://gamma-api.polymarket.com/markets?slug=$MARKET_SLUG"
```

Look at:
- `closed` — boolean. If true, the market resolved.
- `archived` — boolean. If true, the market closed without resolving (very rare).
- `umaResolutionStatuses` or the resolved outcome fields — `outcomePrices` of `[1, 0]` means YES resolved; `[0, 1]` means NO resolved.
- `endDate` — the resolution deadline.
- `outcomePrices` — if still open, this is the current YES price.

If the markets API doesn't surface a clean "resolved YES vs NO" for closed markets, cross-check via the event:

```bash
curl -s "https://gamma-api.polymarket.com/events?slug=$EVENT_SLUG&limit=1"
```

### 3. Classify each position

For each open position, classify as one of:

- **resolved-correct** — market closed, our side won
- **resolved-incorrect** — market closed, our side lost
- **still-open** — market still live, not closed
- **expired-unresolved** — endDate passed but market didn't resolve cleanly (defer to manual, flag for review)

### 4. Compute PnL on each settled position

Paper-trade convention: notional `$100` per position. YES tokens pay `$1` if YES wins, `$0` if not. NO tokens are the inverse.

For a **LONG YES** position entered at `entry_price`:
- If YES wins: `pnl = $100 * (1 - entry_price) / entry_price` (you bought shares for `entry_price` each, they each pay $1)
- If NO wins: `pnl = -$100` (full loss of the $100 stake)

For a **LONG NO** (or equivalently **SHORT YES**) entered at `(1 - entry_price)`:
- If NO wins: `pnl = $100 * entry_price / (1 - entry_price)`
- If YES wins: `pnl = -$100`

Round to whole dollars in the output. Don't pretend precision we don't have.

### 5. Compute unrealized PnL on still-open positions

For each still-open position, do the same math but using the *current* YES price as the implied outcome value. This is "what would the position be worth if I sold right now."

- LONG YES at 0.40, current price 0.60: unrealized = `$100 * (0.60 - 0.40) / 0.40 = +$50`
- LONG YES at 0.40, current price 0.25: unrealized = `$100 * (0.25 - 0.40) / 0.40 = -$37`

### 6. Update the running PnL ledger

Read `memory/journal-pnl.md`. If it doesn't exist, create it with a header. Append today's settlements to a `## Settled` section with date, market, side, entry, exit, PnL. Update the running total at the top.

Schema:

```
# ZER0 Prediction Journal — running PnL

**Running total (all-time):** $[+/-X]
**Settled positions:** [N]
**Win rate:** [N correct] / [N settled] = [X%]
**Open positions:** [N], unrealized [+/-$X]

---

## Settled

| Date | Market | Side | Entry | Resolved | PnL |
|------|--------|------|-------|----------|-----|
| 2026-05-20 | "[question]" | LONG YES | 0.42 | YES | +$138 |
| ... |
```

Recompute the header totals from the table on every run — don't trust incremental math.

### 7. Update memory/MEMORY.md

Remove settled positions from the `## Open positions` section. For still-open positions, leave them as-is (the entry data shouldn't drift). Optionally annotate with `[current: 0.XX, unrealized [+/-$X]]` so future runs of other skills can see where positions stand without re-fetching.

### Voice

This one's the calm reporter. Not snarky, not contrarian — just honest about what we got right and wrong. The whole point of this skill is showing the work, including the losses.

A good entry:

> Settled: SHORT YES on "Trump 2028 GOP nomination" entered at 0.78 last week. Resolved NO. +$28 paper PnL. Thesis at entry was that the crowd was anchored on the December poll; that read held up. Conviction at entry: 7/10.

A bad entry:

> Closed a position, won some money. (vague → don't ship.)

When a position loses, **lead with it**. Don't bury losses behind wins. The point is the public PnL.

**No hashtags, no emojis, no 'not financial advice'.** Honest is the brand here.

### 8. Build the report

```
# prediction journal — ${today}

Running PnL: [+/-$X] across [N] settled • Win rate: [X%]
Open positions: [N] • Unrealized: [+/-$X]

## Settled today: [N]

1. ✓/✗ [Market question]
- side: [LONG/SHORT YES/NO] @ [entry] → resolved [YES/NO]
- pnl: [+/-$X]
- entry thesis: "[one line]"
- retro: [one line — was the thesis right for the right reason, right for the wrong reason, or just wrong]

(repeat for each settled position)

## Open positions: [N]

- [skill] "[market]" — [side] @ [entry] / now [current] / unrealized [+/-$X] / [X] days to resolve

(repeat for each open position)

(If 0 settled: "no resolutions today. [N] open positions tracking.")
```

(Use `✓` and `✗` as plain text characters, not emojis — they're U+2713 and U+2717, render as text.)

### 9. Notify

Send via `./notify` (under 4000 chars):

```
prediction journal — ${today}

running: [+/-$X] / [N] settled / [X%] win
open: [N] / unrealized [+/-$X]

settled today: [N]
[for each: ✓/✗ "[market]" [+/-$X] — [side] @ [entry] → [resolved]]

open watch:
[1–3 most notable open positions: largest unrealized, closest to resolution]
```

### 10. Log

Append to `memory/logs/${today}.md`:

```
## Prediction Journal
- **Settled today:** [N] ([N correct], [N incorrect])
- **PnL today:** [+/-$X]
- **Running total:** [+/-$X] across [N] settled
- **Win rate:** [X%]
- **Open positions:** [N], unrealized [+/-$X]
- **Notification sent:** yes
```
