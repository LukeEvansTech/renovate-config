# Findings: `.renovaterc.json5` fails super-linter `JSONC_PRETTIER`

**Date:** 2026-06-18
**Status:** Confirmed + fixed in `codelooks-com/packer-vsphere` (#72)
**Applies to:** every repo that migrated `renovate.json` → `.renovaterc.json5`
for fleet consistency and is linted by the shared super-linter workflow
(`LukeEvansTech/shared-workflows/.github/workflows/super-linter.yml`).

## Symptom

The **Lint** workflow fails on `main` (and on PRs) with:

```
##[group]JSONC_PRETTIER
[ERROR]   Errors found in JSONC_PRETTIER
[ERROR]   Stderr contents for JSONC_PRETTIER:
[warn] .renovaterc.json5
[warn] Code style issues found in the above file. Run Prettier with --write to fix.
[ERROR]   Super-linter detected linting errors
```

Everything else in the lint job passes — only the prettier check on the
renamed Renovate config trips.

## Root cause

When `renovate.json` was renamed to `.renovaterc.json5`, the **file contents
were left in standard-JSON layout** (double-quoted keys, no trailing comma):

```json5
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>LukeEvansTech/renovate-config"]
}
```

super-linter groups the file under its `JSONC` language and runs **Prettier**
on it. Prettier picks its parser from the file **extension**, and for `.json5`
that is the **`json5` parser**. The json5 parser's canonical style is:

- **unquoted** object keys (`$schema:` not `"$schema":`)
- a **trailing comma** after the last property

So Prettier reformats the file and `--check` fails. The fix the linter is
asking for is purely cosmetic — the JSON is semantically identical and Renovate
accepts both forms.

> The group is named `JSONC_PRETTIER`, which is misleading: it's super-linter's
> *language grouping*, not the Prettier *parser*. The actual parser is `json5`,
> inferred from the extension. Do **not** "fix" this by forcing
> `--parser jsonc` — that keeps quoted keys and only adds the trailing comma,
> which diverges from the rest of the fleet.

## The fix (canonical house format)

Run Prettier's auto-fix, or hand-edit to match:

```bash
npx --yes prettier@3 --write .renovaterc.json5
npx --yes prettier@3 --check .renovaterc.json5   # must say "All matched files use Prettier code style!"
```

Canonical result:

```json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: ["github>LukeEvansTech/renovate-config"],
}
```

This matches the already-passing fleet repos (`LukeEvansTech/talos-cluster`,
`LukeEvansTech/renovate-config`) and the example in this repo's `README.md`.

## How to detect across all repos

For each repo, one local check (no Docker / no super-linter needed — Prettier
default inference reproduces it exactly):

```bash
# in a repo checkout, if it has a .renovaterc.json5:
[ -f .renovaterc.json5 ] && npx --yes prettier@3 --check .renovaterc.json5
```

Fleet-wide sweep (adjust the org/list to taste), read-only, prints repos that
need the fix:

```bash
for repo in $(gh repo list LukeEvansTech --json nameWithOwner --jq '.[].nameWithOwner'); do
  body=$(gh api "repos/$repo/contents/.renovaterc.json5" --jq '.content' 2>/dev/null | base64 -d 2>/dev/null) || continue
  printf '%s' "$body" | npx --yes prettier@3 --parser json5 --check --stdin-filepath .renovaterc.json5 >/dev/null 2>&1 \
    || echo "NEEDS FIX: $repo"
done
```

## Known affected / to verify

Observed on 2026-06-18 via the GitHub API:

| Repo | `.renovaterc.json5` style | Verdict |
|------|---------------------------|---------|
| `codelooks-com/packer-vsphere` | was standard-JSON | **fixed** (#72) |
| `LukeEvansTech/talos-cluster` | json5 (unquoted, trailing comma) | OK |
| `LukeEvansTech/renovate-config` | json5 (unquoted, trailing comma) | OK |
| `LukeEvansTech/shared-workflows` | **standard-JSON** (quoted keys, no trailing comma) | **likely fails if linted — verify & fix** |

`shared-workflows` carries the old standard-JSON layout in its own
`.renovaterc.json5`. If that repo runs super-linter over itself it will hit the
same `JSONC_PRETTIER` failure — confirm and reformat with the one-liner above.

## Prevention

- When renaming `renovate.json` → `.renovaterc.json5`, **always run
  `prettier --write` on the result** in the same change, not just `git mv`.
- Better: copy the canonical 4-line block from this repo's `README.md` verbatim
  instead of carrying the old file's contents forward.
- A repo-local `.pre-commit` / editor Prettier integration catches it before it
  reaches CI.

## Portable prompt (for applying across repos)

> Check whether this repo has a `.renovaterc.json5`. If it does, run
> `npx prettier@3 --check .renovaterc.json5`. If it fails, run
> `npx prettier@3 --write .renovaterc.json5` — Prettier reformats it to the
> json5 style (unquoted keys + trailing comma) that super-linter's
> `JSONC_PRETTIER` expects (the parser is inferred from the `.json5`
> extension). Confirm `--check` passes, then commit as
> `fix(renovate): format .renovaterc.json5 to prettier json5 style`. The change
> is cosmetic; Renovate behavior is unchanged.
