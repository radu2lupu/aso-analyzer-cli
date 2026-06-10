# Architecture

How `aso` works internally. The OSS-facing overview lives in [README.md](../README.md);
this file documents the implementation details the README intentionally omits.

## Shape

One executable file, `aso`, Python 3.10+ standard library only. No package,
no dependencies, no install step beyond a symlink. Five subcommands
(`research`, `score`, `rank`, `suggest`, `search`) dispatch via argparse
`set_defaults(fn=...)`. All results are a single JSON object on stdout;
progress and errors go to stderr. Exit 0 on success, 1 on network/HTTP
failure, 130 on interrupt.

A Claude Code agent skill ships in `skill/SKILL.md`.

## Data sources (all public, no auth)

| Source | URL | Rate spacing | Used for |
|---|---|---|---|
| iTunes Search API | `https://itunes.apple.com/search` (`entity=software&media=software`) | 3.0s | competitor landscape per keyword: ratings, stars, release dates, sellers |
| App Store autosuggest | `https://search.itunes.apple.com/WebObjects/MZSearchHints.woa/wa/hints` with `X-Apple-Store-Front: {id}-1,29` header | 1.0s | keyword expansion (`research`, `suggest`) and the popularity-confidence cross-check |
| App Store web search SSR | `https://apps.apple.com/{cc}/iphone/search` | 1.5s | true ranking order for `rank` (parsed from the `<script id="serialized-server-data">` embedded JSON: shelf lockup `adamId`s, then `nextPage.results`) |

Storefront ids for the autosuggest header are a hardcoded 27-country table
(`STOREFRONTS`); unknown countries fall back to the US storefront for
suggestions only. The iTunes API ordering does NOT mirror device search
ranking; the SSR page does, which is why `rank` scrapes the web page.

### HTTP layer

`_http_get(url, family, headers)`: per-URL 24h file cache in
`~/.cache/aso-cli/` (sha256(url).json; `ASO_CACHE_DIR` / `ASO_CACHE_TTL`
env overrides; `--fresh` bypasses reads but still writes), per-family
rate-limit spacing (`_RATE`), and 3 attempts with retry on 429/500/502/503
(10s for 429, exponential 1s/2s otherwise) and on URLError/timeout.

## Scoring model

Independent reimplementation of the open RespectASO methodology
(AGPL-3.0; this project is AGPL-3.0 as well). All scores are computed from
the iTunes search results for the keyword (default 25 apps).

`_log_interp(value, bands, ceiling)` is the shared calibration primitive:
smooth log interpolation between (threshold, score) points, linear below
the first band, ceiling above the last.

### Popularity (5-100), `popularity_score()` — 6 signals

1. Result count (0-25): `min(25, n * 2.5)`.
2. Leader strength (0-30): log-interp of the max rating count in the top
   half of results, bands (10,1) (100,5) (1k,10) (10k,17) (100k,24) (1M,30).
3. Title match density (0-20): share of titles matching the phrase
   (exact or all-words), `min(20, ratio * 40)`.
4. Market depth (0-10): log-interp of median ratings, bands
   (10,0.5) (100,3) (1k,5) (10k,8) (50k,10).
5. Specificity penalty (0 to -28): linear by keyword word count,
   points (1,0) (2,-3) (3,-8) (4,-15) (5,-22) (6+,-28).
6. Exact-phrase bonus (0-15): `min(15, exact_ratio * 50)`.

Corrections: ratio signals (3 and 6) are dampened linearly below n=10
samples; a backfill-awareness relevance factor (mean title evidence * 2.6,
clamped 0.3-1.0) scales signals 1, 2, and 4. Floor 5, cap 100.

Title evidence (`_title_evidence`) is a match hierarchy: exact phrase =
1.0, all words any order = 0.85 + 0.15 * proximity, partial = up to 0.5
by token overlap.

### Difficulty (1-100), `difficulty_score()` — 7 weighted factors

| Factor | Weight | Computation |
|---|---|---|
| rating_volume | 30% | log-interp of median ratings, bands (50,5)…(100k,95) |
| dominant_players | 20% | continuous log10 dominance per app (ceiling 10M ratings), top half double-weighted |
| review_velocity | 10% | log-interp of median ratings/year, bands (10,5)…(50k,95); 50 if no dates |
| rating_quality | 10% | review-weighted (log1p) average stars, piecewise bands 3.0→20 … 4.5→85 |
| market_age | 10% | average field age in years, piecewise bands 0.5y→10 … 10y→100; 50 if no dates |
| publisher_diversity | 10% | unique sellers / n * 100 |
| title_relevance | 10% | titles matching (exact or all-words) / n * 100 |

Same small-sample dampening (n/10) applies to diversity, relevance,
dominance, quality; the backfill relevance factor scales diversity,
quality, and age.

Post-processing overrides (recorded in `difficulty_breakdown.override`),
correcting for Apple padding thin result sets with big apps from broader
terms:

- `small_result_set`: hard caps for n<=4 (1→10, 2→20, 3→31, 4→40).
- `weak_leader`: if not a brand keyword and the #1 app has <1,000 ratings,
  cap at `15 + 35 * log10(leader+1)/log10(1001)`, blended back toward the
  raw score by title-match ratio when matches exceed 20%.
- `backfill_discount`: additionally, if title-match ratio < 0.2, multiply
  by a discount built from the match ratio and leader strength.
- Brand detection (`_is_brand`): all keyword tokens appear in the #1
  app's seller name, and either the leader has >=1k ratings or the
  surrounding independent field is strong (median >=10k). Exposed as
  `is_brand_keyword`; brand keywords are exempt from the weak-leader cap.

### Opportunity (0-100) and classification

Popularity maps to estimated US daily searches through the 20-point
`_POP_TO_SEARCHES` table (5→1 … 100→32,000, piecewise linear). Opportunity
= log-normalized searches (`log10(1+s)/log10(1+32000)`) x quadratic
difficulty gate (`1 - (d/100)^2`) x 100: difficulty hurts a little at 30
and brutally at 80.

Classification thresholds (`classify()`): Sweet Spot (pop>=40, diff<=40),
Hidden Gem (pop 25-39, diff<=30, opp>=30), Low Volume (pop<15), High
Competition (diff>=65), Good Target (opp>=55), Avoid (opp<=25), else
Moderate.

### Autosuggest confidence cross-check

`_autosuggest_signal()`: type the full keyword into autosuggest. Exact
term present → `high` (plus 1-based `autosuggest_rank`); all words present
in some suggestion → `medium`; neither → `low`; endpoint failure →
`unknown`. `low` means popularity may be backfill-inflated: trust
difficulty, distrust popularity.

## Known calibration weaknesses

- **Compound keywords with a giant head term** ("recipe X", "photo X")
  read 10-15 popularity points hot: the big apps Apple backfills into
  results partially count as demand evidence (leader strength and market
  depth survive the relevance scaling). `popularity_confidence` is the
  designed catch; check it on any compound keyword.
- Difficulty answers "can I get indexed and into the top 10 targeting the
  exact phrase", not occupant strength. Tools reporting occupant strength
  read higher on backfilled keywords; both are valid, different questions.
- Popularity is calibrated within a storefront; cross-country comparisons
  are not meaningful. `est_daily_searches` is a US-baseline index, not a
  measurement.
- Absolute values throughout are estimates; relative ordering is the
  reliable signal.

history: CH-0001
