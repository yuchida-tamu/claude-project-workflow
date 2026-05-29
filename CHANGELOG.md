# Changelog

All notable changes to the claude-project-workflow marketplace are recorded here.  
See individual plugin changelogs for per-plugin history: [`exec-tasks/CHANGELOG.md`](./exec-tasks/CHANGELOG.md), [`init-project/CHANGELOG.md`](./init-project/CHANGELOG.md).

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [0.3.0] — 2026-05-30

### Added

- **`plan-feature` plugin** — the missing **Plan** phase of the design → plan → execute → review workflow. Ported and refined from a battle-tested skill developed in a production project via `/post-session`.
  - **`plan-feature` skill** — complexity assessment (with `/grill-me` hook for underspecified features), codebase research, structured plan creation (mermaid diagrams, file manifest, phased tasks), ADR extraction, user approval gate, GitHub issue creation conforming to workflow contract v3 (labels, milestones, `Depends on` links), and close-out via `doc-feature`.
  - **`doc-feature` skill** — generates `docs/features/{feature-name}.md` from the *actual* implementation after all feature issues are merged. Produces mermaid user-flow and system-flow diagrams, data model, key files table, and design decisions. Callable standalone for existing or refactored features. Output is the canonical reference doc for reviewers and future contributors.

---

## [0.2.0] — 2026-05-30

### Added

- **Auto-merge** (`exec-tasks`) — `exec-tasks` automatically merges PRs after a clean `@claude` review when `Auto-merge: on` is set in the project's `CLAUDE.md`. Opt in with `--auto-merge` at init time or `/auto-merge [on|off]` any time.
- **`/auto-merge` command** (`exec-tasks`) — toggle or set auto-merge mode in any project that uses the workflow contract. No argument toggles the current value.
- **Model selection for execution agents** (`exec-tasks`) — orchestrator now chooses `claude-haiku-4-5` for `type:chore` and `type:docs` tasks, and `claude-sonnet-4-6` for `type:feature` and `type:bug` tasks. Opus is never used for execution agents.
- **`[BLOCKING]` / `[SUGGESTION]` review markers** (`exec-tasks`) — the `@claude` PR review prompt now instructs the reviewer to prefix every must-fix finding with `**[BLOCKING]**` and non-critical observations with `**[SUGGESTION]**`, making auto-merge detection deterministic.
- **Workflow contract v3** — added §9 documenting the `## Workflow settings` section in `CLAUDE.md`, the `Auto-merge` key, the `[BLOCKING]` heuristic, and the `/auto-merge` toggle command.

### Changed

- **`@claude` PR review pinned to `claude-opus-4-8`** — both the project's own `.github/workflows/claude.yml` and the template scaffolded by `init-project` now specify `model: claude-opus-4-8` in the GitHub Action.
- **`plan-task`** — added explicit end-to-end thinking instruction before writing any plan section, reducing plan drift on complex tasks.
- **`exec-tasks` concurrency** — rate-limit recovery note added: if an agent returns a rate-limit error, reduce concurrency from 3 to 2 and retry rather than aborting.
- **`exec-tasks` agent prompts** — P0 and ADR-heavy tasks now include reasoning-depth guidance to spend extra time in the planning phase.
- **`exec-tasks` wakeup prompt** — updated example to read `Auto-merge` from `CLAUDE.md` and either merge silently or surface findings to the user.

---

## [0.1.0] — 2026-04-16

Initial release. Ships `init-project`, `exec-tasks`, and `post-session`.
