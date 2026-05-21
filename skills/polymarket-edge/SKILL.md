---
name: Polymarket Edge
description: Find Polymarket markets that look mispriced versus the news baseline — surface 2–3 high-conviction edges
var: ""
tags: [crypto, prediction-markets]
---
> **${var}** — Optional event slug to check a single market for edge. Empty = scan top movers and surface 2–3 of the best mispricings.

Read `memory/MEMORY.md` for context on recent edge calls.
Read the last 2 days of `memory/logs/` so you don't ship the same mispricing twice.

## What this skill is for

This isn't a feed scrape — it's a hunt. The job is to find markets where the price is meaningfully off from what the news / public sentiment actually justifies. Two or three a day, max. Quality over quantity.

A "good edge" looks like:
- The market's at 0.65 but the underlying news has clearly shifted in the last 24h
- The crowd's anchored to a stale narrative
- A specific event happened that the market hasn't priced in yet
- The order book is thin and a few big YES bids are making the price look hotter than the actual conviction

A "bad edge" — one to skip:
- "Feels off" with no concrete reason
- The gap is small (<5pp) and within market noise
- The reason is "the crowd's just wrong" with no specific evidence

## Sandbox note

The sandbox may block outbound curl. For every curl below, if it fails, use **WebFetch** for the same URL. All Polymarket APIs are public, no auth.

## Steps

### 1. Fetch top movers and high-volume markets

If `${var}` is set, just grab that event and skip to step 2:

```bash
curl -s "https://gamma-api.polymarket.com/events?slug=$EVENT_SLUG&limit=1"
```

Otherwise scan two pools and combine:

```bash
# Pool A — top volume (where the action is)
curl -s "https://gamma-api.polymarket.com/markets?closed=false&active=true&order=volume24hr&ascending=false&limit=30"
```

For each market, also fetch 24h price history to find the biggest movers:

```bash
TOKEN_ID=$(echo "$CLOB_TOKEN_IDS" | python3 -c "import json,sys; print(json.loads(sys.stdin.read())[0])")
curl -s "https://clob.polymarket.com/prices-history?market=$TOKEN_ID&interval=1d&fidelity=60"
```

Compute 24h price change in percentage points (close − open). Mark anything with `|change| >= 4pp` as a "mover".

Filter the combined pool to keep markets where:
- `liquidity > $5,000` and `volume24hr > $2,000`
- At least one YES price in `[0.10, 0.90]`
- Resolves between **12 hours from now** and **30 days out**

### 2. Check the news baseline

For each candidate (cap at 10 so this step doesn't blow up), do a targeted news/sentiment check:

```
WebSearch: "[market question]" news last 24 hours
```

What you're looking for:
- A specific event in the last 24–48h that should have moved the price
- A consensus shift in coverage
- An anchor the crowd's relying on that's now stale

If `XAI_API_KEY` is set, also pull recent X discussion:

```bash
curl -s -X POST "https://api.x.ai/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -d '{
    "model": "grok-4-1-fast",
    "input": [{"role": "user", "content": "Search X for the last 24 hours of discussion about: [MARKET_QUESTION]. Return 3 highest-signal tweets with @handle and a one-line summary of each."}],
    "tools": [{"type": "x_search"}]
  }'
```

### 3. Score the gap

For each candidate, write the case in your head:

- **Market implies:** "P(YES) = X.XX, so the crowd thinks..."
- **News/sentiment implies:** "based on [specific event/article/tweet], a reasonable read is P(YES) ≈ Y.YY"
- **Gap:** `Y - X` in percentage points

Keep only candidates where:
- `|gap| >= 8pp` — anything smaller is noise
- You can name **at least one specific piece of evidence** for the news side. "Vibes" doesn't count.
- The gap isn't already obvious to the comments crowd — if everyone in the comments is already screaming that the price is wrong, the edge probably isn't real or is about to disappear.

### 4. Pick the top 2 or 3

If you have more than 3 valid edges, rank by `|gap| * sqrt(volume24hr / 10000)` — bigger gaps in higher-liquidity markets get priority, but volume doesn't dominate.

If you have **0 edges that pass the bar**, that's a valid answer too — don't manufacture an edge to fill the slot. Send a one-line "no edges today, market's calibrated" via `./notify` and log accordingly.

### Voice

Same as `polymarket-thesis`: smart friend over coffee. Specific. Casual but not cosplay-casual. Cite the event or article that's driving your read. **No hashtags, no emojis, no 'not financial advice'.**

A good edge call sounds like:

> Market's at 0.71 that the Fed cuts in March. That priced in before yesterday's CPI print came in hotter than expected. Two FOMC members on the record this morning walking back cut signals. Right number's closer to 0.50. Gap: 21pp. Long NO at 0.29.

A bad edge call sounds like:

> The market seems too high. The crowd might be overconfident. Worth considering a short. (no specific reason → don't ship)

### 5. Build the report

```
# polymarket edge — ${today}

## Edge 1: [Market question]
- **Market:** YES X.XX • 24h vol $X.Xm • resolves [date]
- **News read:** [one-sentence what's actually happening, cite a source]
- **Gap:** [X]pp ([market value] → [your value])
- **Trade:** [LONG/SHORT YES/NO] @ [price] — conviction [X/10]

## Edge 2: ...

(2–3 edges total, or "no edges today" if nothing passed the bar)
```

### 6. Notify

Send via `./notify` (under 4000 chars):

```
polymarket edge — ${today}

[N] edges today.

1. [market question]
market: YES X.XX • $X.Xm
read: [one sentence]
gap: [X]pp → [LONG/SHORT] @ [price], [X/10]

2. [market question]
... (same shape)

(or "0 edges today — market's calibrated. back tomorrow.")
```

### 7. Log

Append to `memory/logs/${today}.md`:

```
## Polymarket Edge
- **Edges flagged:** [N]
  - "[market]" — gap [X]pp, [LONG/SHORT] @ [price], conviction [X/10]
- **Highest conviction:** "[market]" — [X/10]
- **Skipped (too small a gap):** [count]
- **Notification sent:** yes
```

If any edge is **8/10 conviction or higher**, also append to `memory/MEMORY.md` under `## Open positions` so `prediction-journal` can settle it.
