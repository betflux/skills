# Changelog

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
