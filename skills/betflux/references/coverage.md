# Historical coverage by league and operator

## NCAAF coverage

NCAA Football (NCAAF) odds coverage starts with games on **October 1, 2026**
(US Eastern), during the 2026 regular season. The first covered game is
**Western Kentucky at New Mexico State**, scheduled for **8 PM Eastern**.
FanDuel, DraftKings, BetMGM, and Pinnacle all have retained priced markets
for this game, with captures beginning September 29.

Upcoming games count toward coverage once their odds have been retained.
`sportsbook-lines` includes those pregame observations; `closing-lines` and
`market-results` are published as games settle and processing completes.
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

These dates describe the source history behind the odds datasets
(`closing-lines`, `sportsbook-lines`, and the odds component of
`market-results`). They do not establish that every game, operator, market,
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
| Pinnacle | 2026-09-09 | 2026 regular season, Week 1 |

#### NBA

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2025-10-15 | 2025–26 preseason |
| DraftKings | 2025-10-15 | 2025–26 preseason |
| FanDuel | 2025-10-15 | 2025–26 preseason |
| Pinnacle | 2026-03-26 | 2025–26 regular season |

#### MLB

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2025-10-15 | 2025 postseason, ALCS Game 3 |
| DraftKings | 2025-10-15 | 2025 postseason, ALCS Game 3 |
| FanDuel | 2025-10-15 | 2025 postseason, ALCS Game 3 |
| Pinnacle | 2026-03-26 | 2026 regular season |

#### NHL

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2026-02-25 | 2025–26 regular season |
| DraftKings | 2026-02-25 | 2025–26 regular season |
| FanDuel | 2026-02-25 | 2025–26 regular season |
| Pinnacle | 2026-03-26 | 2025–26 regular season |

#### NCAAM

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2026-02-11 | 2025–26 regular season |
| DraftKings | 2026-02-10 | 2025–26 regular season |
| FanDuel | 2026-02-10 | 2025–26 regular season |
| Pinnacle | 2026-03-26 | 2025–26 postseason |

#### NCAAF

| Operator | First game date (US Eastern) | Season / phase at start |
|---|---|---|
| BetMGM | 2026-10-01 | 2026 regular season |
| DraftKings | 2026-10-01 | 2026 regular season |
| FanDuel | 2026-10-01 | 2026 regular season |
| Pinnacle | 2026-10-01 | 2026 regular season |

### Verify a historical query

Use these dates to choose a discovery window, then list games and check the
requested dataset for a returned game. Game discovery alone does not guarantee
a dataset artifact.

```bash
betflux games --league NFL --date-from 2025-10-16 --date-to 2025-10-19
betflux get closing-lines --game NFL_PIT_CIN_20251016 --operator FANDUEL
```

The NFL Week 7 label is recorded in the source box score and corroborated by
[the Bengals' 2025 schedule](https://www.bengals.com/schedule/2025/).
MLB's first game is [ALCS Game 3 on October 15, 2025](https://www.mlb.com/mariners/video/blue-jays-v-mariners-game-3-highlights-x1251).
