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

**Market freshness check (mandatory):** for every candidate, verify BOTH `acceptingOrders == true` AND `endDate > now + 6 hours` BEFORE forming a thesis. Polymarket markets stay `closed=false` for hours after the underlying event happens (until UMA resolves), so the API filter alone isn't enough. If you're looking at a sports market, also cross-check that the game's tip-off / start time is in the future — sometimes `endDate` is set hours after tip-off and you'll catch a game that's already been played. If a market fails either check, skip it and pick another from the next 25.

From the 25, select three that:
- Have **liquidity > $5,000** and **volume24hr > $1,000**
- At least one outcome price is in `[0.05, 0.95]` (don't write a thesis on a market that's already a coin flip's worth of edge from resolution)
- End date is between **6 hours from now** and **45 days out** — short enough to matter, long enough that I'm not just reading the resolution
- Cover **different categories** if possible (don't ship three NBA markets) — diversify across politics / crypto / geopolitics / culture
- Aren't already in the last 2 days of `memory/logs/` for this skill
- **Deterministic resolution** — the market's `resolutionSource` is named and external (AP race call, FOMC press release, Coinbase BTC close, official vote tally, UMA oracle with a hard question). The resolution criterion must be testable with a clear yes/no answer that doesn't depend on subjective interpretation.

**Skip these market types:**
- Subjective wording in the resolution criterion: "permanent", "successful", "officially recognized as X", "significant", "meaningful" — if a human judge has to decide what counts, you're not pricing a market, you're pricing a coin-flip on the judge's mood
- Resolution authority not named in `resolutionSource`
- Multiple plausible readings of the resolution wording exist
- Routine daily sports/esports fixtures (NBA regular-season game, single LCK match) — these are sportsbook territory, not prediction-market edge cases. Exception: championships, finals, or specific in-game props with named resolution data

**Good market examples:** "Will X win Pennsylvania in the 2026 election" (AP call), "Will BTC close above $100k on Dec 31" (Coinbase close), "Will Senate confirm X by date Y" (official vote count). **Bad examples:** "Will the US-Iran peace deal be permanent" (permanent is undefined), "Will SpaceX have a successful launch" (success is undefined), "Will the war end in 2026" (end is undefined).

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

For each market, write **4-6 short bullets, each a single sentence, each prefixed with `>` as the bullet character**. No prose paragraphs. The `>` bullets are the signature format — they match `soul/STYLE.md` and read as tight, scannable lines.

Bullets to include (mix the order across the three markets so they don't all read identically):

1. **The market line** — price + 24h vol + resolution date in one bullet. Example: `> YES 0.195, $4.8m 24h, resolves in 9 days`
2. **The take** — lead with the claim in one sentence. Example: `> permanent is doing all the work in the resolution language`
3. **Base rate or comparable event** — one bullet of "here's how similar markets resolved." Example: `> last 5 trump-era diplomatic announcements averaged 70+ days to signed text`
4. **One inside-view detail** — something specific you noticed in the comments, news, or order book. Example: `> nyt flagged a 60-day negotiation memo, signed deal lands well past 5/31`
5. **Side + conviction** — explicit, no hedging. Example: `> side: SHORT YES at 0.195, 7/10`
6. (Optional) **What would change your mind** — only if the trade is conviction <= 6/10 and a specific signal would flip it. Example: `> would flip LONG if a draft text leaks before 5/29`

If there's genuinely no edge, replace bullets 2-5 with a single bullet: `> fair price, no read`. Don't fake conviction to fill bullets.

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
# polymarket thesis. ${today}

## 1. [Market question]

> YES X.XX, $X.Xm 24h vol, resolves [relative date]
> [lead with the take in one sentence]
> [base rate or comparable event]
> [the inside-view detail]
> side: [LONG/SHORT] @ [price], conviction [X/10]

## 2. [Market question]

... (same shape)

## 3. [Market question]

... (same shape)

watching tomorrow: [one line on what's brewing that didn't make today's cut]
```

### 5. Notify

Send the full report via `./notify` (under 4000 chars — trim the optional "would flip" bullets first if it overflows, never the conviction lines).

**FORMAT IS NON-NEGOTIABLE.** Every reasoning line starts with the literal character `>` followed by a single space. No prose paragraphs. No exceptions. If you're tempted to write a sentence without `>` in front, stop and rewrite it as a bullet.

Concrete example of correct output (mock data, follow this shape exactly):

```
polymarket thesis. 2026-05-22

1. will trump win pennsylvania in 2026 midterms?
> YES 0.42, $2.1m 24h, resolves nov 4
> ap call is the trigger, pa is the cleanest sample
> dem incumbents averaged 51-53% margins in this district last 4 cycles
> trump's been polling 4-7pp underwater statewide all year
> side: SHORT YES at 0.42, 7/10

2. will btc close above $120k on dec 31?
> YES 0.31, $890k 24h, resolves dec 31
> coinbase close is the trigger, no oracle ambiguity
> historical year-end rallies after similar mid-year setups: 4 of last 7
> current spot $98k means needs +22% in 7 months, plausible but not priced for it
> side: LONG YES at 0.31, 6/10

3. will senate confirm jane doe by june 15?
> YES 0.72, $410k 24h, resolves jun 15
> official vote tally is the trigger
> fair price, no read

watching tomorrow: [one line on what's brewing]
```

Use this EXACT structure: market number + question, then `>` bullets, then blank line before next market.

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
