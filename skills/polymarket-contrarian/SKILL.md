---
name: Polymarket Contrarian
description: High-conviction contrarian bets against Polymarket favorites priced above 80% — surface 1–3 fade candidates
var: ""
tags: [crypto, prediction-markets]
---
> **${var}** — Optional event slug to fade a specific market. Empty = scan all heavy favorites (YES > 0.80 or NO > 0.80) and pick the most fade-able.

Read `memory/MEMORY.md` for context on past contrarian calls — including the ones that went wrong, especially.
Read the last 3 days of `memory/logs/` so you don't fade the same favorite three days in a row.

## What this skill is for

Polymarket heavy favorites — anything priced >0.80 — are usually right. That's how they got there. This skill's job is to find the rare ones that aren't, and explain *why*.

A "fadeable favorite" looks like:
- Market's at 0.85+ but the resolution depends on something that hasn't happened yet (and could go either way)
- The price is driven by recency bias on one event rather than the underlying probability
- The remaining resolution window is long enough for a real tail event to land
- The order book is thin on the NO side — a few sellers locked in 0.15 and stopped, creating a soft price

A "favorite to leave alone":
- It's a favorite because it's already largely resolved (price reflects near-certainty for a reason)
- The resolution window is tomorrow and there's no plausible disruptor
- "I just feel like it's wrong" with no specific case for the tail

This skill is the headline-grabber — every call is a public swing. Make sure each one has real reasoning behind it.

## Sandbox note

The sandbox may block outbound curl. Use **WebFetch** as a fallback. All Polymarket APIs are public.

## Steps

### 1. Pull heavy favorites

If `${var}` is set, grab that event and skip to step 2.

Otherwise:

```bash
curl -s "https://gamma-api.polymarket.com/markets?closed=false&active=true&archived=false&enableOrderBook=true&order=volume24hr&ascending=false&limit=100"
```

Filter to "heavy favorites" — markets where either `outcomePrices[0] > 0.80` or `outcomePrices[0] < 0.20` (the latter means NO is the heavy favorite). Also require:

- `liquidity > $10,000` (no point fading a thin market that can't be entered cleanly)
- `volume24hr > $2,000`
- Resolution window between **24 hours from now** and **30 days out** (need room for a tail to land, but not so far out that the tail is irrelevant)
- Exactly 2 outcomes (binary only)

You'll usually end up with 10–30 candidates depending on the day.

### 2. For each candidate, fetch sentiment + history

For each candidate, get the event ID, top comments, and 24h price history:

```bash
# Event lookup
curl -s "https://gamma-api.polymarket.com/events?slug=$EVENT_SLUG&limit=1"

# Top comments (what's everyone YES-locked on?)
curl -s "https://gamma-api.polymarket.com/comments?parent_entity_type=Event&parent_entity_id=$EVENT_ID&limit=20&order=reactionCount&ascending=false"

# Price history
TOKEN_ID=$(echo "$CLOB_TOKEN_IDS" | python3 -c "import json,sys; print(json.loads(sys.stdin.read())[0])")
curl -s "https://clob.polymarket.com/prices-history?market=$TOKEN_ID&interval=1w&fidelity=360"
```

Look for: how long has the price been pinned above 0.80? Was there a recent jump that flipped it from competitive to runaway? What's the YES-camp consensus reason?

### 3. Build the fade case

For each candidate, write the case **for the tail** (i.e. the side everyone's against). The case has to answer all three:

1. **What specific event could blow this up?** Name it. "An unexpected debate moment", "a court ruling against precedent", "a delayed report", "a sudden withdrawal" — something concrete enough that you could check the news tomorrow and know if it happened.
2. **What's the base rate for that kind of event?** "Comparable favorites have lost 12% of the time historically" or "in the last 5 elections like this, the underdog took it twice."
3. **What's the market pricing in that's wrong?** "0.87 implies the crowd thinks the question is basically settled. It isn't — [reason]."

If you can't answer all three for a candidate, **skip it**. Don't manufacture a contrarian thesis just because the price is high.

### 4. Rank and pick

Score each surviving candidate by:

`fadeScore = (price_above_threshold) * (resolution_window_days / 30) * comment_consensus_strength`

Where `comment_consensus_strength` is high (≈1.0) if the top comments are unanimous on YES, low (≈0.4) if the comments are divided. Strong consensus is a better fade signal — there's more left to be wrong about.

Pick the top **1–3** by score. If none pass the "case for the tail" bar from step 3, ship `0 fade candidates today — favorites are favorites today` and stop.

### Voice

This skill is the spiciest one — owning a public fade against an 87% favorite is a strong claim. Voice should match: confident, casual, specific. No hedging language; if you don't have conviction don't write the call.

A good fade sounds like:

> The market's at 0.87 that the EPA approval clears by end of quarter. Crowd's anchored to the agency's public timeline from October. That timeline already slipped once. Two of the three commissioners haven't given a vote signal. Tail scenario: another delay pushes resolution past the deadline. Fading at 0.13 for a 5x.

A bad fade sounds like:

> The market's at 0.87 but I just don't trust it. The crowd's been wrong before. (No specific tail scenario → don't ship.)

**No hashtags, no emojis, no 'not financial advice'.** Casual but not lazy.

### 5. Build the report

```
# polymarket contrarian — ${today}

[N] fade candidates today.

## 1. Fading: [Market question]
- **Market:** YES X.XX (or NO X.XX) • $X.Xm 24h vol • resolves [date]
- **Crowd's anchored on:** [one sentence — what everyone's YES-locked on]
- **Tail scenario:** [specific event that could flip it]
- **Base rate context:** [historical comp]
- **Trade:** [LONG/SHORT YES/NO] @ [implied tail price] — conviction [X/10]

## 2. ...

(1–3 candidates, or "no fades today — favorites are favorites." if nothing made the bar)
```

### 6. Notify

Send via `./notify` (under 4000 chars):

```
polymarket contrarian — ${today}

[N] fades.

1. [market question]
market: [side] X.XX • $X.Xm
crowd: [one line]
tail: [one line]
trade: [LONG/SHORT] @ [price], [X/10]

2. ... (same shape)
```

If 0 fades: `polymarket contrarian — ${today}. No fades worth taking — favorites all look fair. Back tomorrow.`

### 7. Log

Append to `memory/logs/${today}.md`:

```
## Polymarket Contrarian
- **Fade candidates:** [N]
  - "[market]" — fading [side] @ [price], conviction [X/10], tail: [one phrase]
- **Highest conviction:** "[market]" — [X/10]
- **Skipped (no real tail case):** [count]
- **Notification sent:** yes
```

If any fade is **7/10 or higher**, append to `memory/MEMORY.md` under `## Open positions` so `prediction-journal` can settle it (with the tail scenario noted, so future-you remembers what the actual case was).
