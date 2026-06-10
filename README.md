# aso-cli

**App Store keyword research from your terminal. No accounts, no API keys, no paid services.**

`aso` tells you which App Store keywords are worth targeting: how much search
demand a keyword has (**popularity**), how hard it is to rank for
(**difficulty**), and whether the trade-off is worth it (**opportunity**).
It works entirely from public Apple endpoints, outputs JSON, and is built to
be driven by AI agents as well as humans.

```console
$ aso research "journal"
```
```json
{
  "seed": "journal",
  "underserved": [
    { "keyword": "journal prompts", "popularity": 40, "difficulty": 18,
      "opportunity": 42, "classification": "Sweet Spot", "popularity_confidence": "high" }
  ],
  "all": [ "...every scored candidate, sorted by opportunity..." ]
}
```

One file, Python 3.10+ standard library only. Nothing to `pip install`.

## Installation

```bash
git clone https://github.com/YOURNAME/aso-cli.git
cd aso-cli
chmod +x aso
ln -s "$PWD/aso" /usr/local/bin/aso    # or anywhere on your PATH
aso --version
```

That's the whole setup. To uninstall, delete the symlink and the folder, plus
the response cache at `~/.cache/aso-cli/` if you want a full cleanup.

## Commands

### `aso research "<seed>"` — find keywords worth targeting

The main workflow. Expands a seed keyword through Apple's own autosuggest
(the suggestions the App Store search box shows, which only contains terms
real users type), scores every candidate, and returns them sorted by
opportunity. Keywords classified Sweet Spot, Hidden Gem, or Good Target are
collected in the `underserved` array — that's your shortlist.

```bash
aso research "recipe" --limit 30
aso research "pantry" --deep          # adds a-z prefix expansion + second-hop suggestions
aso --country de research "rezept"    # any storefront, local language works
```

### `aso score "<keyword>" [...]` — score specific keywords

Full analysis of keywords you already have in mind. Multiple keywords come
back sorted by opportunity. Add `--apps` to see the top 10 competing apps
with their rating counts — useful for judging a field with your own eyes.

```bash
aso score "habit tracker" "mood diary" "sleep journal"
aso score "recipe organizer" --apps
```

### `aso rank <app_id> "<keyword>" [...]` — measure where an app actually ranks

Checks an app's position in the real App Store search results (scraped from
the App Store web page, which mirrors device ranking — the iTunes API's
ordering does not). Run it before a metadata change to baseline, and again
after to measure movement. The numeric app id is in any App Store URL
(`apps.apple.com/us/app/.../id1044867788`) or in `aso search` output.

```bash
aso rank 1044867788 "journal" "gratitude journal" "diary"
```

### `aso suggest "<term>"` — Apple's autosuggest, raw

What the App Store suggests while typing this term, in Apple's own
popularity order. Doubles as a demand check: if a phrase never appears in
autosuggest, real users aren't typing it.

```bash
aso suggest "meal pl"
```

### `aso search "<term>"` — raw competitor data

The apps competing for a term with ratings, stars, release dates, and
sellers. This is the raw input every score is computed from.

```bash
aso search "pantry tracker" --limit 25
```

## How to interpret the numbers

| Field | Range | Meaning |
|---|---|---|
| `popularity` | 5–100 | Estimated search demand. **Relative ordering is reliable; absolute values are estimates.** Nobody outside Apple has real per-keyword search counts — every ASO vendor ships estimates. |
| `difficulty` | 1–100 | How hard it is to rank top-10 **if you target the exact phrase** in your title/subtitle. Under 30 is easy, 30–50 doable for a small app, 50–65 needs real traction, over 65 is dominated by established players. |
| `opportunity` | 0–100 | Popularity gated by difficulty. **Sort by this.** High popularity is worthless behind difficulty 80; this number encodes the trade-off. |
| `classification` | label | `Sweet Spot` (high demand, low competition — the target), `Hidden Gem` (modest demand, minimal competition), `Good Target`, `Moderate`, `High Competition`, `Low Volume`, `Avoid`. |
| `est_daily_searches` | count | Popularity mapped to a rough US-storefront daily-search figure. Treat as an index, not a measurement — and only compare within one storefront. |
| `popularity_confidence` | high/medium/low | Cross-check against Apple autosuggest. `high` = the exact term appears in autosuggest, so real users type it. `low` = popularity may be inflated by Apple backfilling results with big apps from broader terms — trust the difficulty number, distrust the popularity number. |
| `autosuggest_rank` | 1–n or null | Position of the exact term in Apple's suggestions when typed in full. |

