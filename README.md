# renovate-config

My shared [Renovate](https://docs.renovatebot.com) configuration. One place to
define house defaults so every repo behaves consistently and improvements roll
out automatically.

## Usage

In any repo, create `.renovaterc.json5` (or `renovate.json`) with:

```json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: ["github>LukeEvansTech/renovate-config"],
}
```

Add repo-specific settings (custom `packageRules`, `ignorePaths`, `schedule`,
non-standard manager file patterns, …) below the `extends` line as needed.

## What it does

It layers house defaults on top of the
[`home-operations/renovate-presets`](https://github.com/home-operations/renovate-presets)
bundle. From that base you get:

- `config:recommended`, `docker:enableMajor`
- GitHub Action digests pinned to semver (`helpers:pinGitHubActionDigestsToSemver`)
- Branch-based automerge, dependency dashboard, no rate limiting
- The `# renovate:` annotated manager plus OCI / CNPG / Talos / Grafana custom managers
- A semantic-commit policy and registry aliases

House additions on top:

- `docker:pinDigests` — also pin container image digests
- `:timezone(Europe/London)`
- GitHub Copilot requested as reviewer on every PR
- `dependencies` label, security labelling for vulnerability alerts
- Quieter dashboard notifications

**Automerge is deliberately narrow** — only GitHub Actions minor/patch/digest
updates automerge. Everything else opens a PR to review.

## How updates flow

- **This repo → home-operations:** the `home-operations/renovate-presets` pin in
  `default.json5` is auto-bumped by the self-update `customManager` in
  [`.renovaterc.json5`](./.renovaterc.json5). Renovate opens a PR here whenever a
  new release ships.
- **Consumer repos → this repo:** they extend `github>LukeEvansTech/renovate-config`
  (tracking `main`), so they pick up changes here automatically with nothing to bump.
