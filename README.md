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

**Automerge is broad, and gated on CI.** Every `minor`, `patch`, `digest` and
`pin` auto-merges once checks are green, as a CI-gated PR merge
(`automergeType: "pr"`, `platformAutomerge: false`, `ignoreTests: false`).
`major` updates never match an automerge rule and additionally queue behind a
checkbox on each repo's Dependency Dashboard, so they stay fully manual.

Because `platformAutomerge` is `false`, Renovate self-gates on CI and
self-merges — no branch-protection ruleset is required, and a repo with no CI
checks at all never auto-merges.

## Rolling this out to a new repo

The model is a **denylist**: the preset auto-merges everything on green, and each
repo names only what must stay human-reviewed.

```json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: ["github>LukeEvansTech/renovate-config"],

  packageRules: [
    {
      description: "Protected infra — never auto-merge, manual review only",
      // Must list every datasource the preset auto-merges, or a tier-0 component
      // tracked through another one slips past the guard entirely.
      matchDatasources: ["docker", "helm", "github-releases"],
      matchPackageNames: ["/cilium/", "/cert-manager/", "…"],
      automerge: false,
    },
  ],
}
```

That is the whole per-repo config, plus anything genuinely local: custom
datasources, custom managers, groups, labels, `ignorePaths`.

## The one rule: never re-state a preset setting

`packageRules` are **last-match-wins**, and preset rules are evaluated **first**.
So restating a setting locally silently overrides the shared decision — no error,
no warning, and nothing in a diff to notice. Adding `automerge: false` for your
own packages is correct; re-declaring _how_ auto-merge works is not.

Both production incidents in `LukeEvansTech/talos-cluster` were this mistake:

| Local restatement                               | What it reverted                 | Result                                                                                                                                                            |
| ----------------------------------------------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimumReleaseAge: "2 days"` on a docker rule  | the preset's docker soak opt-out | GHCR publishes no release timestamp, so the gate never turned green. **38 of 39 open PRs stuck**, one for five days against a two-day gate ([talos-cluster#3866]) |
| `automergeType: "branch"` + `ignoreTests: true` | the preset's CI-gated PR merge   | branch-merge bypasses the PR, so `claude/renovate-review` posted its verdict **76–93s after** the commit was already on `main` ([talos-cluster#3871])             |

Both looked healthy from the outside. The first kept merging digest and Docker
Hub bumps, which have a working timestamp, so only GHCR versioned bumps stalled.
The second ran the review on every PR and passed it every time — it just could
never block anything.

If house behaviour needs to change, **change it here**. Every repo picks it up.

### Two traps worth knowing

- **`minimumReleaseAge` is a permanent block on GHCR/quay**, not a soak. Those
  registries expose no reliable release timestamp, so the gate never turns green.
  The preset opts the `docker` datasource out for this reason; `helm` keeps the
  soak because `index.yaml` carries a real `created` timestamp that expires.
- **`automergeType: "branch"` cannot be gated by any check**, including an AI
  review, because it never opens a PR. It is also incompatible with a
  `pull_request` branch-protection rule, failing with _"Unknown error when
  attempting branch automerge"_ — so it buys nothing.

[talos-cluster#3866]: https://github.com/LukeEvansTech/talos-cluster/pull/3866
[talos-cluster#3871]: https://github.com/LukeEvansTech/talos-cluster/pull/3871

## How updates flow

- **This repo → home-operations:** the `home-operations/renovate-presets` pin in
  `default.json` is auto-bumped by the self-update `customManager` in
  [`.renovaterc.json5`](./.renovaterc.json5). Renovate opens a PR here whenever a
  new release ships.
- **Consumer repos → this repo:** they extend `github>LukeEvansTech/renovate-config`
  (tracking `main`), so they pick up changes here automatically with nothing to bump.
