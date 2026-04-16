# Workflow Contract

**Contract version: 2**

This document defines the shared contract between `init-project` (which creates GitHub issues, labels, milestones, and project documentation) and `exec-tasks` (which reads them to select and execute work). Both plugins MUST conform to the current contract version.

Changes to this document that alter field names, template structure, label vocabulary, or semantics are **behavioral changes** and require:
1. A version bump of the header above
2. Synchronous updates to both `init-project/skills/init-project/SKILL.md` and `exec-tasks/skills/exec-tasks/SKILL.md` in the same PR
3. A note in the commit message describing the migration

---

## 1. Issue body template

Every issue created by `init-project` (and every issue that `exec-tasks` reads) MUST use the following Markdown structure exactly. Heading levels, field names, and order are fixed.

```markdown
## Summary
<one-sentence description of what this issue accomplishes>

## Acceptance Criteria
- <bullet 1>
- <bullet 2>
- <bullet 3>

## Priority
P<n>

## Depends on
<#N, #M, ... or "none">
```

**Notes:**
- The `Summary` section is a single line of prose, not a paragraph.
- `Acceptance Criteria` has 2–5 bullets. Each bullet is a testable outcome, not an implementation step.
- `Priority` contains exactly one value, `P0` through `P3`.
- `Depends on` lists issue numbers prefixed with `#`, comma-separated. If there are no dependencies, the field contains the literal word `none`.

## 2. Label vocabulary

The following labels MUST exist in any repository that `exec-tasks` operates on. `init-project` creates them during scaffold.

**Priority labels** (exactly one per issue):
- `P0`
- `P1`
- `P2`
- `P3`

**Type labels** (exactly one per issue):
- `type:feature`
- `type:bug`
- `type:chore`
- `type:docs`

No aliases, no case variants, no additional priority or type labels. Additional labels (area tags, status, etc.) are permitted but outside this contract — `exec-tasks` ignores them.

## 3. Milestone naming

Milestones MUST match the regex `^M\d+$` (e.g., `M1`, `M2`, `M3`, `M10`). Descriptions are free-form. Due dates are optional and not used by `exec-tasks`.

`exec-tasks` works on the earliest incomplete milestone first (lowest number that has any open issue).

## 4. Dependency syntax

Dependencies are expressed in the `Depends on` section of the issue body (see §1). The exact line format is:

```
<#N>
```

or for multiple dependencies:

```
<#N, #M, #K>
```

Other forms (`Blocked by #N`, `- depends: #N`, `depends-on: #N`) are NOT recognized. `exec-tasks` parses only the `#<number>` tokens on lines within the `Depends on` section.

An issue is **blocked** if any listed dependency references an open issue. `exec-tasks` skips blocked issues.

## 5. Priority sort order

Within a milestone, `exec-tasks` selects issues by:
1. Priority, ascending by number (`P0` > `P1` > `P2` > `P3`)
2. Issue number, ascending (older issues first)

Ties are broken deterministically by issue number. No timestamps.

## 6. Project documentation paths

`init-project` creates the following files and `exec-tasks` reads them for agent context. Paths are fixed relative to the project root.

| Path | Purpose | Required |
|---|---|---|
| `CLAUDE.md` | Project-level conventions | yes |
| `docs/PRD.md` | Product requirements document | yes |
| `docs/glossary.md` | Domain vocabulary | yes |
| `docs/adr/` | Architecture decision records directory | yes |
| `docs/adr/0001-record-architecture-decisions-in-adrs.md` | Meta-ADR | yes |
| `.memory/` | Long-form plans and specs directory | yes |
| `.memory/plans/` | Subdirectory for feature plans | yes |

When `exec-tasks` spawns a coding agent for an issue, it includes these files in the agent's context package.

## 7. `.memory/` structure

`init-project` creates `.memory/` and `.memory/plans/` as empty directories (preserved with `.gitkeep` files). Downstream skills (`plan-feature`, future tools) write content here. `exec-tasks` reads and writes `.memory/plans/*` per §8 below.

## 8. Plan documents

