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

This skill reads issues, labels, and milestones according to the shared workflow contract at [`docs/WORKFLOW_CONTRACT.md`](../../../docs/WORKFLOW_CONTRACT.md) in the `claude-project-workflow` repo (contract version 3). Projects scaffolded by `init-project` conform to this contract automatically.

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
5. **Cap concurrency.** Run at most 3 agents in parallel. If an agent returns a rate-limit error, reduce concurrency to 2 and retry — do not abort the run.

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

#### Pre-flight: worktree availability

Worktree isolation requires that Claude Code recognized this directory as a git repo **at session start**. When a project was just scaffolded by `/init-project` in the *same* session, `git init` ran mid-session and the harness has not re-detected it — `isolation: "worktree"` will fail even though `.git/` exists on disk.

Before spawning agents, detect this case. It applies if `.git/` exists but the session began in a non-git directory — most commonly when `/init-project` ran earlier in this same session, or when the first `isolation: "worktree"` spawn errors with a git/worktree error.

```bash
git rev-parse --is-inside-work-tree 2>/dev/null   # does .git exist on disk?
```

If worktree isolation is unavailable, do NOT abort. Tell the user:

> This repo's git was initialized after the session started, so worktree isolation isn't available yet. Restart Claude Code in this directory for full parallel worktree execution — or I can proceed now by running agents **sequentially in the main working tree** (one feature branch at a time, no worktree isolation). Restart, or proceed sequentially? (restart / sequential)

- **restart** → stop and let the user restart Claude Code; on the next run worktrees work normally.
- **sequential** → fall back to in-place execution: process the selected issues **one at a time** (concurrency 1), each on its own `feature/...` / `fix/...` branch in the main working tree, with `isolation` omitted. After each issue's PR is opened, switch back to `main` before creating the next branch. Every other step (`plan-task`, `self-check`, PR creation, the `@claude` trigger, the wakeup) is unchanged.

The sequential fallback trades parallelism for working without a restart. Note in the Step 6 summary when it was used.

When worktree isolation IS available, spawn an Agent for each selected task with:
- `subagent_type: "expert-programmer"`
- `isolation: "worktree"` (each agent works on an isolated copy)
- **`model`:** chosen by the orchestrator based on task complexity (see model selection table below). Never use an Opus model for execution agents — Opus is reserved for planning review via the GitHub Action.

**Model selection table:**

| Condition | Model |
|---|---|
| `type:chore` or `type:docs`, any priority | `claude-haiku-4-5` |
| `type:feature` or `type:bug`, P2 or P3 | `claude-sonnet-4-6` |
| `type:feature` or `type:bug`, P0 or P1 | `claude-sonnet-4-6` |

Sonnet is the ceiling for execution agents. The extra reasoning capacity that Opus provides is covered by the `plan-task` phase (which runs inside the agent) and the `@claude` PR review (which runs on Opus via the GitHub Action).

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
  5. **Reasoning depth:** For issues labelled `P0` or whose plan's Constraints section lists one or more ADRs, instruct the agent to spend extra time in the planning phase before writing any code — these are the tasks where a poor plan causes the most downstream rework.
  6. **Branch naming:** `feature/{issue-number}-{short-description}` (or `fix/{issue-number}-...` for bugs based on the `type:` label)
  7. **Commit message:** reference the issue number (e.g., "Implement seeded RNG system (#8)")
  8. **Commit the plan documents:** both `.memory/plans/<N>-context.md` and `.memory/plans/<N>-plan.md` must be committed as part of the implementation PR per workflow contract §8.3
  9. **Scope & branch discipline (hard limits):**
     - Work only inside your worktree on your `feature/`|`fix/` branch. Never check out, commit to, or push `main`.
     - Touch only the files your issue requires. Do NOT edit `CLAUDE.md`, `docs/PRD.md`, `docs/glossary.md`, `docs/adr/**`, `docs/WORKFLOW_CONTRACT.md`, or `.github/**` unless the issue's acceptance criteria explicitly name that file. (Committing your two `.memory/plans/<N>-*.md` files is required — that is not a project-meta edit.)
     - If executing the issue reveals a missing **project-wide** decision the docs don't cover (folder layout, a cross-cutting convention, an architectural choice), STOP. Do not author that decision and do not push it. Record it under `## Open questions` in your plan and return the blocker to the orchestrator.

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

