# claude-project-workflow

A Claude Code marketplace for the **design → plan → execute → review** project workflow. Ships plugins that work together: scaffold a new project, plan features, execute issues from a GitHub board, and review the results.

## What this repo is

This is a plugin monorepo, not a software project. Each top-level plugin directory (`init-project/`, `exec-tasks/`, etc.) is a self-contained Claude Code plugin. The marketplace manifest at `.claude-plugin/marketplace.json` registers them.

There are no build or test commands at the repo level. The work here is editing declarative markdown (SKILL.md, commands) and JSON manifests.

## Principles

### Think Before Coding
- Read the relevant files before changing anything. Understand current state — don't assume from memory or naming alone.
- Consult `docs/WORKFLOW_CONTRACT.md` before modifying any skill that reads or writes GitHub issues. The contract is the shared vocabulary between plugins and must stay in sync.

### Simplicity First
- Solve the problem in front of you. Don't add abstractions for hypothetical futures.
- Three similar lines beats a premature helper. A flat list beats a nested config.
- Each plugin is self-contained. Don't cross-reference skills from other plugins — if coordination is needed, do it through the workflow contract.

### Surgical Changes
- Touch only the files that need to change. A doc fix is not a refactoring opportunity.
- Don't rename or reorganize adjacent code while fixing something else.
- When adding a plugin, match the layout used by existing plugins. Don't introduce new patterns without a reason recorded in the commit.

### Separate Structural from Behavioral Changes (Tidy First)
- **Never mix structural and behavioral changes in the same PR.** Structural = renames, moves, reformatting, splitting files — changes that preserve behavior. Behavioral = anything that alters what the skill *does* (new questions in the interview, new GitHub operations, changed contract format).
- **Order:** if a change needs both, land the structural PR first (green, no behavior change), then the behavioral PR on top. This keeps behavioral diffs small and `git blame` honest.
- **One PR, one kind.** If you notice midway that you're doing both, stop and split the branch.
- **Contract changes are behavioral.** Any edit to `docs/WORKFLOW_CONTRACT.md` that changes field names, label names, or issue template structure is a breaking behavioral change and requires a contract version bump.

### Goal-Driven Execution
- Ship through PRs. Never push to `main`.
- Every PR should leave the marketplace in a working state — a user who installs from `main` at any commit should get plugins that load cleanly.

## Repository Structure

```
.claude-plugin/marketplace.json   # Marketplace registry for all plugins
docs/
  WORKFLOW_CONTRACT.md            # Shared contract between init-project and exec-tasks
  adr/                            # Architecture decision records (when created)
<plugin-name>/
  .claude-plugin/plugin.json      # Plugin metadata
  skills/<plugin-name>/SKILL.md   # Skill definition
  commands/<command>.md           # Slash command (if applicable)
  README.md                       # Bilingual EN/JP plugin docs
```

## The four-skill workflow

| Phase | Skill | Status |
|---|---|---|
| Design | `init-project` | implemented |
| Plan | `plan-feature` | planned |
| Execute | `exec-tasks` | implemented (migrated from `work-on-tasks`) |
| Review | `review-impl` | planned |

See `README.md` for the full vision.

## Workflow Contract

`docs/WORKFLOW_CONTRACT.md` defines the shared shape that `init-project` writes and `exec-tasks` reads: issue template, label vocabulary, milestone naming, dependency syntax, priority ordering, and the paths to project documentation files.

**Both plugins must stay in sync with the current contract version.** If you change the contract:
1. Bump the `Contract version` header in `docs/WORKFLOW_CONTRACT.md`
2. Update both `init-project/skills/init-project/SKILL.md` and `exec-tasks/skills/exec-tasks/SKILL.md` in the same PR
3. Note the version bump in the commit message

## Adding a new plugin

1. Create `<plugin-name>/` with `.claude-plugin/plugin.json`, `skills/<plugin-name>/SKILL.md`, and bilingual `README.md`
2. If the plugin exposes a slash command, add `commands/<command>.md`
3. Append an entry to `.claude-plugin/marketplace.json`
4. Update the top-level `README.md` plugin list (EN and JP sections)
5. If the plugin interacts with the workflow contract, link to `docs/WORKFLOW_CONTRACT.md` in its SKILL.md
