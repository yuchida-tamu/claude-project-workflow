---
name: init-project
description: Interview the user about a new project, scaffold a contract-conforming repository (CLAUDE.md, PRD, ADRs, glossary, .memory/, GitHub Actions, hooks), then create the GitHub repo with labels, milestones, and seeded M1 issues. Produces output that `exec-tasks` can immediately consume.
version: 0.1.0
---

# Project Initializer

Bootstrap a new project that conforms to the [workflow contract](../../../docs/WORKFLOW_CONTRACT.md) (contract version 2). The output of this skill is a local scaffold plus a GitHub repository whose issues, labels, and milestones match exactly what `exec-tasks` expects to read.

## When to use

- When the user says `/init-project` with or without a name argument
- When the user asks to "scaffold a new project" or "bootstrap a repo with the workflow contract"

## Workflow Contract

Every artifact this skill writes MUST conform to [`docs/WORKFLOW_CONTRACT.md`](../../../docs/WORKFLOW_CONTRACT.md) (contract version 2). The load-bearing points:

- **Issue body template** has exactly `## Summary`, `## Acceptance Criteria`, `## Priority`, `## Depends on` sections (in that order)
- **Priority labels:** `P0`, `P1`, `P2`, `P3` (exactly one per issue)
- **Type labels:** `type:feature`, `type:bug`, `type:chore`, `type:docs` (exactly one per issue)
- **Milestones** match `^M\d+$` (e.g., `M1`, `M2`, `M3`)
- **Dependencies** live in the `## Depends on` section as `#N, #M` tokens, or the literal word `none`
- **Required project paths:** `CLAUDE.md`, `docs/PRD.md`, `docs/glossary.md`, `docs/adr/0001-record-architecture-decisions-in-adrs.md`, `.memory/plans/`

Any deviation from these names, formats, or paths is a contract violation. Do not improvise alternate spellings.

## Invocation & flags

- Slash command: `/init-project [name]`
- **Directory handling:**
  - If `name` is given: create `<cwd>/<name>/`, `cd` into it, scaffold there
  - If `name` is omitted: scaffold the current directory (must pass the empty-ish guard below)
- **Empty-ish guard.** Refuse to proceed if any of these already exist in the target directory: `CLAUDE.md`, `docs/`, `.memory/`, `.github/workflows/claude.yml`. Allow `.git/`, `.DS_Store`, and a pre-existing `README.md`. If the guard fails, print which files blocked the scaffold and stop.
- **Flags:**
  - `--skip-github` — stop after phase 2 (local files + git init + initial commit); no `gh` calls
  - `--no-hook` — do not install the `.claude/settings.json` PostToolUse hook
  - `--dry-run` — print the full plan (files + gh commands) and exit without writing anything or running git
  - `--resume` — read `.init-project-state.json` and continue from the first incomplete phase
  - `--name=<name>` — with `--resume`, allow overriding the target repo name (e.g., after a name collision on `gh repo create`)
  - `--license=mit|apache-2.0|none` — default `mit`

## Workflow

### Step 1: Interview

Ask these 13 questions **one at a time**, inline. Do not dump them all at once. Keep follow-ups short and only when an answer is ambiguous. If the user passes a `name` arg, skip Q1.

