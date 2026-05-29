---
description: Toggle or set auto-merge mode for this project. With auto-merge on, exec-tasks merges PRs automatically once the @claude review reports no blocking findings.
allowed-tools: Bash, Read, Edit
argument-hint: "[on|off]"
---

Read `CLAUDE.md` in the current working directory.

Find the `## Workflow settings` section and the `Auto-merge:` line within it.

- If the argument is `on`: set the value to `on`.
- If the argument is `off`: set the value to `off`.
- If no argument is given: toggle the current value (`on` → `off`, `off` → `on`).
- If `## Workflow settings` does not exist in `CLAUDE.md`: append the section at the end of the file with the requested value (defaulting to `on` if no argument was given).

After updating, print the result to the user:

```
Auto-merge: on   ✓ exec-tasks will merge PRs automatically after a clean @claude review.
```
or
```
Auto-merge: off  ✓ exec-tasks will surface review findings and wait for your merge decision.
```
