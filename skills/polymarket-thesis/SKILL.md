---
name: Polymarket Thesis
description: Daily take on three top Polymarket markets — one-paragraph thesis on each with side, price, and conviction
var: ""
tags: [crypto, prediction-markets]
---
> **${var}** — Optional event slug to write a single-market thesis on. Empty = pick three from today's top markets.

Read `memory/MEMORY.md` for context on recent thesis runs.
Read the last 3 days of `memory/logs/` to see which markets have been covered — don't repeat the same take two days in a row.

## What this skill is for

A daily morning drop: three Polymarket markets that matter today, with a real opinion on each. Not a feed scrape — a take. Each thesis is one substantive paragraph: what the market is asking, where it's priced, what I think the right number is, and why.

This is the flagship skill. The output is the raw material `write-tweet` chews on, and the long-form goes straight to Telegram via `./notify`.

## Sandbox note

The sandbox may block outbound curl. For every curl call below, if it fails or returns empty, use **WebFetch** for the same URL — all Polymarket APIs are public, no auth needed.

## Steps

### 1. Pick three markets

If `${var}` is set, treat it as a single event slug and skip to step 2 with just that one.

Otherwise, fetch the top markets by 24h volume and pick three:

```bash
curl -s "https://gamma-api.polymarket.com/markets?closed=false&active=true&archived=false&enableOrderBook=true&order=volume24hr&ascending=false&limit=25"
```

From the 25, select three that:
- Have **liquidity > $5,000** and **volume24hr > $1,000**
- At least one outcome price is in `[0.05, 0.95]` (don't write a thesis on a market that's already a coin flip's worth of edge from resolution)
- End date is between **6 hours from now** and **45 days out** — short enough to matter, long enough that I'm not just reading the resolution
- Cover **different categories** if possible (don't ship three NBA markets) — diversify across politics / sports / crypto / culture
- Aren't already in the last 2 days of `memory/logs/` for this skill

Each market has: `id`, `question`, `slug`, `description`, `resolutionSource`, `outcomes` (JSON), `outcomePrices` (JSON — index 0 = YES), `volume24hr`, `volumeNum`, `liquidityNum`, `endDate`, `clobTokenIds` (JSON).

### 2. For each market, fetch sentiment context

For each pick, get the event ID and the top community comments — this tells you what the crowd is thinking before you form an opinion.

```bash
# Find parent event from market slug
# (event slug is typically the market slug minus trailing date/variant)
curl -s "https://gamma-api.polymarket.com/events?slug=$EVENT_SLUG&limit=1"

# Then top comments by reactions
curl -s "https://gamma-api.polymarket.com/comments?parent_entity_type=Event&parent_entity_id=$EVENT_ID&limit=15&order=reactionCount&ascending=false"
```

`parent_entity_type` must be capitalized `Event`. Skip the comments fetch silently if it fails — sentiment is nice-to-have, not required.

### 3. Form the thesis

For each market, write **a few sentences (60–150 words is the right zone)** that cover these beats — but mix the order across the three so they don't all read the same:

1. **What the market is actually asking** — one sentence, plain language. If the question is jargon-heavy ("Fed cuts 50bp at March meeting"), translate it.
2. **Where it's priced right now** — exact YES price, exact 24h volume.
3. **What the crowd thinks** — what the price implies people believe. If the top comments are saying something useful, point at it ("YES camp's all leaning on X; NO side's mostly Y").
4. **Where I disagree (or agree but think the price is off)** — the actual read. Base rate from comparable events. One inside-view detail. One thing that would change my mind.
5. **My side and conviction** — "long at 0.42, conviction 7/10." If there's genuinely no edge, say so: "fair price, no read."

### Voice

Talk like a person, not a market analyst. Short sentences. Contractions. Specific over abstract — name the market, quote the price, point at the timeframe. Crypto-native vocabulary is fine in moderation (fade, long, short, conviction, "the crowd's wrong about X") — but **no hashtags, no emojis, no 'not financial advice' boilerplate, no forced lowercase-twitter cosplay**. If a take's hot, write it hot. If it's lukewarm, say so. Faking conviction is the worst failure mode here.

The voice is "smart friend explaining the trade over coffee" — not Bloomberg op-ed, not anon shitposter. A real opinion, said plainly, backed by one or two specific things you noticed.

Avoid:
- "Might", "could potentially", "it's worth considering" — pick a side or say there's no edge
- Generic Polymarket framing ("prediction markets are interesting because...")
- Three-letter-acronym soup without translation
- Repeating yesterday's thesis with new numbers — if today's take's the same as yesterday's, just note "no change from yesterday" and pick a different market

### 4. Build the report

```
# polymarket thesis — ${today}

## 1. [Market question]

YES X.XX • 24h vol $X.Xm • resolves [relative date]

[one-paragraph thesis as drafted in step 3]

→ ZER0's side: [LONG/SHORT YES/NO] @ [price] — conviction [X/10]

## 2. [Market question]

... (same shape)

## 3. [Market question]

... (same shape)

---
Three markets covered. Watchlist for tomorrow: [one line on what's brewing that didn't make today's cut]
```

### 5. Notify

Send the full report via `./notify` (under 4000 chars — trim the longest paragraph if it overflows, never the conviction lines):

```
polymarket thesis — ${today}

1. [market question]
YES X.XX • $X.Xm 24h • resolves [date]
[2–3 sentences from the paragraph, lead with the take]
→ [LONG/SHORT] @ [price] — [X/10]

2. [market question]
... (same shape)

3. [market question]
... (same shape)

watching tomorrow: [one line]
```

### 6. Log

Append to `memory/logs/${today}.md`:

```
## Polymarket Thesis
- **Markets covered:** 3
  - "[market 1]" — [LONG/SHORT] @ [price], conviction [X/10]
  - "[market 2]" — [LONG/SHORT] @ [price], conviction [X/10]
  - "[market 3]" — [LONG/SHORT] @ [price], conviction [X/10]
- **Top conviction:** "[market]" — [X/10]
- **Watchlist for tomorrow:** [one line]
- **Notification sent:** yes
```

If today's top conviction is **8/10 or higher**, also note the market and side in `memory/MEMORY.md` under a `## Open positions` section so `prediction-journal` can settle it later.