1. **Project name** (skip if passed as arg)
2. **One-sentence pitch:** what does it do and for whom?
3. **Platform/runtime:** mobile / web / CLI / library / service?
4. **Tech stack:** one line — language, framework, state, persistence, test runner, lint/format. Example: "TypeScript / React Native Expo / Zustand / SQLite / Jest / Biome". If Rust, also ask: binary or library?
5. **Core user loop:** the one action a user does most often.
6. **Explicit non-goals:** what this is NOT.
7. **Riskiest technical unknown:** what could kill this project?
8. **First-year milestones:** M1 (MVP), M2 (polish), M3 (ship) — one line each.
9. **First 3–5 concrete issues for M1.** For each, ask inline: issue title, 2–3 acceptance criteria bullets, priority (P0–P3), type (`feature`/`bug`/`chore`/`docs`), dependencies (comma-separated issue numbers of earlier issues in this list, or `none`).
10. **Any hard architectural constraints that should become ADR-0002?** (e.g., "offline-first", "monolith not microservices", "no external deps"). If the user says "none", skip ADR-0002.
11. **Domain vocabulary:** 3–5 key nouns that need glossary entries. For each: term and short definition.
12. **Any repo-specific conventions beyond the five principles?** (e.g., "no barrel exports", "type over interface", "semantic keys"). Free-form, optional.
13. **GitHub org/user and visibility.** Default org/user: `gh api user --jq .login`. Visibility: public or private.

#### grill-me escape hatch

If an answer feels underspecified (too short, vague, self-contradictory), offer exactly this:

> That answer sounds underspecified. Want me to dig deeper with **grill-me** — a companion skill that interviews you one decision at a time until every branch is resolved? (yes / no / skip this question)

`yes` → invoke the `grill-me` skill on that specific question, then resume.
`no` → proceed with the given answer.
`skip this question` → mark the answer as `TODO` in the generated file and move on.

### Step 2: Preview gate

Before writing anything, show the user this preview (adapt the file list and excerpts to the actual interview answers and flags):

```
Here's what I'll create:

📁 Files (N total):
  CLAUDE.md
  docs/PRD.md
  docs/glossary.md
  docs/adr/0001-record-architecture-decisions-in-adrs.md
  docs/adr/0002-<slug>.md           [only if Q10 provided]
  .memory/plans/.gitkeep
  .github/workflows/claude.yml
  .github/workflows/ci.yml
  .claude/settings.json              [unless --no-hook]
  .gitignore
  LICENSE                            [unless --license=none]
  README.md                          [unless pre-existing]

🔍 Key excerpts:
  PRD Overview:    "<one-liner from Q2>"
  ADR-0002:        "<Q10 decision>"   [if present]
  Glossary terms:  <comma-separated list from Q11>
  Issues:          #1 <title>, #2 <title>, ... [from Q9]

📡 GitHub operations:                [omit block if --skip-github]
  Create repo: <org>/<name> (public|private)
  Labels: P0, P1, P2, P3, type:feature, type:bug, type:chore, type:docs
  Milestones: M1, M2, M3
  Issues: <N> seeded in M1

Proceed? (y/N)
```

If the user answers `N`, ask:

> What was wrong? (1) Interview answers — restart interview. (2) Abort entirely. (3) Proceed anyway.

Handle each choice literally. `y`/`yes` proceeds to Step 3.

### Step 3: Phased execution

Phases execute in strict order. On any hard failure, **stop immediately**, preserve all local state, update `.init-project-state.json` with the list of completed phases and all interview answers, print the exact `gh` or shell command the user can run to finish the failed phase manually, and tell them they can re-run with `--resume` (optionally with `--name=<different-name>`).

#### Phase 1: Local files

Before writing anything else, write `.init-project-state.json` at the project root with:

```json
{
  "contract_version": 1,
  "target_name": "<name>",
  "target_org": "<org>",
  "visibility": "<public|private>",
  "flags": { "skip_github": false, "no_hook": false, "license": "mit" },
  "answers": { "q1": "...", "q2": "...", "...": "..." },
  "completed_phases": []
}
```

Then write every file listed under "Files to write" below. If `--dry-run`, print the file list plus the planned `gh` commands and exit without touching the filesystem or git.

After every file is written, append `"phase1_local_files"` to `completed_phases` and save the marker.

#### Phase 2: git init + initial commit

```bash
git init
git add .
git commit -m "Initial scaffold via /init-project"
```

Append `"phase2_git_init"` to `completed_phases`. If `--skip-github` is set, skip to "Successful completion" below. If `--dry-run`, do nothing here.

