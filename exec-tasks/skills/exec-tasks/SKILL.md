---
name: exec-tasks
description: Review GitHub project issues, select the next tasks based on priority and dependencies, spawn coding agents to execute them in parallel where possible, and create PRs that trigger automated code review.
version: 0.1.0
---

# Task Executor

Review the GitHub project board of the current repository, identify the highest-priority unblocked tasks, and execute them by spawning coding agents.

## When to use

- When the user says "/exec-tasks" or asks to "execute the next tasks"
- When the user asks to "pick up work" or "continue building"

## Workflow Contract

This skill reads issues, labels, and milestones according to the shared workflow contract at [`docs/WORKFLOW_CONTRACT.md`](../../../docs/WORKFLOW_CONTRACT.md) in the `claude-project-workflow` repo (contract version 2). Projects scaffolded by `init-project` conform to this contract automatically.

**Key contract points this skill relies on:**
- **Issue body template** has `## Summary`, `## Acceptance Criteria`, `## Priority`, `## Depends on` sections
- **Priority labels:** `P0` > `P1` > `P2` > `P3` (exactly one per issue)
- **Type labels:** `type:feature`, `type:bug`, `type:chore`, `type:docs`
- **Milestones** match `^M\d+$` (e.g., `M1`, `M2`)
- **Dependencies** are listed in the `## Depends on` section as `#N` tokens; the literal `none` means no dependencies
- **Priority sort order:** P0 first, ties broken by issue number ascending
- **Project docs** live at fixed paths: `CLAUDE.md`, `docs/PRD.md`, `docs/glossary.md`, `docs/adr/`, `.memory/`

If you are operating on a repo that does NOT conform to this contract, report that to the user and stop — don't guess at alternate formats.

## Workflow

### Step 1: Assess project state

Determine the current repository from git remote, then query its state:

```bash
# Determine current repo (nameWithOwner)
REPO=$(gh repo view --json nameWithOwner --jq .nameWithOwner)

# Get all open issues with labels and milestones
gh issue list --repo "$REPO" --state open --json number,title,labels,milestone,body --limit 100

# Check what's currently in progress (open PRs)
gh pr list --repo "$REPO" --state open --json number,title,headRefName

# Check recent closed issues to understand what's done
gh issue list --repo "$REPO" --state closed --json number,title --limit 20
```

### Step 2: Select tasks

Apply these rules in order (per the workflow contract):

1. **Filter to current milestone.** Work on the earliest incomplete milestone first (lowest `M<n>` that has open issues).
2. **Check dependencies.** Parse each issue's `## Depends on` section for `#<number>` tokens. A task is **blocked** if any referenced dependency issue is still open. Skip blocked tasks.
3. **Sort by priority.** Among unblocked tasks in the current milestone, select by priority label: `P0` > `P1` > `P2` > `P3`. Ties broken by issue number ascending.
4. **Identify parallelizable tasks.** Two tasks can run in parallel if:
   - Neither depends on the other
   - They don't modify the same files (infer from issue bodies and acceptance criteria)
   - Both are unblocked
5. **Cap concurrency.** Run at most 3 agents in parallel.

### Step 3: Gather context for each task

For each selected issue, invoke the **`prepare-context` sub-skill** (in `exec-tasks/skills/prepare-context/SKILL.md`). It produces `.memory/plans/<issue_number>-context.md` containing:

- Issue summary and linked-issue graph
- Relevant ADR excerpts
- Relevant PRD section(s)
- Glossary terms used in the issue
- Symbol grep results for key nouns
- Recent merged PRs touching hinted files
- Library documentation (best-effort WebFetch for third-party deps mentioned in the issue)

The orchestrator runs `prepare-context` once per selected issue, in parallel with other issues' context preparation where safe. Each resulting context file is passed to the coding agent spawned in Step 4.

Do not duplicate context-gathering work inline here — delegate to the sub-skill so every issue gets a consistent, structured package written to disk.

### Step 4: Spawn coding agents

For each selected task, spawn an Agent with:
- `subagent_type: "expert-programmer"`
- `isolation: "worktree"` (each agent works on an isolated copy)
- A comprehensive prompt that includes:
  1. **The task:** issue number, issue title, and a pointer to `.memory/plans/<N>-context.md` (produced in Step 3) as the primary context input
  2. **Required workflow for the coding agent** (the agent MUST follow this order):
     1. Read `.memory/plans/<N>-context.md` end-to-end
     2. Invoke the **`plan-task` sub-skill** to produce `.memory/plans/<N>-plan.md` before writing any implementation code
     3. Implement the task, using the plan's file manifest as a commitment (update the plan first if the manifest needs to change)
     4. Run the project's check command (from CLAUDE.md) until it passes
     5. Invoke the **`self-check` sub-skill** to cross-check the implementation against the plan
     6. If self-check returns FAIL (iterate), fix the findings and re-run self-check. If round 2 also fails, proceed to draft PR per the self-check protocol.
     7. Only after self-check PASSes (or returns FAIL (draft PR)), commit and push
  3. **Project rules:** read CLAUDE.md for conventions and the check command
  4. **Architecture constraints:** ADRs in `docs/adr/` (the plan will list the constraining ones)
  5. **Branch naming:** `feature/{issue-number}-{short-description}` (or `fix/{issue-number}-...` for bugs based on the `type:` label)
  6. **Commit message:** reference the issue number (e.g., "Implement seeded RNG system (#8)")
  7. **Commit the plan documents:** both `.memory/plans/<N>-context.md` and `.memory/plans/<N>-plan.md` must be committed as part of the implementation PR per workflow contract §8.3

