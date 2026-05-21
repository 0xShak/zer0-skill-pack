---
name: Polymarket Alpha Comments
description: Surface high-signal commentary from Polymarket markets — whale callouts, insider-sounding takes, sharp contrarians
var: ""
tags: [crypto, prediction-markets]
---
> **${var}** — Optional topic filter (e.g. "elections", "crypto"). Empty = scan top trending markets across all categories.

Read `memory/MEMORY.md` for context on recent alpha pulls.
Read the last 2 days of `memory/logs/` so we don't re-surface the same comments.

## What this skill is for

`polymarket-comments` (the built-in Aeon skill) is a feed scrape — it surfaces interesting comments from trending markets. This skill is sharper: it hunts for **alpha**.

Alpha-grade comments are:
- **Whale callouts** — "guy just dropped $500k YES at 0.42, biggest order of the day"
- **Insider-sounding takes** — "source close to the campaign tells me X" / "I was at the meeting today" — flagged as unverified but worth surfacing
- **Sharp contrarian reasoning** — someone explaining why the heavy favorite is wrong with specific evidence
- **Resolution-mechanic alpha** — "this market resolves on UMA Y/N, and the proposed outcome reads X — vote ends in 4h"
- **Cross-market arbitrage tips** — "Kalshi has the same question at 0.62, Polymarket at 0.48"

What does NOT count:
- General opinions ("I think YES is too cheap")
- Memes, jokes, one-liners
- Whale callouts without a price/size detail
- "I have a source" with no specific direction

Quality bar: each surfaced comment needs **one specific actionable thing** — a price, a size, a vote deadline, a cross-venue spread, a name, a source claim with detail. If you wouldn't change your read on the market based on the comment, don't ship it.

## Sandbox note

The sandbox may block outbound curl. Use **WebFetch** as a fallback. All Polymarket APIs are public.

## Steps

### 1. Pull trending markets

If `${var}` is set, use it as a topic filter (keyword match against `question` and `category` fields).

```bash
# Top markets by 24h volume
curl -s "https://gamma-api.polymarket.com/markets?closed=false&active=true&order=volume24hr&ascending=false&limit=25"
```

From the 25, pick **8 markets** that:
- `volume24hr > $5,000` (need real activity to have real commentary)
- At least one outcome price in `[0.10, 0.90]` (markets close to coin flip or close to resolution attract less interesting comments)
- If `${var}` filter is set, only markets matching the topic
- Skip pure sports — game-time markets attract emotional comments, not alpha

### 2. Pull comments for each — both top and latest

For each of the 8 markets, get the event ID first:

```bash
curl -s "https://gamma-api.polymarket.com/events?slug=$EVENT_SLUG&limit=1"
```

Then both top-by-reactions and most-recent:

```bash
# Top by reactions (settled wisdom of the crowd)
curl -s "https://gamma-api.polymarket.com/comments?parent_entity_type=Event&parent_entity_id=$EVENT_ID&limit=30&order=reactionCount&ascending=false"

# Most recent (fresh whale moves, real-time alpha)
curl -s "https://gamma-api.polymarket.com/comments?parent_entity_type=Event&parent_entity_id=$EVENT_ID&limit=20&order=createdAt&ascending=false"
```

Combine and dedupe. `parent_entity_type` must be capitalized `Event`.

Each comment has: `body`, `profile.username` (often null → "anon"), `reactionCount`, `createdAt`.

### 3. Classify each comment

For each combined comment, classify into one of:

- **whale** — names a price + size + side ("just dropped $250k NO at 0.18")
- **insider** — claims privileged knowledge with specific detail ("I work at X, the announcement is Y day")
- **contrarian** — well-reasoned case against the consensus with evidence
- **resolution** — alpha on how the market actually resolves (UMA vote, oracle dispute, etc)
- **arb** — cross-venue price spread observation
- **noise** — none of the above (this is the bulk of comments — skip)

### 4. Score and rank

Keep only `whale`, `insider`, `contrarian`, `resolution`, `arb`. Score each:

- **Specificity** (0–3): how concrete? "$500k YES at 0.42" = 3; "big buy" = 0
- **Plausibility** (0–3): does it pass a smell test? Insider claims with named events are 2–3; "trust me" claims with no detail are 0–1
- **Actionability** (0–3): would this change a real trader's behavior?
- **Reactioncount bonus** (+1 if `reactionCount >= 10`)

Total score per comment. Pick the **top 5 across all 8 markets** — not 5 per market. Could be 3 from one market and 0 from others, that's fine.

### 5. Mark unverified claims explicitly

For any `insider` comment surfaced, prepend the entry with `[unverified]`. We're a curator, not a primary source. The reader needs to know that "source at campaign" is a claim, not a fact.

If a comment is clearly a self-pump ("I bought a million YES, you should too"), demote it heavily — that's promotion, not alpha.

### Voice

Voice on this skill is "alpha curator". Less opinionated than the thesis skills — let the comments do the talking. Frame each entry as "here's what someone said and why it matters."

A good surfaced comment:

> [whale] anon on "Will the Senate confirm X by June": "just dropped $180k NO at 0.31, that's the third six-figure NO bid this week, all from the same wallet pattern." 42 reactions. Worth watching — if the pattern's real, it's a sustained accumulation play, not a one-off bet.

A bad one (skip these):

> anon: "YES is too cheap, free money." (no specific anything → noise)

**No hashtags, no emojis, no 'not financial advice'.** Just the curation + why it matters.

### 6. Build the report

```
# polymarket alpha — ${today}

[N] alpha calls surfaced across [M] markets.

## 1. [classification tag] on "[Market question]" (YES X.XX, $X.Xm vol)
**@user (or anon)**: "[comment body, trimmed if >300 chars]" ([X] reactions, [Y]h ago)
**Why it matters:** [one sentence — what this implies for the read on the market]

## 2. [classification tag] on ...

(top 5 alpha calls total)
```

### 7. Notify

Send via `./notify` (under 4000 chars):

```
polymarket alpha — ${today}

[N] alpha calls.

1. [tag] "[market]" — YES X.XX
@user: "[comment, trimmed]" ([X] reacts)
→ [why it matters, one line]

2. ... (same shape)
```

If 0 alpha-grade comments today: `polymarket alpha — ${today}. Nothing alpha-grade in the comments today. Back later.`

### 8. Log

Append to `memory/logs/${today}.md`:

```
## Polymarket Alpha Comments
- **Markets scanned:** 8
- **Alpha calls surfaced:** [N]
  - [tag] on "[market]" — @user, score [X]
- **Notable [whale/insider/contrarian/resolution/arb]:** "[short excerpt]"
- **Notification sent:** yes
```
