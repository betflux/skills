# Changelog

## 0.8.0 — 2026-11-23

0.7.0 was never tagged; its changes ship in this release.

- The prediction-market datasets `prediction-market-lines`,
  `prediction-market-closing-lines` and `prediction-market-results` (Kalshi and
  Polymarket), with a reference page each and the skill's guidance on reading
  an exchange price against a book's. `prediction-market-lines` is served live
  (`live-board --dataset prediction-market-lines`). The error table adds them
  to 403 `plan-excludes` (`limits.prediction_markets`). Needs betflux 0.13.0.
- The six market dataset references list `outcome_key` and `settlement_key`
  as filters. A new "Comparing venues" section of `SKILL.md` says how to find
  the same bet at several venues, books against books as well as against
  exchanges: group on `outcome_key` (the same game result pays), add
  `settlement_key` for the identical contract (the same settlement terms too;
  `terms_status` says how sure), and fall back to `market_type`, `period`,
  `side` and the line only where `outcome_key` is null. It used to line an
  exchange up against a book on those columns alone. `references/recipes.md`
  has a worked join ("The same bet at every venue").
- `references/coverage.md` lists Kalshi and Polymarket start dates by league.
  Polymarket's prices start at 2026-10-16; its results reach back to 2025.
- Overtime markets ("the game reaches overtime") carry `outcome_key`:
  `outcome_min` and `outcome_max` count the overtime periods played (extra
  innings in MLB), so Yes is `outcome_min` 1 and No is `outcome_max` 0. Games
  built before the change leave them unkeyed until they are rebuilt.
- Odd/even markets, game and team, carry `outcome_key`, with the winning
  parity in a new `outcome_parity` column (`odd` or `even`; a score of 0 is
  even) and `outcome_min`, `outcome_max` and `outcome_push` null. The column
  is on the six market datasets from `sportsbook-lines` v9,
  `sportsbook-closing-lines` v13, `sportsbook-market-results` v15,
  `prediction-market-lines` and `prediction-market-closing-lines` v4 and
  `prediction-market-results` v5.
- `sportsbook-market-results` (dataset v17) is one row per offer: one
  selection at one line, named by `offer_key`. A book that moves the line on
  one selection (FanDuel main, period and over/under markets, Pinnacle props)
  gets a settled row per line it held, each with its own close.
  `sportsbook-closing-lines` (v15) has no row for a selection quoted on more
  than one line before the game started, so a closing row's opening and close
  are always on one line. Join results to the lines and closing-lines datasets
  on `offer_key`, not `market_key` plus selection id; `market_key` stays for
  market-level joins. Games built before v17 keep one row per selection
  until they are rebuilt, so a pull can mix the two; check
  `dataset_version`. `references/recipes.md` shows the join.
- `sportsbook-market-results` outcomes include `VOID` and `PARTIAL`, with
  stake fractions, `settlement_rule`, `settlement_source` and `settled_at`
  (dataset version v10). Results are described as settled;
  `graded_at` is deprecated.
- Every game counts toward the allowance: no plan reads recent games free any
  more, so the skill no longer mentions `free_recent_game_days` or `exempt`
  as a cost (both stay in the API, always null and 0).
- 402 `subscription-required` is now rare and brief: the key's record was
  cached before the account's plan was known (every account has at least the
  free sample). `SKILL.md` and `references/python.md` say to retry once after
  a few minutes and report it only if it persists. It used to mean the
  account had no active subscription, with an `upgrade_url`.
- `references/http-api.md` lists the live routes per dataset,
  `/v1/games/{game_id}/{dataset}/live/segments` and `/live/board`; the
  `sportsbook-lines` paths without `/live` are marked deprecated.
- `references/http-api.md` no longer offers `exempt` as a preview cost, an
  `x-betflux-game` value or a `/v1/me/games` cost (`free_recent_game_days`
  and `games_exempt` are documented as always null and 0), and the preview
  adds `restricted` (outside a trial key's sample) and `enforced`.
- `references/cli.md`: `--outcome` takes `VOID` and `PARTIAL` and works on both
  results datasets; range addressing (`--league`, `--date-from` and `--date-to`
  together) covers the prediction-market closing-lines and results datasets,
  and `--game` every dataset but `injuries`; `--player-id` matches either
  player-id list on the six lines datasets; `games --status` lists all six
  statuses; the `--dry-run` JSON lists `restricted` and `enforced`.
- `references/recipes.md` ("Did these bets win?", "Model-ready frames") lists
  every outcome and prices a settled row with the stake fractions:
  return = stake × (`stake_won_fraction` × decimal odds +
  `stake_refunded_fraction`).
- `SKILL.md`, `references/cli.md`, `references/python.md` and
  `references/reference-data.md`: `/v1/games` takes at most 366 days per
  query and neither the CLI nor the SDK splits a longer range; a range read
  skips games without a file silently, so compare it with the game list
  before calling it complete. The error table adds 400 `validation-error`.
- `SKILL.md` lists NCAAF among the leagues and says why a close can be
  missing or null; `sportsbook-closing-lines`' closing columns say the same.
- `references/datasets/sportsbook-lines.md` describes `timestamp` and the
  de-vig columns: the six methods (logit is the odds-ratio method), when each
  is null, the 5% default synthetic margin, and that artifacts before v5 need
  a rebuild before point-in-time use. In `sportsbook-market-results`, tied
  leaders in a most-of market are `PARTIAL` (`dead_heat`) from v12, not
  `INDETERMINATE`.

## 0.6.0 — 2026-11-12

- Datasets about prices and bets are named for their venue:
  `closing-lines` is now `sportsbook-closing-lines` and `market-results` is
  now `sportsbook-market-results`. Their references move to
  `references/datasets/sportsbook-closing-lines.md` and
  `references/datasets/sportsbook-market-results.md`, and the CLI and Python
  examples use betflux 0.11.0's names (`bf.sportsbook_closing_lines`,
  `bf.sportsbook_market_results`). The old names still work in the API, the
  CLI and the SDK, flagged deprecated. Needs betflux 0.11.0.

