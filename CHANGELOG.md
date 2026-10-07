# Changelog

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
