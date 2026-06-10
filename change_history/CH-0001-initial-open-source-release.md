# CH-0001: Initial open-source release of aso-cli

**Date**: 2026-06-10
**Type**: implementation

## What changed

First public release (v0.1.0, commit 261ac3d). A single-file,
zero-dependency Python 3.10+ executable `aso` with five commands:
research, score, rank, suggest, search. Data comes from three public
Apple endpoints (iTunes Search API, MZSearchHints autosuggest with the
X-Apple-Store-Front header, and the apps.apple.com search page's embedded
serialized-server-data JSON for true ranking order), with a 24h response
cache in ~/.cache/aso-cli and per-endpoint-family rate spacing (3s search,
1s hints, 1.5s SSR). Includes a Claude Code agent skill in skill/SKILL.md.

## Why

App Store keyword research without accounts, API keys, or paid services.
Scoring follows the open RespectASO methodology (AGPL-3.0, independently
reimplemented; this project is AGPL-3.0): a 6-signal popularity estimator
(5-100), a 7-factor weighted difficulty score (1-100) with backfill
corrections (small-result-set cap, weak-leader cap, backfill discount,
brand-keyword detection), an opportunity score, and classification labels.
Autosuggest doubles as a popularity-confidence cross-check. Known
calibration weakness documented up front: compound keywords with giant
head terms read 10-15 popularity points hot.

## Affects

- ARCHITECTURE: INDEX.md created (shape, endpoints, scoring internals, calibration weaknesses)
- ROADMAP: INDEX.md created (prefix-depth autosuggest signal, exact-relevance leader weighting, Google Play, CSV output)
- README: no (already comprehensive, OSS-facing, kept as-is)