#### Phase 3: gh repo create

```bash
gh repo create <org>/<name> --source=. --push --<public|private>
```

On "already exists" error, print:

> Repo `<org>/<name>` already exists. Either delete it on GitHub, or re-run with `--resume --name=<different-name>` to retry with a different name.

Append `"phase3_gh_repo_create"` on success.

#### Phase 4: labels + milestones (parallel)

Run these two sub-steps as parallel tool calls in a single message:

**Labels.** For each of `P0`, `P1`, `P2`, `P3`, `type:feature`, `type:bug`, `type:chore`, `type:docs`:

```bash
gh label create "<label>" --repo <org>/<name> --description "<desc>" --color "<hex>" --force
```

Suggested colors: `P0`=`b60205`, `P1`=`d93f0b`, `P2`=`fbca04`, `P3`=`0e8a16`, `type:feature`=`1d76db`, `type:bug`=`d73a4a`, `type:chore`=`cfd3d7`, `type:docs`=`0075ca`.

**Milestones.** For each of `M1`, `M2`, `M3`, use `gh api` to POST to `/repos/<org>/<name>/milestones` with the title and the one-line description from Q8:

```bash
gh api -X POST /repos/<org>/<name>/milestones -f title="M1" -f description="<Q8 M1 line>"
```

Record the returned milestone number for each milestone. Append `"phase4_labels_milestones"` on success.

#### Phase 5: issues

For each of the 3–5 issues from Q9, create the issue with `gh issue create`, attaching the M1 milestone, the priority label, and the type label:

```bash
gh issue create --repo <org>/<name> \
  --title "<title>" \
  --body "$(cat <<'EOF'
## Summary
<one-sentence description>

## Acceptance Criteria
- <bullet 1>
- <bullet 2>
- <bullet 3>

## Priority
P<n>

## Depends on
<#N, #M or none>
EOF
)" \
  --label "P<n>" \
  --label "type:<type>" \
  --milestone "M1"
```

**Issue number resolution for dependencies.** GitHub assigns issue numbers in creation order. Create issues in the order the user listed them in Q9. When a later issue depends on an earlier one, substitute the actual issue number returned by `gh issue create` before writing the `Depends on` line.

Append `"phase5_issues"` on success.

#### Successful completion

Delete `.init-project-state.json` and print:

```
✓ Project scaffolded at <path>
✓ GitHub repo: https://github.com/<org>/<name>

To enable @claude PR review, run this once from inside this repo:
  /install-github-app

This installs the Claude GitHub App on the repo and configures the
CLAUDE_CODE_OAUTH_TOKEN secret. The .github/workflows/claude.yml file
this scaffold wrote is already aligned with what /install-github-app
expects, so when it asks to overwrite the workflow you can decline
(or accept — the contents are equivalent).

Next step:
  /plugin install exec-tasks@claude-project-workflow  (if not already installed)
  /exec-tasks    # picks up M1 issues and starts executing
```

(If `--skip-github` was set, omit the GitHub line and the `/install-github-app` block, and end at the `/exec-tasks` hint.)

### Step 4: Resume handling

When invoked with `--resume`:

1. Read `.init-project-state.json`. If missing, tell the user there is no resume state and stop.
2. Replay the answers (do not re-interview).
3. Skip every phase listed in `completed_phases`.
4. Start from the first incomplete phase and proceed normally.
5. If `--name=<new>` was passed, update `target_name` in the marker, and if phase 3 is the first incomplete phase, use the new name when running `gh repo create`.

## Files to write

All paths are relative to the project root being scaffolded. Every file is mandatory unless marked optional.

### `CLAUDE.md`

Use this exact spine (copy verbatim, then fill placeholders) and append the generated sections after it:

````markdown
# {{PROJECT_NAME}}

{{ONE_LINE_PITCH}}

