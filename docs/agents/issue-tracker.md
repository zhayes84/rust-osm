# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues. Use the `gh` CLI for all operations.

## Conventions

- Create: `gh issue create --title "..." --body "..."`
- Read: `gh issue view <number> --json number,title,body,labels,comments`
- List: `gh issue list --state open --json number,title,body,labels,comments`
- Comment: `gh issue comment <number> --body "..."`
- Apply or remove labels: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- Close: `gh issue close <number> --comment "..."`

Infer the repository from `git remote -v`; `gh` does this automatically inside this clone.

## GitHub CLI authentication

`gh` uses the directory in `GH_CONFIG_DIR` when that variable is set; otherwise it uses `%APPDATA%\GitHub CLI`. In this environment, `GH_CONFIG_DIR` can point at another project's config and mask the authenticated Windows config. If `gh auth status` says you're not logged in, check `$env:GH_CONFIG_DIR` and clear the override for the current PowerShell session with `Remove-Item Env:GH_CONFIG_DIR`, or set it to the normal config directory with `$env:GH_CONFIG_DIR = Join-Path $env:APPDATA 'GitHub CLI'`. Then rerun `gh auth status`.

## Pull requests as a triage surface

**PRs as a request surface: no.** Set to `yes` if this repo treats external PRs as feature requests.

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

Read it using `gh issue view`.

## Wayfinding operations

Use a single issue labelled `wayfinder:map` as the map and GitHub sub-issues as child tickets. Add dependencies with GitHub's native issue dependencies when available; otherwise record `Blocked by: #<n>` in the child body. Claim a ticket with `gh issue edit <n> --add-assignee @me`.
