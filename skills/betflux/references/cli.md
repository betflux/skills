# betflux CLI reference

Every command and flag. For query behavior and planning, see
`SKILL.md` — this file is the surface, not the strategy.

## Global options

These go **before** the command: `betflux --timeout 5 games`, not
`betflux games --timeout 5`.

| Option | Env | Default | Notes |
|---|---|---|---|
| `--api-key` | `BETFLUX_API_KEY` | — | Prefer the env var or `--api-key-file`; a key on the command line is visible in `ps` and shell history |
| `--api-key-file PATH` | — | — | Reads the first line of the file |
| `--base-url` | `BETFLUX_BASE_URL` | `https://api.betflux.ai` | |
| `--timeout` | — | `30.0` | Per-request network timeout, seconds |
| `--version` | — | — | Reads installed distribution metadata |

Credentials are resolved in order: `--api-key`, `BETFLUX_API_KEY`,
`--api-key-file`, then saved login for the selected API URL.

## Commands

### `betflux login`

Opens the browser and prints a URL and code. The user
signs in, enters the code, and approves access. Saves a new installation key in
the system keychain for CLI commands and Python `Client()`. Existing saved login
is reused without creating another key. Onboarding and account key limits apply.

- `--no-browser`: print the link without opening a browser.
- `--credential-store keyring|file`: default `keyring`; explicitly select `file`
  for protected local file storage on headless hosts.
- `--name TEXT`: override the default installation name.
- `--auth-url URL`: website origin; required for a custom API URL, or set
  `BETFLUX_AUTH_URL`. Put global `--base-url` before `login`.

### `betflux logout`

Revokes the saved installation key and removes local login. Explicit keys are
unaffected. Failed revocation retains the local key for retry; successful
revocation may take five minutes to propagate through the API edge cache.

`--local-only` clears local login without contacting the server. It can recover
malformed metadata or an unavailable keychain reference, and warns if a keychain
entry may remain. Revoke the old key through the account page separately. Use
the same `--base-url` when resetting a custom API login.

### `betflux plugin install --codex | --claude | --cursor`

Specify one agent per invocation. Run the command separately for each agent
you use; installing in both Claude Code and Codex is supported.
Claude Code and Codex register `betflux/skills` and install through
their native plugin managers; Claude uses user scope. The host CLI must already
be installed. An API key is not needed for plugin installation.

Rerunning refreshes the repository, updates the plugin, and enables it.
`--cursor` prints manual setup
instructions and exits nonzero without installing. The command never copies
the SDK's bundled skill or deletes existing files. Start a new
agent session after installing or updating; use the native manager for removal.

### `betflux agent-guide`

Prints the guide bundled with the installed SDK to stdout, without network
access or installation. This snapshot may differ from an independently
released plugin.

### `betflux keys check [--json]`

Validates the key and reports its plan, the period's games and the plan's
limits, every number from `GET /v1/me`. Free: it never counts a game and
works at the allowance.

```
key valid (https://api.betflux.ai)
plan: <label> (<TIER>) — <n> requests/min
games: <used> of <allowance> used this period (<pct>%) · <left> left · <exempt> recent (free) · resets YYYY-MM-DD
limits: games from the last <d> days are free · live: yes|no · league datasets: yes|no
```

