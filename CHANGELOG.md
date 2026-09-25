# Changelog

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
