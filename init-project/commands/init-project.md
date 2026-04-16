---
description: Interview the user about a new project, then scaffold CLAUDE.md, PRD, ADRs, glossary, GitHub repo, labels, milestones, and seeded M1 issues that conform to the workflow contract.
allowed-tools: Bash, Read, Glob, Grep, Write, Edit, Skill
argument-hint: "[name] [--skip-github] [--no-hook] [--dry-run] [--resume] [--name=<name>] [--license=mit|apache-2.0|none]"
---

Invoke the `init-project` skill. Pass through any positional `name` argument and flags (`--skip-github`, `--no-hook`, `--dry-run`, `--resume`, `--name=<name>`, `--license=...`). Follow the skill's workflow exactly: interview, preview gate, phased execution with hard-fail preservation.