`difficulty_breakdown` exposes the seven sub-scores (rating volume, review
velocity, dominant players, rating quality, market age, publisher diversity,
title relevance) plus `is_brand_keyword` (true means the #1 app's publisher
owns this term — "spotify" is not an opportunity no matter what the numbers
say) and `override` (set when a correction fired, see below).

### A worked example

```
pantry tracker   pop 76  diff 40  opp 57  Sweet Spot   conf: high
recipe           pop 96  diff 78  opp 37  High Competition
recipe import    pop 50  diff 45  opp 40  Moderate
```

"pantry tracker" has strong demand and the current top-10 includes apps with
double-digit rating counts: target it. "recipe" has enormous demand but the
field is owned by apps with hundreds of thousands of ratings: skip it.
"recipe import" is real (autosuggest confirms users type it) but modest:
worth a slot in your keyword field, not your title.

## How it works

There are two ways ASO tools get these numbers:

1. **Apple Search Ads.** Commercial tools (Astro, Appfigures, Sensor Tower
   and friends) read the per-keyword "Search Popularity" integer (5–100) from
   the Apple Search Ads backend, which requires an ASA account. That metric
   is a relative ad-bidding signal, not search volume, and it has failed
   structurally: in October 2025 Apple silently collapsed 77% of US keywords
   to the floor value of 5. If a tool shows you popularity 5 for a keyword
   that autosuggest demonstrably completes, you're looking at the artifact.

2. **Derived scores from public data.** Estimate demand and competition from
   what public search results reveal: who ranks for a term, how many ratings
   they have, how many titles target it, how old and diverse the field is.

`aso` implements approach 2, following the scoring methodology published by
the open-source [RespectASO](https://github.com/respectlytics/respectaso)
project (this is an independent reimplementation, with thanks).

**Data sources (all public, no auth):**

| Endpoint | Used for |
|---|---|
| `itunes.apple.com/search` | competitor data per keyword |
| App Store autosuggest (`MZSearchHints` + storefront header) | keyword expansion + the confidence cross-check |
| `apps.apple.com/{cc}/iphone/search` embedded JSON | true ranking order for `rank` |

**Popularity (5–100)** is built from six signals: result count, leader
strength (log-scaled ratings of the strongest top-half app), title-match
density, market depth (median ratings), a word-count specificity penalty,
and an exact-phrase bonus, with dampening for small samples and for
backfilled results.

**Difficulty (1–100)** is seven weighted factors: median rating volume (30%),
dominant players (20%), and 10% each for review velocity, rating quality,
market age, publisher diversity, and title relevance. Post-processing then
corrects for Apple's habit of padding thin results with big apps from broader
terms: a small-result-set cap, a weak-leader cap (if the #1 app has under
1,000 ratings the keyword is objectively winnable), a backfill discount, and
brand-keyword detection.

**Opportunity (0–100)** is log-normalized estimated demand multiplied by a
quadratic difficulty gate, so difficulty hurts a little at 30 and brutally at
80.

## Honest limitations

- **Estimates, not measurements.** Use the scores to order keywords and make
  targeting decisions, not to forecast downloads.
- **Compound keywords with a giant head term** ("recipe X", "photo X") can
  read 10–15 popularity points hot, because the big apps Apple backfills
  into results partially count as demand evidence. The
  `popularity_confidence` field exists to catch this — check it.
- **Difficulty assumes you target the exact phrase.** It answers "can I get
  indexed and into the top 10", not "can I take the #1 spot from an app with
  238k ratings". Tools that report occupant strength (how strong the apps
  currently in the results are) will read higher than `aso` on backfilled
  keywords; both views are valid, they answer different questions.
- **Cross-country comparisons:** popularity is calibrated within a
  storefront. Compare keywords inside one country, not a keyword across
  countries.

## For AI agents

Everything is JSON on stdout; progress and errors go to stderr. Exit code 0
on success, 1 on network/rate-limit failure.

An [Agent Skill](https://docs.claude.com/en/docs/agents/skills) ships in
[`skill/`](skill/SKILL.md). To install it for Claude Code:

```bash
mkdir -p ~/.claude/skills
cp -r skill ~/.claude/skills/aso-keyword-research
```

Then ask your agent things like "find underserved App Store keywords around
meditation" and it will drive the CLI itself.

## Rate limits and caching

Apple's search API allows roughly 20 requests/minute per IP, so uncached
keyword scores are spaced ~3 seconds apart: a 30-keyword research run takes
about 90 seconds cold. Every response is cached for 24 hours in
`~/.cache/aso-cli/` (override with `ASO_CACHE_DIR` / `ASO_CACHE_TTL`), so
re-runs and overlapping research are instant. `--fresh` bypasses the cache
when you need today's data.

## License

[AGPL-3.0](LICENSE). The scoring methodology follows RespectASO (AGPL-3.0);
this project keeps the same license.
