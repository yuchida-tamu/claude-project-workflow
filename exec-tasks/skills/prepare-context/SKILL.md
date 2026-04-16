---
name: prepare-context
description: Gather codebase, GitHub, and library-documentation context for a single GitHub issue, then write a structured context package to .memory/plans/<issue#>-context.md for downstream skills (plan-task, coding agent). Invoked by the exec-tasks orchestrator before spawning a coding agent.
version: 0.1.0
---

# Prepare Context

Gather everything a coding agent needs to execute a GitHub issue, and write it to a single context-package file on disk. This skill is invoked by the `exec-tasks` orchestrator in Step 3 (Gather context) before spawning a coding agent.

## When to use

- Invoked programmatically by `exec-tasks` once per selected issue, before Phase 4 (spawn agents).
- Not a user-facing slash command. No frontmatter command wrapper.

## Inputs

- `issue_number` — the GitHub issue number this context package is for
- `repo` — `<owner>/<name>`, determined from `gh repo view --json nameWithOwner`

## Output

A single file at `.memory/plans/<issue_number>-context.md` with the structure defined in `docs/WORKFLOW_CONTRACT.md` §8.

If `.memory/plans/` does not exist, create it.

## Workflow

Run these three gathering steps. Each step is best-effort: if a sub-step fails (network error, missing file, rate limit), record the failure in the output file and keep going. Do NOT abort the whole skill on a partial failure.

### Step 1: Read the issue

```bash
gh issue view <issue_number> --repo <repo> --json number,title,body,labels,milestone,comments
```

Parse:
- **Linked issues** — grep the issue body and all comments for `#\d+` tokens and `Fixes #N`, `Related: #N`, `Depends on #N`, `Closes #N` patterns. Deduplicate.
- **Third-party dependencies** — scan the body and acceptance criteria for package names, library names, or doc-URL patterns. Heuristic: words matching `@[\w-]+/[\w-]+` (scoped npm), or words that appear in the project's dependency manifest (`package.json`, `Cargo.toml`, `requirements.txt`, `go.mod` — whichever exists).

### Step 2: Gather codebase context

Read in this order:

1. `CLAUDE.md` — conventions, check command, architecture overview
2. `docs/PRD.md` — locate and extract the section most relevant to the issue (search for keywords from the issue title)
3. `docs/glossary.md` — extract terms that appear in the issue body or acceptance criteria
4. `docs/adr/` — read every file, extract ADRs whose titles or decisions are relevant to the issue
5. **Symbol grep** — for each key noun in the issue title (excluding glossary stopwords like "user," "system"), run `Grep` with that symbol as the pattern. Record the top 5 files with the most matches per symbol.
6. **File manifest hint** — if the issue body mentions specific file paths, read them.

Record each input with a heading and the raw relevant content, trimmed to 20 lines max per source.

### Step 3: Gather GitHub context

Three sub-queries:

1. **Linked issues** — for each issue number found in Step 1, fetch its title, state, and body summary:
   ```bash
   gh issue view <N> --repo <repo> --json number,title,state,body
   ```
   Record a one-line summary per linked issue.

2. **Recent closed PRs touching relevant files** — if the issue body hints at file paths, find recently merged PRs that touched them:
   ```bash
   gh pr list --repo <repo> --state merged --limit 20 --json number,title,mergedAt,files
   ```
   Filter to PRs whose `files` overlap with the issue's hinted paths. Record up to 5.

3. **Comments on this issue** — any comments already on the issue (captured in Step 1). Summarize if >3 comments, otherwise include verbatim.

### Step 4: Fetch library documentation (best-effort)

For each third-party dependency identified in Step 1:

1. Construct a likely docs URL:
   - npm packages: `https://www.npmjs.com/package/<name>` (for overview)
   - scoped packages: `https://www.npmjs.com/package/@<scope>/<name>`
   - If the issue body contains an explicit doc URL, use that instead
2. `WebFetch` the URL with a prompt like "Extract the public API surface, installation instructions, and any sections relevant to: <issue title>."
3. If fetch fails (timeout, 404, rate limit, offline), record `⚠️ Could not fetch docs for <name>: <reason>` and move on.
4. Cap at 3 library doc fetches per issue — if the issue references more than 3 deps, include the top 3 by frequency and note the rest as "not fetched."

### Step 5: Write the context package

Write `.memory/plans/<issue_number>-context.md` with this exact structure:

```markdown
# Context package for issue #<N>: <title>

**Generated:** <ISO-8601 timestamp>
**Contract version:** 2

## Issue summary
<issue body Summary section, verbatim>

## Linked issues
- #<N> <title> — <state> — <one-line summary>
- ...
(or "None" if no links)

## Relevant ADRs
### ADR-<N>: <title>
<decision + rationale, trimmed to 10 lines>

(or "None" if no relevant ADRs)

## Relevant PRD sections
<extracted section(s) from docs/PRD.md>

## Glossary terms
- **<term>**: <definition>
- ...

## Symbol references in codebase
### `<symbol>`
- `path/to/file.ext` (N matches)
- ...

## Recent related PRs
- #<N> <title> (merged <date>) — touched files: <list>
- ...
(or "None")

## Issue comments
<comments verbatim or summarized>

## Library documentation
### <package name>
<extracted API surface + install instructions>

### <package name>
⚠️ Could not fetch docs: <reason>

## Gathering notes
- <any step failures or partial data notes>
```

## Rules

- **Never fail silently on partial data.** Every failure must be recorded in the output file under "Gathering notes" or inline with a ⚠️ marker. The caller (`exec-tasks` orchestrator) must be able to see what was and wasn't gathered.
- **Never invent content.** If a section has no data (no linked issues, no ADRs, no library deps), write "None" explicitly — do not write plausible-looking placeholder content.
- **Best-effort library fetches.** Library docs are the most fragile input. Never abort the skill on a failed WebFetch.
- **Do not read the full PRD.** Extract only the relevant section(s). Coding agents get overwhelmed by unfiltered context.
- **Do not modify the issue or post comments.** This skill is read-only on GitHub and write-only to `.memory/plans/`.
- **File size cap.** Context package must not exceed ~1000 lines. If it does, trim the lowest-priority sections (symbol references first, then closed PRs, then library docs).
