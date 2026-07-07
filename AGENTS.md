# renovate-config

Shared [Renovate](https://docs.renovatebot.com) configuration, extended by other repos via `github>LukeEvansTech/renovate-config`, so house defaults (branch-based automerge, digest pinning, London timezone, Copilot review, narrow automerge scope) roll out to consumer repos automatically.

Status: active (last commit 2026-06-23).

## Key files

- `.renovaterc.json5` — the config itself: layers house defaults on top of the `home-operations/renovate-presets` bundle.
- `default.json` — the default preset consumer repos extend; pins the `home-operations/renovate-presets` version, auto-bumped by a self-update customManager.
- `README.md` — usage instructions for consumer repos and how updates flow both directions (this repo ↔ home-operations upstream, this repo ↔ consumer repos).
- `FINDINGS-json5-prettier.md` — investigation notes on a JSON5/prettier/super-linter CI failure across the repo fleet, and its resolution.

## Notes

No build/test — this is Renovate config only, validated by Renovate itself (and previously by super-linter, per the findings doc) when it runs against consumer repos.
