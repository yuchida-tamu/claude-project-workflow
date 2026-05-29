# Changelog — plan-feature

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [0.1.0] — 2026-05-30

Initial release. Ported and refined from the `plan-feature` skill developed in the basketball-sim-game project via `/post-session`.

### Skills

- **`plan-feature`** — Design a feature before implementation: complexity assessment (with `/grill-me` hook), research, structured plan creation, ADR extraction, user approval gate, GitHub issue creation (contract v3 conforming), and close-out via `doc-feature`.
- **`doc-feature`** — Generate feature documentation from implemented code: produces `docs/features/{feature-name}.md` with mermaid user-flow and system-flow diagrams, data model, key files table, and design decisions. Called by `plan-feature` at close-out and available standalone.
