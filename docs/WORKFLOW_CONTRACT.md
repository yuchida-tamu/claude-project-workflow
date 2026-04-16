# Workflow Contract

**Contract version: 1**

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

`init-project` creates `.memory/` and `.memory/plans/` as empty directories (preserved with `.gitkeep` files). Downstream skills (`plan-feature`, future tools) write content here. `exec-tasks` reads `.memory/plans/*` for any existing specs relevant to a task.

---

## Changelog

- **v1 (2026-04-16):** Initial contract. Established issue template, label vocabulary, milestone naming, dependency syntax, priority ordering, documentation paths, and `.memory/` structure.
