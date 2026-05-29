# Changelog — init-project

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [0.2.0] — 2026-05-30

### Added

- `--auto-merge` flag — sets `Auto-merge: on` in the generated `CLAUDE.md`, skipping Q14 in the interview.
- **Q14 (Auto-merge)** — new interview question asking whether to enable auto-merge. Skipped automatically if `--auto-merge` flag was passed.
- `## Workflow settings` section in generated `CLAUDE.md` — records the `Auto-merge` preference (`on` or `off`).
- `auto_merge` key in `.init-project-state.json` flags object — preserved across `--resume` runs.

### Changed

- Generated `CLAUDE.md` and `.github/workflows/claude.yml` now pin the `@claude` review GitHub Action to `model: claude-opus-4-8`.
- Workflow contract reference updated to v3.

---

## [0.1.0] — 2026-04-16

Initial release.