## Principles

### Think Before Coding
- Read the relevant source files before changing anything. Understand the current state — don't assume from memory or naming alone.
- Check `.memory/` and `docs/adr/` for existing design decisions. If a decision exists, follow it or propose a revision — don't silently diverge.
- Use `/plan-feature` for any feature touching multiple files. Skip planning only for isolated bug fixes or single-file changes.

### Simplicity First
- Solve the problem in front of you. Don't add abstractions, helpers, or config for hypothetical futures.
- Three similar lines beats a premature function. A flat `if` beats a strategy pattern.
- The codebase is small. Grep and read before inventing — the utility you need may already exist.

### Surgical Changes
- Touch only the files that need to change. A bug fix is not a refactoring opportunity.
- Don't reformat, rename, or reorganize code adjacent to your change.
- When adding a feature, match the patterns already in use. Don't introduce new patterns without an ADR.

### Separate Structural from Behavioral Changes (Tidy First)
- **Never mix structural and behavioral changes in the same PR.** Structural = renames, extractions, moves, reformatting, reorganizing imports, splitting files — changes that preserve behavior. Behavioral = anything that alters what the code *does* (new logic, bug fixes, formula tweaks, new actions).
- **Order:** if a change needs both, land the structural PR first (empty-diff in behavior, green tests), then the behavioral PR on top. This makes behavioral diffs small and reviewable, and keeps `git blame` honest.
- **One PR, one kind.** If you notice midway that you're doing both, stop and split the branch. Do not rationalize "it's small, I'll bundle it."
- **Red flag in review:** a PR titled as a bug fix with renamed symbols, moved files, or reformatted blocks. Call it out and request a split.
- Applies to commits within a PR too — prefer separate commits for structural vs behavioral work even when they ship together.

### Goal-Driven Execution
- Every change must pass `{{CHECK_COMMAND}}` ({{CHECK_DESCRIPTION}}). The post-edit hook enforces this automatically.
- Ship through PRs. Never push to `main`. Branch → implement → PR → review → merge.
- Present the PR link to the user after creating it. Wait for their review before merging.
````

**Do not paraphrase the spine.** Copy it verbatim into every generated CLAUDE.md, substituting only `{{PROJECT_NAME}}`, `{{ONE_LINE_PITCH}}`, `{{CHECK_COMMAND}}`, and `{{CHECK_DESCRIPTION}}`.

After the spine, append these generated sections:

- `## Architecture` — from Q3 (platform) + Q4 (stack). If Q5 gave enough detail for a state-flow diagram, include a short one; otherwise write `TODO: document core state flow`.
- `## Tech Stack` — bulleted list from Q4 with rows Runtime / Language / Framework / State / Persistence / Test / Lint
- `## Commands` — derived from Q4 tech stack:
  - **Node/TypeScript:** `npm run check`, `npm test`, `npm run typecheck`, `npm run lint`
  - **Rust:** `cargo check`, `cargo test`, `cargo clippy`, `cargo fmt`
  - **Python:** `pytest`, `ruff check`, `mypy`
  - **Go:** `go build ./...`, `go test ./...`, `go vet ./...`
  - **Unknown:** `./scripts/check.sh  # TODO: fill in`
- `## Conventions` — from Q12 (free-form) plus the universal lines: "Ship through PRs. Never push to main. Record significant decisions in `docs/adr/`."
- `## Workflow` — template, exactly 7 steps:
  1. Branch from `main`
  2. Consult `.memory/` and `docs/adr/` for relevant specs
  3. Implement
  4. Run the check command (must pass)
  5. Push branch, create PR via `gh pr create`
  6. Wait for review
  7. On merge, switch to `main`, pull, delete local branch

