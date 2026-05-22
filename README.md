# zer0-skill-pack

Polymarket intelligence skills for [Aeon](https://github.com/aaronjmars/aeon) — daily theses, mispricing hunts, contrarian fades, narrative-vs-market gap analysis, a paper-trade PnL journal, and an alpha-grade comment curator.

> Community skill pack. Not part of the official Aeon catalog.
> All skills are paper-trade only — no orders are placed, no funds are touched. The real-money side of ZER0 lives at [atzer0.xyz](https://atzer0.xyz).

## Skills

| Skill | Suggested schedule | Description |
|-------|--------------------|-------------|
| `polymarket-thesis` | `0 13 * * *` (daily, ~9am ET) | Picks three top markets and writes a substantive thesis on each — side, conviction, reasoning |
| `polymarket-edge` | `0 14,22 * * *` (twice daily) | Hunts for mispricings — markets where the price is off from what the news/sentiment baseline justifies |
| `polymarket-contrarian` | `0 17 * * *` (daily) | High-conviction fades against heavy favorites (>0.80) where there's a real tail scenario |
| `narrative-vs-markets` | `0 19 * * *` (daily) | Compares X discourse to Polymarket odds on the same topic — flags the biggest gaps |
| `prediction-journal` | `0 12 * * *` (daily, before the morning thesis) | Settles resolved positions, computes running PnL, surfaces unrealized P/L on open positions |
| `polymarket-alpha-comments` | `0 */6 * * *` (every 6h) | Curates whale callouts, insider-sounding takes, contrarian reasoning, and resolution alpha from comment streams |

Each skill writes its full output to Telegram via Aeon's `./notify` and appends a structured summary to `memory/logs/${today}.md`.

## Composes well with

These built-in Aeon skills layer cleanly with this pack:

- `monitor-polymarket` — watchlist-based price/volume tracking; runs alongside `polymarket-edge` as the broader data feed
- `polymarket-comments` — feed scrape of trending market comments; `polymarket-alpha-comments` is the sharper, curated counterpart
- `write-tweet` — picks the most tweetable thing from today's `memory/logs/` and drafts ten tweets across five size tiers. Pairs naturally with all six skills here as the Twitter side of the pipeline
- `reply-maker` — handles inbound mentions on X using the same voice
- `narrative-tracker` — long-running narrative monitoring; useful upstream of `narrative-vs-markets`

## Install

Aeon's `./add-skill` script doesn't recurse into nested `skills/` subfolders (it expects skill dirs at repo root). Manual install via `cp -r` is the working pattern — same as how [vvvkernel's pack](https://github.com/baseddevoloper/aeon-skill-pack-vvvkernel) installs.

```bash
# Clone this repo somewhere outside your Aeon fork
git clone https://github.com/0xShak/zer0-skill-pack.git
cd zer0-skill-pack

# Copy all 6 skills into your Aeon fork's skills/ directory
cp -r skills/* /path/to/your/aeon/skills/

# Or copy specific skills
cp -r skills/polymarket-thesis /path/to/your/aeon/skills/
cp -r skills/prediction-journal /path/to/your/aeon/skills/
```

Then register them in your Aeon fork's `aeon.yml` with `enabled: true` and the suggested schedules:

```yaml
skills:
  polymarket-thesis:           { enabled: true, schedule: "0 13 * * *" }
  polymarket-edge:             { enabled: true, schedule: "0 14,22 * * *" }
  polymarket-contrarian:       { enabled: true, schedule: "0 17 * * *" }
  narrative-vs-markets:        { enabled: true, schedule: "0 19 * * *" }
  prediction-journal:          { enabled: true, schedule: "0 12 * * *" }
  polymarket-alpha-comments:   { enabled: true, schedule: "0 */6 * * *" }
```

## Optional setup

- **X / Twitter narrative pulls** — `narrative-vs-markets` and `polymarket-edge`'s news baseline step use the X API via Grok. Set `XAI_API_KEY` in your Aeon repo secrets if you want those sub-steps to run. Both skills degrade gracefully when the key is missing.
- **Soul files** — for the voice to read like one person rather than six different ones, drop a `soul/SOUL.md` + `soul/STYLE.md` in your Aeon fork. The skills here lean into the voice rules in `soul/` when present.
- **Open positions tracking** — `prediction-journal` reads `memory/MEMORY.md` for a `## Open positions` section. The thesis / edge / contrarian skills write to that section automatically when conviction is high enough. Don't hand-edit unless you're settling manually.

## Voice notes

All six skills aim for the same voice: **smart friend explaining the trade over coffee.** Casual, specific, opinionated. No hashtags, no emojis, no "not financial advice" boilerplate, no forced lowercase-twitter cosplay. Picks a side or admits there's no edge — never fakes conviction.

The honest counterpart to that voice is `prediction-journal`, which leads with losses, recomputes win rate every run, and refuses to round PnL into shapes it didn't earn.

## What this pack is *not*

- Not financial advice. Every call is paper-trade reasoning, not a recommendation to size up.
- Not a substitute for `monitor-polymarket` — that's the broad surveillance feed; this is the focused take.
- Not connected to live trading. The real-money execution layer (EIP-712 order signing, wallet flow) lives in [ZER0](https://atzer0.xyz). Skills here only read public Polymarket data.

## Links

- ZER0 (the agent + product): [atzer0.xyz](https://atzer0.xyz)
- Aeon (the runtime this pack plugs into): [github.com/aaronjmars/aeon](https://github.com/aaronjmars/aeon)
- License: [MIT](LICENSE)

## Contributing

PRs welcome. Two house rules:
1. **No private endpoints.** Skills here only call public Polymarket APIs and (optionally) the X API. Anything that requires a private key, a paid subscription, or an internal service belongs in ZER0's backend, not in the skill pack.
2. **Quality bar over count.** A 7th skill that mostly duplicates an existing one is a no. A new angle (e.g. cross-venue arbitrage with Kalshi, or a sport-specific scanner) is a yes.