## 0.5.0 — 2026-10-28

- The MSF datasets are now called sports data, so the skill no longer ties
  them to one provider. `references/msf.md` is now
  `references/sports-data.md` (and the coverage section is now
  `#sports-data-feed-history`), and the Python examples use betflux
  0.10.0's names: `bf.sports_dataset()`, `SportsDataSnapshot`,
  `SportsDataRevision` and `SportsDataIntegrityError`. The old `Msf*` names
  still work in the SDK but warn. Needs betflux 0.10.0.

## 0.4.0 — 2026-10-27

- Usage is metered in distinct games per billing period, never in rows
  (ADR-0092). `SKILL.md` gains a "Limits on this key" section right after
  setup: the counting rules (a game counts once per period on its first read
  of any dataset; re-reads, other datasets, listings, reference data and
  injuries are free; recent games are free on plans with that window; at the
  cap only a new game is refused; the trial sample is the first weekend of
  each month; excluded features are 403 `plan-excludes`) and six habits —
  `betflux keys check --json` first, `get … --dry-run` before a range pull,
  first-weekend games on a trial key, prefer games already counted (`betflux
  usage games`, `betflux games --cost`), skip live when `limits.live` is
  false, stop on 429 `quota-exhausted`. Every number comes from `/v1/me` at
  runtime; the skill states no allowance, price or rate.
- The error table lists 429 `quota-exhausted` (games), 403 `plan-excludes`,
  429 `rate-limited` and `throttled`, and `history-restricted` now means a
  trial key asked for a game outside the first-weekend sample.
- Row-quota wording is gone from `SKILL.md`, `references/python.md`,
  `cli.md`, `msf.md` and `http-api.md`. The datasets table keeps rows per
  game as download size, with the note that cost is per game, not per row.
- `references/cli.md`: `keys check --json`, `betflux usage` and `usage games
  [--league] [--from] [--to] [--json]`, `games --cost`, `get --dry-run`,
  `--allow-partial` and `--quiet`, the `games:` stderr line after `get`,
  `live-board` and `live-tail`, and the per-error messages.
- `references/python.md`: `Client.usage()`, `preview_games()`,
  `counted_games()`, `Dataset.preview()` and `iter(games=)`, `.cost` on
  reads, `GameDone.counted` / `.games_remaining`, `PlanRequiredError`, and
  `QuotaExceededError` in games.
- `references/http-api.md` (generated): the `/v1/me` shape, `POST
  /v1/me/games/preview`, `GET /v1/me/games`, the `x-betflux-game` and
  `x-betflux-games-remaining` headers and the 429/403 bodies.
- The example game is `NBA_ORL_DET_20260503` (Magic at Pistons, the Sunday of
  May 2026's first weekend) and the workflow's range is that weekend, so a
  trial key can run every example in the skill as written; 0.2.0's May 2026
  examples were chosen for the sample month this rule replaces.
- Needs betflux 0.9.0 for the usage commands and the preview.

## 0.3.0 — 2026-10-27

- `game-state-timeline` is the estimated game state with confidence (dataset
  v3, beta): say that every value is an estimate, and document its columns,
  the as-of join to line moves, and that a row is a best estimate built from
  later reports, including the full play-by-play. Drop the `--field` /
  `--source` filters. Older betflux versions still read it but show every
  column; 0.8.0 shows the score, period, clock and situation.

## 0.2.0 — 2026-10-16

- Add the MSF datasets — game state, plays, player stats, team stats and
  injuries — with snapshot/changes protocol, CLI and SDK references, and
  one generated page per dataset. They need betflux 0.7.0.
- Document when each MSF dataset's history starts (games from 2025-01-01
  00:00 UTC), and that a `404` `not-yet-available` means a game or league has
  nothing published yet: wait for publication rather than retry, except for a
  game before that start, which never will be.
- Document the MSF feeds' current limits: `freshness` is always null, no
  archives are published, `metric` covers only mapped stats, and the injuries
  feed mirrors a once-a-day source report that is not a complete list.
- Move the examples to May 2026, the demo key's sample month.

## 0.1.6 — 2026-10-07

- Document the game record's score, venue and weather fields on
  `betflux games` / `bf.games()`, and the CLI's curated table + `--wide`.
- Generated `references/reference-data.md`: columns and filters for leagues,
  teams, players and games from the OpenAPI spec, with the current-state rule
  (use `game-state-timeline` / `sportsbook-lines` for history).

## 0.1.5 — 2026-10-03

- Document the `is_mainline` column and the Python `mainline=True` filter on
  the three lines datasets.

## 0.1.4 — 2026-09-25

- Use saved credentials from `betflux login` for CLI commands and Python
  `Client()`.
- Check for an existing login before requesting setup. Document browser approval,
  explicit credentials for automation/raw HTTP, and protected-file storage for
  headless hosts.
- Document login/logout commands, credential precedence, and local recovery.
- Keep secrets out of agent conversations and let the CLI/SDK access credential
  storage directly. Clarify that project SDK installation is separate from the
  CLI's `uv tool` environment.