Read the project's CLAUDE.md to discover the exact check command (commonly `npm run check`, `cargo check`, `pytest`, etc.). Pass it to the agent as the required pre-completion gate.

**Sub-skill invocation by the coding agent.** The three sub-skills (`plan-task`, `self-check`, and optionally re-running `prepare-context` for additional gathering) are installed as part of this plugin and discoverable via the agent's skill system. The orchestrator's prompt must explicitly instruct the agent to invoke them by name at the specified phases — do not assume the agent will discover them on its own.

If tasks are independent, spawn multiple agents in a **single message** (parallel tool calls).

If tasks are sequential (one depends on the other), spawn them one at a time, waiting for the previous to complete.

### Step 5: Create PRs and trigger review

After each agent completes and returns its worktree branch:

1. Check the self-check disposition the agent reported back:
   - **PASS** → normal PR
   - **FAIL (draft PR)** → the agent has already opened a draft PR with findings in the body (per the `self-check` protocol). The orchestrator does NOT re-open it; it only needs to post the `@claude` review comment below and surface the draft status to the user in Step 6.
2. Push the branch to origin (if not already pushed by the agent)
3. Create a PR via `gh pr create` with (for PASS case only):
   - Title referencing the issue: e.g., "Implement seeded RNG system (#8)"
   - Body with summary, test plan, and closing keyword: `Closes #8`
   - Labels matching the issue's priority and type labels
   - Milestone matching the issue milestone

PR body template:
```markdown
## Summary
{what was implemented}

## Test plan
{test scenarios covered}

Closes #{issue_number}

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

3. **Trigger code review** by posting a `@claude` comment on the PR. This triggers the `claude.yml` GitHub Action (installed by `init-project` or manually configured) which responds to `@claude` mentions.

Use this command for each PR:
```bash
gh pr comment {pr_number} --repo "$REPO" --body '@claude Please review this PR.

In addition to standard code-level review (correctness, style, bugs, security), perform the following project-level checks:

1. **ADR compliance** — Read docs/adr/ and verify the implementation does not violate any accepted ADR.
2. **Glossary consistency** — Read docs/glossary.md and verify correct ubiquitous language usage.
3. **PRD alignment** — Read docs/PRD.md and verify the implementation matches the spec.
4. **Documentation completeness** — Flag if new ADRs, feature docs, glossary updates, or CLAUDE.md changes are needed.
5. **Convention compliance** — Read CLAUDE.md and verify the project conventions are followed.

Report findings in clearly labeled sections.'
```

### Step 6: Report to user

After all agents complete and PRs are created, present a summary:

- Which issues were worked on
- PR links for each, **marked `[DRAFT]` if self-check ended in FAIL after 2 rounds** — include the unresolved findings inline so the user can triage without opening the PR
- Any issues that were skipped (blocked, unclear requirements)
- What the next unblocked tasks will be after these merge

## Rules

- **Never work on blocked tasks.** If all tasks in the current milestone are blocked, report this to the user and suggest unblocking actions.
- **Always verify the project's check command passes** before creating a PR. If an agent's work fails checks, diagnose and fix before pushing.
- **Respect the Development Workflow** in CLAUDE.md: branch → PR → review → merge. Never push to main.
- **Each agent gets one issue.** Don't combine multiple issues into one agent — it makes review harder and risks merge conflicts.
- **Update issue status automatically.** The PR's `Closes #N` will auto-close the issue on merge. No manual status update needed.
- **If a task requires design decisions not covered by the PRD or ADRs**, don't guess. Report it to the user and suggest running `/plan-feature` (when available) or a design discussion first.
- **Agents must run the check command before considering their work done.** Read CLAUDE.md to find it.
- **Agents must invoke `plan-task` before writing code and `self-check` before committing.** These are not optional. A coding-agent run that skips either is a contract violation — the orchestrator's agent prompt must name both sub-skills explicitly and state the order.
- **Plan documents are committed, not gitignored.** Per workflow contract §8.3, both `.memory/plans/<N>-context.md` and `.memory/plans/<N>-plan.md` ship with the implementation PR.
- **Contract non-conformance is a hard stop.** If issues in the repo don't match the workflow contract (missing Depends on section, wrong label names, milestones not matching `^M\d+$`), stop and report — don't silently pick alternate parsing.