The `{{CHECK_COMMAND}}` and `{{CHECK_DESCRIPTION}}` substitutions for the Principles spine come from the same tech-stack mapping above. For Node use `npm run check` / "typecheck + lint + test". For Rust use `cargo check && cargo test && cargo clippy` / "check + test + clippy". For Python use `pytest && ruff check && mypy` / "tests + lint + types". For Go use `go build ./... && go test ./... && go vet ./...` / "build + test + vet". For Unknown use `./scripts/check.sh` / "project check".

### `docs/PRD.md`

Skeleton with these sections, populated where possible from the interview:

```markdown
# <Project Name> — PRD

## Overview
<Q2 one-sentence pitch, expanded to 1–3 sentences if the user said more>

## Goals
TODO: list concrete success criteria

## Non-Goals
- <Q6 bullet 1>
- <Q6 bullet 2>
...

## Users
TODO: describe target user

## Core Loop
<Q5 core user loop, prose>

## Milestones
- **M1 (MVP):** <Q8 M1 line>
- **M2 (Polish):** <Q8 M2 line>
- **M3 (Ship):** <Q8 M3 line>

## Open Questions
- <Q7 riskiest unknown>
```

### `docs/glossary.md`

```markdown
# Glossary

<For each term from Q11:>
- **<term>** — <definition>
```

If Q11 was empty, seed with `TODO: add domain vocabulary`.

### `docs/adr/0001-record-architecture-decisions-in-adrs.md`

Always written. Standard MADR template:

```markdown
# 1. Record architecture decisions in ADRs

Date: <YYYY-MM-DD>

## Status
Accepted

## Context
We need a lightweight, version-controlled way to record significant architectural decisions so future contributors (human or agent) can understand the reasoning behind the current shape of the codebase.

## Decision
We will record architecture decisions here using the ADR format (Markdown Architectural Decision Records). Each ADR lives in `docs/adr/` and is numbered sequentially.

## Consequences
- Every non-obvious architectural decision gets a short written rationale.
- Agents and reviewers can grep `docs/adr/` before proposing divergent patterns.
- New decisions require a new ADR; supersessions are recorded explicitly.
```

### `docs/adr/0002-<slug>.md`

**Only if Q10 was non-empty.** Slug is derived from the decision title, kebab-cased. Same MADR template (Status, Context, Decision, Consequences) with the Q10 answer as the decision.

### `.memory/plans/.gitkeep`

Empty file to preserve the directory.

### `.github/workflows/claude.yml`

Write this file verbatim. This matches the workflow that Claude Code's `/install-github-app` slash command produces, so a user who later runs `/install-github-app` against the scaffolded repo will not be prompted to overwrite this file.

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code
        id: claude
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          additional_permissions: |
            actions: read
```

Authentication uses `CLAUDE_CODE_OAUTH_TOKEN` (a Claude.ai OAuth token), not `ANTHROPIC_API_KEY`. The token is set up by `/install-github-app` as part of the GitHub App installation flow — see "Successful completion" below.

### `.github/workflows/ci.yml`

Stub that runs the check command derived from Q4. Triggers: `push` and `pull_request` on `main`. Pick the runner and setup steps from the tech stack mapping:

- **Node/TypeScript:** `actions/setup-node@v4` with `node-version: 20`, `npm ci`, `npm run check`
- **Rust:** `dtolnay/rust-toolchain@stable`, `cargo check && cargo test && cargo clippy`
- **Python:** `actions/setup-python@v5` with `python-version: '3.12'`, `pip install -r requirements.txt`, `pytest && ruff check && mypy .`
- **Go:** `actions/setup-go@v5` with `go-version: '1.22'`, `go build ./... && go test ./... && go vet ./...`
- **Unknown:** single step `run: ./scripts/check.sh`

### `.claude/settings.json`

Skip if `--no-hook`. Otherwise write this JSON, substituting `{{CHECK_COMMAND}}`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "{{CHECK_COMMAND}}"
          }
        ]
      }
    ]
  }
}
```

