# Historical coverage by league and operator

## NCAAF coverage

NCAA Football (NCAAF) odds coverage starts with games on **October 1, 2026**
(US Eastern), during the 2026 regular season. The first covered game is
**Western Kentucky at New Mexico State**, scheduled for **8 PM Eastern**.
FanDuel, DraftKings, BetMGM, and Pinnacle all have retained priced markets
for this game, with captures beginning September 29.

Upcoming games count toward coverage once their odds have been retained.
`sportsbook-lines` includes those pregame observations; `sportsbook-closing-lines` and
`sportsbook-market-results` are published as games settle and processing completes.
Box scores support NCAAF results, while questions requiring unavailable
play-by-play remain `INDETERMINATE`. `game-state-timeline` is unavailable.

## Historical odds start dates

NCAAF was verified against retained odds on **2026-10-01**; the other leagues
were verified on **2026-09-25**. Dates below are the earliest **game dates
in US Eastern** with retained market data,
not the date collection began or a promise of complete seasons. Pregame
observations can precede the game date. Operator starts differ within a league.
Sportsbook starts require a successful response containing markets; captures
containing only errors or metadata do not count.

| League | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| NFL | 2025-10-16 | 2025 regular season, Week 7 |
| NBA | 2025-10-15 | 2025–26 preseason |
| MLB | 2025-10-15 | 2025 postseason, ALCS Game 3 |
| NHL | 2026-02-25 | 2025–26 regular season |
| NCAAM | 2026-02-10 | 2025–26 regular season |
| NCAAF | 2026-10-01 | 2026 regular season |

The league table describes the source history behind the sportsbook odds datasets
(`sportsbook-closing-lines`, `sportsbook-lines`, and the odds component of
`sportsbook-market-results`). The Kalshi and Polymarket rows in the per-league
tables below describe the prediction-market datasets, verified on
**2026-11-23**. Kalshi's three datasets start on the same date. Polymarket's
prices (`prediction-market-lines` and `prediction-market-closing-lines`) start
when its order books were first captured; `prediction-market-results` comes
from its settled contracts and reaches further back. They do not establish that every game, operator, market,
or derived Parquet file is available from that date onward. Stats backfills
can reach earlier than odds; they do not establish earlier sportsbook coverage
or the start of `game-state-timeline`.

### Start dates by league and operator

#### NFL

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2025-10-19 | 2025 regular season, Week 7 |
| DraftKings | 2025-10-16 | 2025 regular season, Week 7 |
| FanDuel | 2025-10-16 | 2025 regular season, Week 7 |
| Kalshi | 2025-08-07 | 2025 preseason, Week 1 |
| Pinnacle | 2026-09-09 | 2026 regular season, Week 1 |
| Polymarket — lines, closing lines | 2026-10-18 | 2026 regular season, Week 6 |
| Polymarket — results | 2025-08-09 | 2025 preseason, Week 1 |

#### NBA

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2025-10-15 | 2025–26 preseason |
| DraftKings | 2025-10-15 | 2025–26 preseason |
| FanDuel | 2025-10-15 | 2025–26 preseason |
| Kalshi | 2025-04-15 | 2024–25 postseason, play-in |
| Pinnacle | 2026-03-26 | 2025–26 regular season |
| Polymarket — lines, closing lines | 2026-10-16 | 2026–27 preseason |
| Polymarket — results | 2025-04-15 | 2024–25 postseason, play-in |

#### MLB

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2025-10-15 | 2025 postseason, ALCS Game 3 |
| DraftKings | 2025-10-15 | 2025 postseason, ALCS Game 3 |
| FanDuel | 2025-10-15 | 2025 postseason, ALCS Game 3 |
| Kalshi | 2025-04-16 | 2025 regular season |
| Pinnacle | 2026-03-26 | 2026 regular season |
| Polymarket — lines, closing lines | 2026-10-17 | 2026 postseason |
| Polymarket — results | 2025-04-16 | 2025 regular season |

#### NHL

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2026-02-25 | 2025–26 regular season |
| DraftKings | 2026-02-25 | 2025–26 regular season |
| FanDuel | 2026-02-25 | 2025–26 regular season |
| Kalshi | 2025-04-19 | 2024–25 postseason |
| Pinnacle | 2026-03-26 | 2025–26 regular season |
| Polymarket — lines, closing lines | 2026-10-16 | 2026–27 regular season |
| Polymarket — results | 2025-04-19 | 2024–25 postseason |

#### NCAAM

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2026-02-11 | 2025–26 regular season |
| DraftKings | 2026-02-10 | 2025–26 regular season |
| FanDuel | 2026-02-10 | 2025–26 regular season |
| Kalshi | 2025-11-03 | 2025–26 regular season |
| Pinnacle | 2026-03-26 | 2025–26 postseason |
| Polymarket — lines, closing lines | 2026-11-05 | 2026–27 regular season |
| Polymarket — results | 2025-11-03 | 2025–26 regular season |

#### NCAAF

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2026-10-01 | 2026 regular season |
| DraftKings | 2026-10-01 | 2026 regular season |
| FanDuel | 2026-10-01 | 2026 regular season |
| Kalshi | 2026-08-27 | 2026 regular season, Week 1 |
| Pinnacle | 2026-10-01 | 2026 regular season |
| Polymarket — lines, closing lines | 2026-10-16 | 2026 regular season, Week 7 |
| Polymarket — results | 2026-08-27 | 2026 regular season, Week 1 |

### Verify a historical query

Use these dates to choose a discovery window, then list games and check the
requested dataset for a returned game. Game discovery alone does not guarantee
a dataset artifact.

```bash
betflux games --league NFL --date-from 2025-10-16 --date-to 2025-10-19
betflux get sportsbook-closing-lines --game NFL_PIT_CIN_20251016 --operator FANDUEL
```

The NFL Week 7 label is recorded in the source box score and corroborated by
[the Bengals' 2025 schedule](https://www.bengals.com/schedule/2025/).
MLB's first game is [ALCS Game 3 on October 15, 2025](https://www.mlb.com/mariners/video/blue-jays-v-mariners-game-3-highlights-x1251).

## Sports data feed history

The sports datasets have their own history, verified against what the API
published on **2026-10-16**. Game datasets cover games that started at or after
**2025-01-01 00:00 UTC** (7 p.m. ET on 2024-12-31); nothing earlier is served,
and no start below promises a complete season. The first game date (ET) in each
league:

| League | `game-state`, `player-stats`, `team-stats` | `plays` |
|---|---|---|
| NBA | 2024-12-31, evening games only | 2024-12-31, evening games only |
| NHL | 2024-12-31, evening games only | 2024-12-31, evening games only |
| NCAAM | 2024-12-31, evening games only | not covered |
| NFL | 2025-01-04 | 2025-01-04 |
| MLB | 2025-02-20 | 2025-02-20 |

`betflux games` lists games from before that start. Their sports datasets answer
`404` `not-yet-available` and never will be published: for a game that started
before 2025-01-01 00:00 UTC, read that answer as "not covered", not "wait".

`injuries` starts with its first published observation, 2026-03-20 (UTC), in
every league.

A league dataset's manifests report the history before its first observation
as a coverage gap, so every `injuries` revision carries `degraded` coverage and
`provisional` status. For a league feed, neither word tells one revision from
another.
A covered scope with nothing published yet, usually a game that has not
started, answers `404` with problem type `not-yet-available`. Treat that as
"not published yet", not as missing coverage to retry: wait for publication.
See [sports-data.md](sports-data.md).
