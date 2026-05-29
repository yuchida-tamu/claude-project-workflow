# Changelog — exec-tasks

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [0.2.0] — 2026-05-30

### Added

- **Auto-merge** — after the wakeup fires, automatically merges PRs that have no `[BLOCKING]` findings when `Auto-merge: on` is set in the project's `CLAUDE.md`. Merge is performed with `gh pr merge --squash --delete-branch`.
- **`/auto-merge` command** — toggle or set `Auto-merge` in the current project's `CLAUDE.md`. `/auto-merge on`, `/auto-merge off`, or bare `/auto-merge` to toggle.
- **Model selection table** — orchestrator selects `claude-haiku-4-5` for `type:chore` / `type:docs` tasks and `claude-sonnet-4-6` for `type:feature` / `type:bug` tasks. Opus is never used for execution agents.
- **Reasoning-depth guidance** — P0 and ADR-heavy tasks include an explicit instruction in the agent prompt to spend extra time in the planning phase before writing any code.
- **Rate-limit recovery** — concurrency cap note updated: on rate-limit error, reduce from 3 to 2 agents and retry rather than aborting the run.

### Changed

- `@claude` PR review prompt now instructs the reviewer to prefix every must-fix finding with `**[BLOCKING]**` and non-critical observations with `**[SUGGESTION]**`, enabling deterministic auto-merge detection.
- Wakeup prompt example updated to read `Auto-merge` from `CLAUDE.md` and act accordingly (merge or surface to user).
- Workflow contract reference updated to v3.

---

## [0.1.0] — 2026-04-16

Initial release.
