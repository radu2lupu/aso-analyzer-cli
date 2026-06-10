---
name: aso-keyword-research
description: >
  App Store keyword research with the aso CLI: find underserved keywords
  (high search popularity, low competition), score and compare keywords,
  check an app's real search rank, and recommend title/subtitle/keyword-field
  metadata. Use when the user asks about ASO, App Store keywords, keyword
  popularity or difficulty, app search rankings, or how to position an iOS
  app's metadata.
---

# App Store keyword research with `aso`

The `aso` CLI scores App Store keywords from public Apple endpoints. All
commands print one JSON object to stdout; progress goes to stderr. If `aso`
is not on PATH, look for it in the aso-cli repository checkout.

## Commands

| Command | What it does | Cost |
|---|---|---|
| `aso research "<seed>" [--limit N] [--deep]` | Expand seed via Apple autosuggest, score every candidate, sort by opportunity. `underserved` array = the shortlist. | ~3s per keyword, uncached |
| `aso score "<kw>" [...] [--apps]` | Score specific keywords. `--apps` includes top-10 competitors. | ~3s per keyword |
| `aso rank <app_id> "<kw>" [...]` | App's position in real App Store search results. | ~2s per keyword |
| `aso suggest "<term>"` | Raw Apple autosuggest (popularity-ordered, only real user search terms). | ~1s |
| `aso search "<term>" [--limit N]` | Raw competitor apps for a term. | ~3s |

Global flags: `--country <cc>` (two-letter storefront, default us; use local
language for non-English storefronts, e.g. `--country de research "rezept"`),
`--fresh` (bypass the 24h cache).

## Interpreting output

- `popularity` (5-100): estimated search demand. Relative ordering reliable,
  absolute values are estimates.
- `difficulty` (1-100): how hard to rank top-10 when targeting the exact
  phrase in title/subtitle. <30 easy, 30-50 doable for a small app, 50-65
  needs traction, >65 dominated.
- `opportunity` (0-100): popularity gated by difficulty. Sort by this.
- `classification`: Sweet Spot / Hidden Gem / Good Target are worth
  targeting; High Competition / Low Volume / Avoid are not.
- `popularity_confidence`: `high` means the exact term appears in Apple's
  autosuggest (real users type it). `low` means popularity may be inflated
  by Apple backfilling results with big apps from broader terms — trust
  difficulty, distrust popularity, and say so when reporting.
- `difficulty_breakdown.is_brand_keyword`: true means the #1 app's publisher
  owns the term ("spotify"). Never recommend targeting a brand keyword.
- `difficulty_breakdown.leader_ratings` and `median_ratings`: cite these
  when explaining why a keyword is winnable ("the #3 app has 29 ratings").
- `est_daily_searches`: US-calibrated index, not a measurement. Only compare
  within one storefront, never across countries.

## Workflow for "which keywords should my app target"

1. Identify the app: `aso search "<app name>"` to get its numeric id,
   current title, and rating count. The title shows which keywords it
   currently targets (only title, subtitle, and the hidden keyword field are
   indexed for search — the description is NOT).
2. Baseline ranks: `aso rank <id> "<kw>"...` for keywords matching its
   features. Not appearing in ~200 results means invisible for that term.
3. Research the space: `aso research "<feature seed>"` for each feature
   area (2-4 seeds). Add hand-picked phrases with `aso score`.
4. Shortlist: take Sweet Spot / Hidden Gem / Good Target keywords with
   confidence `high`, weigh against the app's rating count — an app with
   150 ratings can win fields whose leaders have hundreds of ratings, not
   fields whose median is 10k+.
5. Recommend metadata: title (30 chars) carries the highest-value phrase,
   subtitle (30 chars) the next two, keyword field (100 chars,
   comma-separated, no spaces, no duplicates of title/subtitle words) the
   rest as single words — Apple combines words across all three fields into
   phrases, so "pantry" in the title plus "tracker" in the keyword field
   matches "pantry tracker".
6. After the user ships the change, re-run step 2 to measure movement.

## Multi-country analysis

Run steps 2-4 per storefront with `--country`. Fields differ sharply between
countries (a keyword impossible in the US is often a Sweet Spot in GB/AU).
For non-English storefronts, research native-language seeds — users search
in their own language, and an English-only title in e.g. Germany means zero
visibility on German terms.

## Performance notes

- Responses are cached 24h (`~/.cache/aso-cli/`), so overlapping runs are
  free and re-analysis is instant. Don't avoid re-scoring out of cost
  concern; only uncached calls are rate-limited.
- Budget ~3s per uncached keyword. For large sweeps (>40 keywords or
  multiple countries), run `aso` in a background shell and collect the JSON
  files afterward.
- HTTP 403/429 means Apple rate-limited the IP: wait ~60s, the tool retries
  automatically once.

## Caveats to pass on to the user

- All numbers are estimates; nobody outside Apple has real per-keyword
  search counts. Commercial tools reading Apple Search Ads "popularity" are
  quoting a relative ad metric that structurally broke in Oct 2025 (most
  keywords pinned to the floor value of 5).
- Difficulty answers "can I get indexed and into the top 10 by targeting the
  exact phrase", not "can I take #1 from an app with 200k ratings".
- Ratings volume is the user's ceiling: metadata gets an app indexed,
  review velocity moves it up.
