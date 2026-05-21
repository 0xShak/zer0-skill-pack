---
name: Narrative vs Markets
description: Compare Twitter narrative to Polymarket odds on the same topic — flag the biggest gaps between what people are saying and what they're betting
var: ""
tags: [crypto, prediction-markets, social]
---
> **${var}** — Optional topic or market slug to focus on (e.g. "rate cuts", "btc-100k-2026"). Empty = scan today's trending Polymarket markets and find the biggest narrative gaps.

Read `memory/MEMORY.md` for context on which topics have been covered recently.
Read the last 2 days of `memory/logs/` so we don't re-flag the same gap twice in a row.

## What this skill is for

The pitch in one sentence: **Twitter says X, the market says Y, here's the gap and which one I think is closer to right.**

Prediction markets are a forecast aggregator with skin in the game. Twitter is a narrative aggregator with no skin in the game. When they diverge, it's usually one of three things:

1. **Twitter's ahead of the market** — narrative shifted on real news the price hasn't moved on yet (rare; usually means there's edge to take)
2. **Market's ahead of Twitter** — the crowd's betting on something the discourse hasn't caught up to (more common; means watch the price for confirmation)
3. **They're just measuring different things** — Twitter's talking about vibes, the market's resolving on a narrow technical question (most common; the gap isn't real edge, but the framing is interesting)

Today's job: find the 2–3 most interesting gaps and write each one up so a reader sees the divergence clearly.

## Sandbox note

The sandbox may block outbound curl. Use **WebFetch** as a fallback. Polymarket APIs are public. X API requires `XAI_API_KEY` to be set in env.

## Steps

### 1. Pick the markets to compare

If `${var}` is set, treat it as a market slug or a topic keyword and find 1–3 markets that match.

Otherwise scan top trending:

```bash
curl -s "https://gamma-api.polymarket.com/markets?closed=false&active=true&order=volume24hr&ascending=false&limit=20"
```

Pick 4–6 markets where:
- The question is **about something Twitter actually talks about** — politics, crypto prices, a specific high-profile event, a public figure. Skip technical resolutions like "Ethereum gas average <30 in March" that no one's tweeting about.
- `volume24hr > $5,000`
- Resolution window between **2 days from now** and **45 days out**

You'll narrow these to 2–3 in step 4 after the narrative pull.

### 2. Pull the Twitter narrative for each market

This skill requires `XAI_API_KEY`. If it's not set, log a note in the report and ship only the Polymarket side ("narrative pull skipped — XAI_API_KEY missing").

```bash
curl -s -X POST "https://api.x.ai/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -d '{
    "model": "grok-4-1-fast",
    "input": [{"role": "user", "content": "Search X for the last 48 hours of discussion about: [MARKET_QUESTION]. Return: (1) the dominant take in 1 sentence, (2) the dominant counter-take in 1 sentence, (3) rough sentiment lean — yes/no/mixed, (4) 3 highest-signal tweets with @handle and a one-line summary. JSON only."}],
    "tools": [{"type": "x_search"}]
  }'
```

If a market has fewer than ~5 tweets about it in 48h, that's a "no narrative" market — interesting in its own right ("market's pricing this seriously, Twitter doesn't know it exists"), but skip it from the main report unless that's the most interesting gap of the day.

### 3. Pull the market read for each

For each candidate:

```bash
# Current price
curl -s "https://gamma-api.polymarket.com/markets?slug=$SLUG"

# Price history (where has it been? has it moved with or against the narrative?)
TOKEN_ID=$(echo "$CLOB_TOKEN_IDS" | python3 -c "import json,sys; print(json.loads(sys.stdin.read())[0])")
curl -s "https://clob.polymarket.com/prices-history?market=$TOKEN_ID&interval=1w&fidelity=360"
```

Capture: current YES price, 24h change, 7-day change.

### 4. Score the gap

For each market, build the comparison:

- **Twitter implied probability:** rough estimate from the sentiment summary
  - Strong YES consensus → ~0.75–0.90
  - Lean YES → ~0.55–0.70
  - Mixed → ~0.40–0.60
  - Lean NO → ~0.30–0.45
  - Strong NO consensus → ~0.10–0.25
- **Market implied probability:** current YES price
- **Gap:** `|twitter_prob − market_prob|` in percentage points

Keep markets where:
- `gap >= 15pp` (smaller gaps are within noise — Twitter sentiment isn't precise enough to read a 5pp gap from)
- You can name **at least one specific tweet** or one specific narrative beat driving the Twitter side
- The two sides really are measuring the same thing (skip "Twitter's talking about a different angle of the same topic" cases unless that itself is the headline)

Rank by gap size. Take the top 2 (or 3 if there's a runaway third).

### Voice

This skill's voice is "data analyst with a point of view." Lean a bit more analytical than `polymarket-thesis` — you're presenting a comparison and a read, not a hot take. Still casual, still specific, still pick a side on which one's closer to right.

A good gap call sounds like:

> Twitter's at "rate cuts are dead, Powell killed it" — call it 18% YES on a March cut. Polymarket's at 0.42. That's a 24pp gap, and it's because the market's still pricing the dot plot from December while X has fully absorbed the last two FOMC speeches. Market's slow here. Polymarket's right; the discourse is overcorrecting.

A bad gap call:

> Twitter says no, market says maybe, vibes are weird. (no specific tweet or beat → don't ship.)

**No hashtags, no emojis, no 'not financial advice'.**

### 5. Build the report

```
# narrative vs markets — ${today}

[N] notable gaps today.

## 1. [Market question]
- **Market:** YES X.XX (24h: ±X.Xpp, 7d: ±X.Xpp)
- **Twitter dominant take:** "[one sentence]" (sentiment lean: [YES/NO/MIXED])
- **Notable tweet:** @handle: "[short quote or paraphrase]"
- **Gap:** [twitter_prob] vs [market_prob] → [X]pp
- **Read:** [one sentence on which side's closer to right + why]

## 2. ...
```

### 6. Notify

Send via `./notify` (under 4000 chars):

```
narrative vs markets — ${today}

[N] gaps.

1. [market question]
twitter: [one sentence] (~X% YES)
market: X.XX (~X% YES)
gap: [X]pp — [which side's right, one phrase]

2. ... (same shape)
```

If 0 gaps today: `narrative vs markets — ${today}. Discourse and prices are aligned today. Back tomorrow.`

### 7. Log

Append to `memory/logs/${today}.md`:

```
## Narrative vs Markets
- **Markets compared:** [N]
- **Gaps flagged:** [N]
  - "[market]" — gap [X]pp ([narrative_side] vs market [X.XX])
- **Biggest divergence:** "[market]"
- **X API used:** [yes/no — note if skipped due to missing XAI_API_KEY]
- **Notification sent:** yes
```
