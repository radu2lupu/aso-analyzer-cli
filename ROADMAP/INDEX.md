# Roadmap

Current state: v0.1.0, initial open-source release. Five commands
(research, score, rank, suggest, search) working against public Apple
endpoints. No backlog commitments; items below are candidates, unordered.

## Candidate improvements

- **Prefix-depth autosuggest demand signal.** Use how few characters of a
  keyword you must type before Apple suggests it as an additional demand
  signal (earlier completion = stronger demand), feeding popularity rather
  than only the post-hoc confidence label.
- **Exact-relevance weighting of leader strength in popularity.** Weight
  the leader-strength signal by whether the leading apps actually target
  the phrase, to shave the 10-15 point hot read on compound keywords with
  giant head terms (see ARCHITECTURE, known calibration weaknesses).
- **Google Play support.** Same workflow against Play Store public data.
- **CSV output.** Alternative output format for spreadsheet workflows.

history: CH-0001
