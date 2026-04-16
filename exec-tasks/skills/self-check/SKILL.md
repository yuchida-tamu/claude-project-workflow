---
name: self-check
description: Cross-check a completed implementation against its plan document. Verifies file manifest, mermaid diagram correspondence, acceptance criteria coverage, and ADR compliance. Runs up to 2 rounds. On final failure, opens the PR as a draft with unresolved findings in the body. Invoked by the coding agent before committing.
version: 0.1.0
---

# Self-Check

Cross-check an implementation against its plan document before the coding agent commits and opens a PR. Catches the five most common ways an agent-produced PR drifts from its plan.

## When to use

- Invoked programmatically by the coding agent spawned by `exec-tasks`, **after** the implementation is written and tests pass, **before** the agent runs `git commit`.
- Not a user-facing slash command.

## Inputs

- `issue_number` — the GitHub issue this check is for
- The plan document at `.memory/plans/<issue_number>-plan.md`
- The current working-tree changes (via `git status` and `git diff`)

## Output

A structured findings report and a disposition:

- **PASS** — all checks green. Agent proceeds to commit and open a normal PR.
- **FAIL (iterate)** — at least one check failed, but this is round 1. Report findings and instructions, return control to the coding agent for a fix round.
- **FAIL (draft PR)** — at least one check failed after 2 rounds. Agent commits, opens a **draft** PR with the findings as a warning section in the PR body, and reports to the orchestrator.

## The five checks

### Check 1: File manifest matches the diff

Parse the plan's "File manifest" section into three lists: created, modified, deleted. Run `git status --porcelain` and `git diff --name-only HEAD` to get the actual changes. Compare:

- **Missing files**: any file in the plan manifest that is NOT in the actual diff → FAIL (the agent planned to touch a file but didn't)
- **Unexpected files**: any file in the actual diff that is NOT in the plan manifest → FAIL (the agent touched a file without planning to — suggests the plan is stale)
- **Wrong operation**: any file that was planned as "created" but exists as "modified" in the diff (or vice versa) → FAIL

Exceptions allowed without failure:
- Lock files (`package-lock.json`, `Cargo.lock`, `yarn.lock`, `pnpm-lock.yaml`, `go.sum`) — auto-updated, never require plan entries
- Test snapshot files (`__snapshots__/*`) — auto-generated, don't require plan entries
- Generated files the project explicitly marks as auto-generated (look for a `.gitattributes` `linguist-generated` entry)

### Check 2: Mermaid diagrams reference real components

For each mermaid code block in the plan, extract every node label that looks like a file path, function name, or component identifier (heuristic: PascalCase, camelCase with dots, or paths with `/`). For each extracted identifier:

- If it looks like a file path → verify the file exists on disk after the implementation
- If it looks like a function or type → grep the codebase for its definition

Any identifier in the diagram that cannot be found → FAIL (the diagram is lying about what was built).

Non-file nodes that are clearly abstract labels (e.g., `[User]`, `[Database]`, `[API]`) are allowed without verification — the heuristic is "does this look like a code symbol or path?"

### Check 3: Acceptance criteria have corresponding tests

Parse the plan's "Acceptance criteria" section (the verbatim copy from the issue). For each criterion, look at the "Test plan" section and verify:

- There is a line mapping this criterion to a test file
- The test file exists on disk
- The test file was touched in this diff (i.e., the agent actually wrote the test, not just referenced a pre-existing one)

Any criterion without a corresponding test → FAIL.

If the test file was pre-existing and unchanged, the criterion must explicitly say `(pre-existing test, verified by this task)` in the Test plan section to be acceptable. Otherwise → FAIL.

### Check 4: ADR compliance

Re-read every file in `docs/adr/` that was listed in the plan's "Constraints" section. For each ADR, check whether the implementation violates the stated decision:

- Parse the ADR's "Decision" section
- Scan the diff for code patterns that would violate it (heuristic, not exhaustive — e.g., if ADR-003 says "no global state," grep the diff for `export let`, module-level `var`, singletons)

Any clear violation → FAIL with the ADR number and the violating location.

This check is intentionally conservative: if the heuristic is unsure, PASS it. False negatives are better than false positives here — the `@claude` review at PR time will catch anything self-check missed.

### Check 5: Plan document is non-stub

Verify the plan document itself:

- Exists at `.memory/plans/<issue_number>-plan.md`
- All mandatory sections from the contract §8 are present and non-empty (no `TODO` placeholders, no empty lists, no "write this later")
- Both mermaid code blocks parse (have valid opening `flowchart`/`stateDiagram-v2`/`classDiagram`/`erDiagram` directive and non-empty body)

Any missing or stub section → FAIL.

## Iteration protocol

Self-check runs up to **2 rounds** total.

### Round 1

Run all five checks. Aggregate findings into a list. Each finding has:
- **Check**: which check (1–5)
- **Severity**: FAIL (blocks PR) or WARN (note but don't block)
- **Location**: file path or plan section
- **Description**: what's wrong
- **Instruction**: what the coding agent should do to fix it

If all checks PASS or all FAILs are WARN-only → **disposition: PASS**. Return to caller.

If any FAIL → return the findings report with **disposition: FAIL (iterate)**. The coding agent reads the instructions, makes fixes (including updating the plan if the plan was wrong), then invokes self-check again.

### Round 2

Run all five checks again. This time:

- If all PASS → **disposition: PASS**
- If any FAIL → **disposition: FAIL (draft PR)**. Do NOT iterate a third time.

### Disposition: FAIL (draft PR)

When self-check ends in "FAIL (draft PR)":

1. The coding agent commits its work as-is (with the failing state documented)
2. The coding agent opens the PR with `gh pr create --draft`
3. The PR body includes this section at the top:

```markdown
> ⚠️ **Self-check failed after 2 rounds.** This PR is opened as a draft because automated cross-checking detected discrepancies between the plan and the implementation. Resolve the findings below before promoting to ready-for-review.
>
> ### Unresolved findings
>
> - **[Check N]** <description> — <location>
>   - **To fix:** <instruction>
> - ...
```

4. The coding agent reports the draft status back to the `exec-tasks` orchestrator, which surfaces it in the final summary as a warning.

## Rules

- **Two rounds, no more.** Unbounded iteration masks real problems. If a task cannot pass self-check in 2 rounds, the plan was wrong, the issue was underspecified, or the implementation is fundamentally off — all cases require a human.
- **Fixing the plan is a valid fix.** If Check 1 fails because the agent touched a file that wasn't in the manifest, the correct Round 1 fix is often "update the plan to add the file" — not "revert the file change." Self-check is verifying that plan and code agree, not that the plan is immutable.
- **Never skip a check.** All five run every round. A failure in one check doesn't short-circuit the others — the findings report should be complete so the agent can fix everything in one batch.
- **Draft PRs are not failures of this skill.** They are the designed degraded-state outcome. The skill's job is to open a draft correctly with accurate findings, not to prevent drafts from ever happening.
- **Do not commit fixes for the coding agent.** Self-check reports findings and returns control. The coding agent does the actual fixes.
- **Check 4 is conservative.** Prefer false negatives to false positives. Human review catches what heuristics miss.
