# Changelog

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