When `exec-tasks` executes an issue, it produces two files in `.memory/plans/` per issue. Both are committed with the implementation so that `review-impl` (and human reviewers) can cross-reference plan and result later.

### 8.1 Context package

**Path:** `.memory/plans/<issue_number>-context.md`

**Produced by:** the `prepare-context` sub-skill, invoked by the `exec-tasks` orchestrator before spawning a coding agent.

**Required sections** (in order):

1. Header with issue number, title, generation timestamp, and `Contract version: 2`
2. `## Issue summary` — verbatim from the issue body's Summary
3. `## Linked issues` — from grep of body and comments for issue references
4. `## Relevant ADRs` — ADRs whose decisions constrain this task
5. `## Relevant PRD sections` — extracted sections only, not the whole PRD
6. `## Glossary terms` — terms from the issue that are defined in `docs/glossary.md`
7. `## Symbol references in codebase` — grep results for key nouns in the issue title
8. `## Recent related PRs` — recently merged PRs that touched files hinted by the issue
9. `## Issue comments` — verbatim or summarized
10. `## Library documentation` — best-effort WebFetch of third-party dep docs (may be ⚠️ if failed)
11. `## Gathering notes` — any partial-failure records

Sections with no data MUST contain the literal string "None" — not placeholder content.

### 8.2 Plan document

**Path:** `.memory/plans/<issue_number>-plan.md`

**Produced by:** the `plan-task` sub-skill, invoked by the spawned coding agent as its first action, after reading the context package.

**Required sections** (in order):

1. Header with issue number, title, generation timestamp, `Contract version: 2`, and a link to the context package
2. `## Summary` — 2–3 sentences on what the task accomplishes and why
3. `## Approach` — numbered list of 5–10 implementation steps
4. `## Constraints` — ADR-imposed or convention-imposed constraints, referenced by ADR number. "None" if no constraints apply.
5. `## File manifest` — three subsections (created/modified/deleted), each listing paths with one-line purposes. "Deleted" subsection omitted if empty.
6. `## Data model` — type or schema definitions if this task changes data shapes. Otherwise the literal string "No data model changes."
7. `## System flow diagram` — one mermaid `flowchart` (or `sequenceDiagram`) code block, with a 1–2 sentence description below it. Every node that looks like a code symbol or file path MUST correspond to a real artifact that will exist after the task.
8. `## State model / Data model diagram` — one mermaid `stateDiagram-v2`, `classDiagram`, or `erDiagram` code block, with a 1–2 sentence description below it. Pick the diagram type that fits the task; do not produce placeholder diagrams.
9. `## Acceptance criteria` — **verbatim** copy from the issue body's Acceptance Criteria section, as an unchecked checkbox list.
10. `## Test plan` — for each acceptance criterion, a line mapping it to a specific test file and test case.
11. `## Open questions` — anything the agent is unsure about that a human should resolve before merge. "None" if empty.

**Validation rules enforced by `self-check`:**

- File manifest must match the diff (modulo lock files and generated files — see `self-check` SKILL.md for the full exemption list)
- Mermaid diagrams must reference only real files, functions, types, or components
- Every acceptance criterion must have a corresponding entry in the Test plan section, and the referenced test file must exist and be touched in this diff (unless explicitly marked `(pre-existing test, verified by this task)`)
- No TODO placeholders or empty required sections

### 8.3 Commit policy

Both `.memory/plans/<issue>-context.md` and `.memory/plans/<issue>-plan.md` are committed as part of the implementation PR. They are not gitignored. `review-impl` (future skill) reads them during retrospective, and human reviewers use them to understand the agent's intent.

---

## Changelog

- **v2 (2026-04-16):** Added §8 Plan documents. Specifies location (`.memory/plans/<issue>-context.md` and `<issue>-plan.md`), required sections, validation rules enforced by `self-check`, and commit policy. No breaking changes to §1–§7.
- **v1 (2026-04-16):** Initial contract. Established issue template, label vocabulary, milestone naming, dependency syntax, priority ordering, documentation paths, and `.memory/` structure.