Report findings in clearly labeled sections. For each finding that must be resolved before this PR can merge, begin the line with **[BLOCKING]**. For suggestions or non-critical observations, begin with **[SUGGESTION]**. A PR with no [BLOCKING] findings is safe to merge.'
```

4. **Schedule a wakeup to triage review feedback.** The `claude.yml` GitHub Action takes a few minutes to post its review. Don't wait synchronously and don't ask the user to ping you back — schedule a return visit via `ScheduleWakeup` so the review can be triaged in the same session.

   - **Delay:** 240–270 seconds (~4 min). Long enough for the action to post its review in the typical case, while staying inside the 5-min prompt-cache TTL so the wakeup is cheap. Don't pick 300s — that's the worst-of-both (cache miss without amortizing it). If the review hasn't landed on wake-up, schedule one more 240s wakeup; don't busy-poll.
   - **Reason field:** be specific — e.g. `"checking PR #N for @claude review feedback"`.
   - **Prompt field:** pass back enough context for cold-start. The wakeup may reload the conversation lossily, so write the prompt as if context were missing. Example: `"Check PR #101 (https://github.com/<owner>/<repo>/pull/101) for the @claude review comment via 'gh pr view 101 --comments'. Read CLAUDE.md and check the 'Auto-merge:' value under '## Workflow settings'. If the review has posted: (a) if Auto-merge is 'on' and the review contains no [BLOCKING] markers, merge immediately with 'gh pr merge 101 --repo <owner>/<repo> --squash --delete-branch' and report to the user; (b) otherwise, summarize findings to the user and propose follow-ups. If the review hasn't posted yet, schedule another 240s wakeup."`
   - **Multiple PRs in one run:** schedule a single wakeup that triages all of them in one pass (`gh pr view <num> --comments` for each), not one wakeup per PR.
   - **Skip the wakeup** only if the user has explicitly said they'll handle review themselves.

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
- **No project-meta edits outside an issue.** Neither the orchestrator nor any coding agent may modify `CLAUDE.md`, `docs/PRD.md`, `docs/glossary.md`, `docs/adr/**`, `docs/WORKFLOW_CONTRACT.md`, or `.github/**` unless an issue's acceptance criteria explicitly require it. Those are project-wide decisions owned by `/init-project` and `/plan-feature`. The orchestrator's only writes are the per-issue `.memory/plans/*` files and PR creation. Do not rely on the harness's auto-approve mode to gate a `git push` — never attempt one against `main`.
- **Surface project-wide gaps; don't fill them.** If a task can't proceed without a project-wide decision the docs don't cover (e.g., the folder structure is undefined — note that `init-project` now pins this in `docs/adr/0003-project-structure.md`), stop that task and report it to the user, suggesting `/init-project --resume` or `/plan-feature`. Authoring the decision ad hoc and pushing it — especially to `main` — is a contract violation, not a convenience.
- **Each agent gets one issue.** Don't combine multiple issues into one agent — it makes review harder and risks merge conflicts.
- **Update issue status automatically.** The PR's `Closes #N` will auto-close the issue on merge. No manual status update needed.
- **If a task requires design decisions not covered by the PRD or ADRs**, don't guess. Report it to the user and suggest running `/plan-feature` (when available) or a design discussion first.
- **Agents must run the check command before considering their work done.** Read CLAUDE.md to find it.
- **Agents must invoke `plan-task` before writing code and `self-check` before committing.** These are not optional. A coding-agent run that skips either is a contract violation — the orchestrator's agent prompt must name both sub-skills explicitly and state the order.
- **Plan documents are committed, not gitignored.** Per workflow contract §8.3, both `.memory/plans/<N>-context.md` and `.memory/plans/<N>-plan.md` ship with the implementation PR.
- **Always schedule a wakeup after triggering `@claude` review.** A PR with a pending automated review is not "done" — the review feedback needs triage in the same session. Use `ScheduleWakeup` with a 240–270s delay (see Step 5.4) so the review can be acted on without the user having to ping back. Skip only if the user explicitly takes review on themselves.
- **Contract non-conformance is a hard stop.** If issues in the repo don't match the workflow contract (missing Depends on section, wrong label names, milestones not matching `^M\d+$`), stop and report — don't silently pick alternate parsing.
- **Never spawn an Opus execution agent.** Opus is reserved for the `@claude` PR review step (GitHub Action). Execution agents use `claude-sonnet-4-6` or `claude-haiku-4-5` based on task type and priority — see the model selection table in Step 4.