For unknown tech stacks, set `{{CHECK_COMMAND}}` to `./scripts/check.sh` and also create `scripts/check.sh` with:

```sh
#!/bin/sh
# TODO: add project check command
exit 0
```

Then `chmod +x scripts/check.sh`.

### `.gitignore`

Universal base, always included:

```
.DS_Store
*.log
.env
.env.*
.idea/
.vscode/*
!.vscode/settings.json
!.vscode/extensions.json
*.swp
.claude/settings.local.json
.init-project-state.json
```

Plus a stack-specific block appended below, chosen from Q4:

- **Node/TypeScript:** `node_modules/`, `dist/`, `build/`, `coverage/`, `.next/`, `.expo/`, `*.tsbuildinfo`
- **Rust:** `target/`. Add `Cargo.lock` only if Q4 says it is a **library**, not a binary.
- **Python:** `__pycache__/`, `*.pyc`, `.venv/`, `venv/`, `.pytest_cache/`, `.mypy_cache/`, `dist/`, `build/`, `*.egg-info/`
- **Go:** `bin/`, `*.exe`, `*.test`, `*.out`, `vendor/`
- **Unknown:** `# TODO: add stack-specific ignores`

### `LICENSE`

- `--license=mit` (default): MIT template with `<year>` = current year and `<author>` = `gh api user --jq .name`, falling back to `gh api user --jq .login` if `.name` is empty.
- `--license=apache-2.0`: standard Apache-2.0 text.
- `--license=none`: skip the file entirely.

### `README.md`

**Do not overwrite** if the file already exists. Otherwise write:

```markdown
# <Project Name>

<Q2 one-line pitch>

## Getting started

TODO: install and run instructions
```

### `.init-project-state.json`

Written at the start of Phase 1 (see above). Deleted on successful completion of the final applicable phase.

## Rules

- **Contract conformance is mandatory.** Every issue body, label name, milestone name, and required path must match `docs/WORKFLOW_CONTRACT.md` v2 exactly. Do not improvise spellings or orderings.
- **Never overwrite existing files.** The empty-ish guard (CLAUDE.md, docs/, .memory/, .github/workflows/claude.yml) is a hard stop. `README.md` is the single exception — if it already exists, leave it untouched.
- **Always write the state marker first.** `.init-project-state.json` must exist before any other file in Phase 1 so a crash mid-scaffold is recoverable with `--resume`.
- **Resume respects completed phases.** Never re-run a phase listed in `completed_phases`. Never re-interview; replay answers from the marker.
- **Hard-fail preserves state.** On any phase failure, stop immediately, update the marker, and print exact remediation commands + a `--resume` hint. Do not roll back.
- **Spine is verbatim.** The CLAUDE.md Principles section is copied character-for-character. Only the four placeholders (`PROJECT_NAME`, `ONE_LINE_PITCH`, `CHECK_COMMAND`, `CHECK_DESCRIPTION`) may be substituted.
- **Interview one question at a time.** Do not batch the 13 questions into a single prompt.
- **Offer grill-me only on underspecified answers.** Do not offer it as a default; it is an escape hatch, not a greeting.
- **Dry-run writes nothing.** `--dry-run` must not touch the filesystem, must not run git, and must not make any `gh` calls. It only prints the plan.
- **Parallel labels + milestones.** In Phase 4, issue the label creation and milestone creation `gh` calls in a single message with parallel tool calls. Issue creation in Phase 5 is strictly sequential because later issues may reference earlier issue numbers in `Depends on`.
- **Tell the user to run `/install-github-app` at the end.** Do not attempt to install the GitHub App or set the OAuth secret yourself — `/install-github-app` is interactive (OAuth flow) and the user must run it. The scaffold's `claude.yml` is intentionally identical to what `/install-github-app` produces so the user can decline the overwrite prompt.
- **Final next-step hint points at `/exec-tasks`.** The two plugins are designed to hand off; always tell the user what comes next.