An uncapped plan prints `games: <used> used this period · no cap`; a trial key
adds `sample: first-weekend games only`. `--json` prints the raw `/v1/me`
body (`tier`, `plan`, `limits`, `usage`, `upgrade_url`) for agents to parse;
see [http-api.md](http-api.md#limits-and-usage) for the fields. On a gateway
from before games metering the games lines are absent.

### `betflux usage` / `betflux usage games`

`betflux usage` prints the same plan and games lines as `keys check`.
`betflux usage games` lists the games counted or exempt this period, newest
read first — every one is free to read again, in any dataset, until the
period resets. Listing them counts nothing.

| Option | Notes |
|---|---|
| `--league` | e.g. `NFL` |
| `--from`, `--to` | Inclusive game date, `YYYY-MM-DD` |
| `--json` | A JSON array of `{game_id, first_read_at, first_dataset, cost}` (`cost` is `counted` or `exempt`) |

### `betflux datasets [--format]`

Lists datasets with title, leagues, and column count. No
auth needed. The table view flattens the nested column list; `--format json`
or `record` keeps the full detail.

### `betflux games`

Lists games — the id plus the full game record: schedule (`start_time`,
`status`, `season`, `week`, `broadcasters`), both teams, the score
(`home_score`, `away_score`, `current_period`, `current_period_seconds_remaining`,
`period_scores_home/away`), the venue (`venue_name`, `venue_city`,
`venue_latitude/longitude`, `venue_capacity`, `venue_surface`,
`venue_has_roof`) and game-time weather (`weather_type`,
`weather_temperature_f`, `weather_wind_speed_mph`, `weather_precipitation_type`, …).
Every field beyond the teams and schedule is nullable.

| Option | Notes |
|---|---|
| `--league` | e.g. `NBA` |
| `--date-from`, `--date-to` | Inclusive ET game date, `YYYY-MM-DD` |
| `--team` | Canonical team name; matches home or away |
| `--status` | `SCHEDULED`, `STARTED`, `COMPLETED` |
| `--id` | Fetch one game by id instead of listing |
| `--cost` | Adds a `free_to_read` column: `true` when the game is already counted or exempt this period, so reading it costs nothing; `false` when it would count as a new game. One preview call (`POST /v1/me/games/preview`) over the listed ids; opt-in because it is one more request |
| `--format`, `--columns`, `--wide` | Output control; the default table is a curated subset, `--wide` shows every column |

Listing games is free and counts nothing. `--id` is exclusive: pairing it with any filter is an error. For one game's
whole record use `--id <id> --format record`. The record is current state
(latest score, status, roster), not history — see
[reference-data.md](reference-data.md).

### `betflux leagues` / `betflux teams` / `betflux players`

Reference data. `teams` takes `--league`; `players` takes `--league` and/or
`--team-id`. All support `--format` and `--columns`.

### `betflux get <dataset>`

Downloads dataset files and returns matching rows. Each game read counts
once against the period's games allowance on its first read of any dataset;
re-reads and the other datasets of a counted game are free.

**Cost first.** A date-range pull previews its cost before downloading
anything (one free `POST /v1/me/games/preview` over the games in the range):

- `--dry-run` prints the preview and exits 0 without downloading or counting
  anything: how many games are new (would count), already counted (free) and
  recent (free), and whether the pull fits in what is left. With `--format
  json` the preview is JSON (`games` per id, `new`, `already`, `exempt`,
  `invalid`, `games_used`, `games_allowance`, `games_remaining`, `fits`,
  `over_by`). With `--game` it previews that one game; on a league
  snapshot it is refused, since league snapshots never count a game.
- A pull that would exceed the allowance aborts before downloading (exit 1,
  `this pull would exceed your games allowance: …`). `--allow-partial` pulls
  what fits instead — every already-counted or exempt game plus the first new
  ones that fit, in listing order — and prints a `note:` naming how many were
  skipped.
- On a gateway without the preview endpoint a plain pull proceeds without the
  check (the API's own admission still refuses a game past the allowance) and
  `--dry-run` fails, since it has nothing to report.

After the read, one stderr line reports what it cost and what is left:
`games: N new, M already counted · X of Y left · resets YYYY-MM-DD`, with
`, K recent (free)` appended when the plan's recent-game window applied. An
uncapped plan reads `games: N read · U used this period · no cap`. `--quiet`
silences it (and the `note:` lines). A single-game `get` on a plan that sends
no cost headers prints no line rather than spending an extra request.

**Addressing** — exactly one of:

| Form | Datasets |
|---|---|
| `--date-from` + `--date-to` (+ optional `--league`) | `closing-lines`, `market-results` |
| `--game <id>` | the four original datasets plus the sports datasets `game-state`, `plays`, `player-stats`, `team-stats` |
| `--league <code>` without `--game` | `injuries` |

`sportsbook-lines`, `game-state-timeline`, and the four sports data game datasets are
`--game`-only. `injuries` is `--league`-only.

**Filters** (rows are filtered locally after download; team and league also
narrow game discovery on range queries, reducing downloads):

| Option | Applies to |
|---|---|
| `--operator` | the three lines datasets |
| `--market-type` | the three lines datasets |
| `--team` | the three lines datasets; matches home or away |
| `--side` | the three lines datasets, e.g. `HOME`, `OVER` |
| `--player-id` | Original line/result datasets, player stats and injuries |
| `--outcome` | `market-results` only: `WON`, `LOST`, `PUSH`, `INDETERMINATE` |
| `--metric` | Player/team stats; applies the reviewed restatement policy |
| `--team-id` | Player/team stats and injuries |

A filter naming a column the dataset lacks is rejected before any download.

**Bounding and output**

| Option | Notes |
|---|---|
| `--limit N` | Range queries only: stop after N matching rows. Does not cap downloaded rows; sparse filters can download the entire range |
| `--output PATH` | Original datasets: with `--game`, save raw Parquet. Sports data snapshots: save `.parquet` or `.csv` for the dataset's game or league scope |
| `--columns a,b,c` | Override the default projection |
| `--wide` | Table only: every column instead of the curated subset |
| `--format` | `table` (default), `record`, `json`, `jsonl`, `csv` |
| `--no-progress` | Also `BETFLUX_NO_PROGRESS=1`; auto-disabled when stderr is not a terminal |
| `--dry-run` | Preview the cost and stop; nothing is downloaded or counted (see **Cost first**) |
| `--allow-partial` | Date ranges only: when the pull would exceed the allowance, download the games that fit instead of aborting |
| `--quiet` | Silence the `games:` usage line and the cost notes on stderr |

`jsonl` and `csv` stream rows as each game's file arrives. `record` prints
vertical `key: value` blocks — the readable choice for a single wide row.

For a sports dataset, `get` reads the current snapshot; its files carry the frozen public schema.
`game-state`, `plays`, `player-stats`, `team-stats` and `injuries`
have separate columns and revision feeds. A game snapshot counts the game once
per period like any other read of it; an `injuries` snapshot counts nothing,
but needs a plan with `limits.league_datasets` (403 `plan-excludes`
otherwise). See [sports-data.md](sports-data.md) and the five generated dataset pages.

**Live games.** A `sportsbook-lines` game that has not settled has no final
file; `get` assembles its rows from the live feed and prints
`provisional: <game> is live; …` on stderr. stdout is unchanged, so pipes are
unaffected. Report the provisional status when you pass such rows on. Each run
downloads every live segment again; the game counts once per period on its
first read, so a rerun is free, but it is still a full re-download — use
`live-tail` rather than looping `get`. Live lines need a plan with
`limits.live`: on any other key a live game's segments answer 403
`plan-excludes`, and `get --game` stops with `your plan does not include live
lines` (exit 1). `--output` on a live game saves the segments re-encoded by
the SDK, not a server file.

### `betflux live-board <game>`

The current board for a live game: one row per selection each operator is
believed to be quoting, with `quote_state` (`live` / `stale_suspect`),
`last_changed_at` and `operator_observed_through` on top of the
`sportsbook-lines` columns. A snapshot, not history. A board read counts the
game once per period like any other read of it, so the board of a game
already counted is free; it needs a plan with `limits.live` (403
`plan-excludes` otherwise). Once the game has settled it exits 0 and points
at `betflux get sportsbook-lines --game <game>`. It ends with the same
`games:` stderr line as `get`.

| Option | Notes |
|---|---|
| `--operator` | One operator's board file instead of every operator's |
| `--output PATH` | Save as Parquet instead of rendering rows |
| `--columns`, `--wide`, `--format` | As `get` |
| `--quiet` | Silence the `games:` usage line |

### `betflux live-tail <game>`

Follows a live game's new lines until Ctrl-C. Each poll lists what is new
since the cursor (a position, not a timestamp; the listing is free) and
downloads the segments it names — the open tail on every poll in which it
grew. The game counts once per period, on the first segment read; every later
poll is free. A first poll with neither `--cursor` nor `--from-now`
backfills the whole history published so far. Stops by itself once the game
settles, pointing at
`betflux get sportsbook-lines --game <game>` for the settled history, and
prints the `games:` stderr line when it ends. Live lines need a plan with
`limits.live`; otherwise the tail stops at once with the `plan-excludes`
message and exit 1.

| Option | Notes |
|---|---|
| `--every N` | Seconds between polls, default `5.0`; the API caches live listings for 5 s, so anything faster re-downloads the same bytes |
| `--cursor CURSOR` | Resume from the cursor the previous tail printed when it stopped (e.g. `v5:3:1200`), instead of backfilling |
| `--from-now` | Take the current position from one free listing and follow only rows published after it |
| `--output PATH` | Append new rows to a file instead of printing them; resume with `--cursor` so the file holds the history once |
| `--columns`, `--format` | As `get`, except `--format` defaults to `jsonl` — rows arrive per poll — and `json` is refused (an array per poll is not one document) |
| `--quiet` | Silence the `games:` usage line |

Ctrl-C is the normal way to end it and exits 0, not 130; the line it prints
carries the cursor reached. A live log rebuilt under the cursor (409
`live-cursor-reset`) ends the tail with exit 1 — restart without `--cursor`,
or with `--from-now`; rows already written may be superseded. A dataset
version bump rebuilds every in-flight game's live log at the new version, so
a cursor from the old version ends the tail the same way. A poll whose
listing and segment bodies keep disagreeing prints a `note:` and polls again
with the cursor unchanged.

### Sports data revision tail

Use `betflux live-tail GAME --dataset game-state|plays|player-stats|team-stats`,
or `betflux live-tail --dataset injuries --league NFL` without `GAME`. The
default dataset remains `sportsbook-lines` for compatibility. Sports data output is
JSONL only: one complete object per revision, including zero-row revisions.

| Option | Sports data behavior |
|---|---|
| `--cursor CURSOR` | Resume after a snapshot or prior revision cursor |
| `--from-now` | Read the current snapshot cursor and emit later revisions only |
| `--every N` | Override the `poll_after` seconds the snapshot or changes listing advises (the default, a resumed tail included) |
| `--metric`, `--player-id`, `--team-id` | Local stat/injury filters where supported; filters may emit deletes when membership changes |
| `--availability` | Injury follower only |
| `--output PATH` | Append complete JSONL revisions and persist `<PATH>.cursor.json` after each synced line |

The same output file must resume with the same dataset, scope and filters.
Re-run the same command to resume; `--from-now` only chooses where a new output
starts and is ignored once a checkpoint exists, as is the `--cursor` the output
started from. A `--cursor` from another stream, or ahead of the checkpoint, is
refused.
An incomplete trailing line is discarded on managed resume. A `409`
(`stream-reset` problem) or `410` (`cursor-expired` problem) requires a new
snapshot; do not append another epoch to the old file.

Started without `--cursor` on a game or league with nothing published yet
(`404` `not-yet-available`; see [sports-data.md](sports-data.md)), the tail prints
`waiting: nothing published yet for <scope> <dataset> (…); checking again in
30s, Ctrl-C to stop` on stderr and retries every poll interval (30 s, 300 s for
injuries, or `--every`). Any other 404 — an unknown game, an uncovered league —
ends it with exit 1.

## Output notes

- Default table shows a curated column subset; the gold datasets are wide.
- Timestamp columns are real Parquet timestamps and render as ISO 8601 in every
  format.
- Progress renders on stderr, so `--format json > out.json` stays clean.

## Exit behavior

Failures print `Error: …` to stderr and exit 1. `BETFLUX_DEBUG=1` adds a
traceback. Ctrl-C exits 130, except in `live-tail`, where it is the normal way
to stop and exits 0. `get --dry-run` exits 0 after printing the preview; a
range pull the allowance cannot cover exits 1 before any download unless
`--allow-partial` is given. The access errors each have their own message:
429 `quota-exhausted` (`this game would be a new one … Games already read
this period stay free … Resets … Upgrade: …`), 403 `plan-excludes` (`your
plan does not include <feature>. Plans that include it: … Upgrade: …`) and
403 `history-restricted` (the trial sample rule and the upgrade link). None of
them is worth retrying on the same key.
