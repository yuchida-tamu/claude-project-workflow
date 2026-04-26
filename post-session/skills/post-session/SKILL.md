---
name: post-session
description: Review the current session, extract durable findings (gotchas, workflows, preferences, repeated patterns), classify each by where it belongs (skill, skill-update, memory, or nowhere), and act on the skill-worthy ones by drafting or updating the relevant SKILL.md files. Use this whenever the user wants to wrap up a session, capture lessons learned, or compound knowledge for future sessions. Trigger phrases include "post-session", "review the session", "let's wrap up", "extract findings", "what did we learn", "make next time better", "compound this", "capture lessons". Run this proactively at the end of any non-trivial session — the goal is that the next session is materially more efficient because the friction we hit this time is captured somewhere durable.
---

# Post-Session Review

Compound the lessons from this session so the next session is more efficient. Look at what slowed us down or surprised us, decide which lessons are worth keeping, and put each one in the right place: a new skill, an update to an existing skill, a memory entry, or — sometimes — nowhere.

## When to use

- When the user says "/post-session", "let's wrap up", "review the session", "extract findings", "make next time better", "what did we learn".
- Proactively at the end of any session that included real friction the user worked through (a workflow that took several attempts, a gotcha that wasn't documented, a preference the user stated firmly).
- After completing a multi-PR feature where the orchestration pattern itself is reusable.

Skip if the session was trivial (a single short edit, a one-line answer, a question-only conversation). The threshold is: *would future-me have benefited from a written-down version of what we just did?*

## Workflow

### Step 1: Survey the session

Read back through the conversation in your own context. Don't ask the user to re-summarize what happened — they were there, they want to know what *you* extracted. Pay attention to:

- **Friction points** — moments where something failed and we had to retry or work around it (build errors, command failures, unexpected tool behavior, prompts that blocked non-interactive mode, paths that didn't exist, environments that needed setup).
- **Sequences that worked** — multi-step procedures that produced a clean outcome the second or third time we tried them. The first attempt is the discovery; the version that finally worked is the runbook.
- **User corrections and confirmations** — places where the user said "no, do it this way" or "yes, exactly that". Both directions matter.
- **Repeated patterns** — if we did the same kind of thing 3+ times in this session (or across recent sessions), the steps are probably worth capturing.
- **Context that was hard to reconstruct** — if you spent a turn or two grepping the codebase, reading docs, or asking the user to clarify something that should have been obvious, the answer to that is a candidate finding.
- **Decisions and preferences** — if the user expressed a strong preference about how work should happen ("always X", "never Y", "from now on Z"), that's a finding even if it didn't fix any specific friction.

Don't filter at this stage. Get the raw list down first.

### Step 2: Classify each candidate

For each finding, decide where it belongs. Use this decision tree:

```
Is it a multi-step workflow / runbook / gotcha-laden process?
├── Yes, and there's no existing skill for it          → NEW SKILL
├── Yes, and an existing skill could absorb it         → SKILL UPDATE
└── No
    Is it a fact, preference, or piece of context that
    should persist across sessions?
    ├── Yes — about the user                            → user memory
    ├── Yes — feedback / correction / confirmation     → feedback memory
    ├── Yes — about the project's current state        → project memory
    ├── Yes — pointer to an external system            → reference memory
    └── No                                              → discard
```

The auto-memory system handles all four memory categories — see the harness's memory instructions for the format. **Don't duplicate**: if something already exists in this project's `MEMORY.md` (under the auto-memory directory `~/.claude/projects/<project-slug>/memory/`, where the slug is derived from the project's absolute path with `/` replaced by `-`) or in the codebase itself (CLAUDE.md, ADRs, glossary), don't re-capture it.

#### When is something skill-worthy?

A finding is skill-worthy when **all** of these hold:

1. **It's a process, not a fact.** Skills are runbooks; memory is for facts.
2. **It has at least 3 non-obvious steps**, or fewer steps with at least one real gotcha that bit us this session. A one-step "do X" is not a skill.
3. **It will recur.** Either the same task will come up again, or the same shape of task will come up. If this was genuinely one-off, it doesn't earn a skill.
4. **It's not already covered.** Check `.claude/skills/` and the user's global `~/.claude/skills/` first.

Examples of *good* skill candidates:
- "How to verify a UI change on the local simulator/emulator on this machine" — multi-step, gotcha-laden (PTY wrapper, monitor switching, terminal click-through), recurring.
- "How to roll a task-execution iteration with parallel worktrees on this repo" — already a skill in many setups; a new pattern would be a skill update.
- "How to set up a fresh project that matches this org's conventions" — multi-step, recurring whenever a sister project starts.

Examples of *bad* skill candidates:
- "The user prefers tabs over spaces" — that's a preference; it goes in memory.
- "The simulator is at iPhone 17 Pro UDID 22D095..." — that's a fact, and it'll change. Memory at best, probably discard.
- "We renamed a function from X to Y" — that's git history.

#### When is a skill *update* better than a new skill?

If the finding fits inside the scope of an existing skill (read its description), updating is almost always better. Three small skills that overlap are worse than one well-organized skill. The exception: when a finding is in a clearly different domain from any existing skill.

### Step 3: Draft proposals (don't write yet)

Before writing or editing any files, write a short bullet list to the user:

```
Findings from this session:

Skill-worthy:
- [NEW SKILL: <name>] — <one-line summary of what it captures and why>
- [UPDATE skill <name>] — <what to add and why>

Memory-worthy:
- [<type> memory] — <one-line summary>

Discarded (already captured / not durable / one-off):
- <one-line summary> — <reason>
```

Ask the user: "Should I proceed with these, or do you want to adjust the list first?" This is a cheap checkpoint that prevents you from writing the wrong thing — the user has full session context and can veto fast.

If the user confirms or course-corrects, proceed. If they say "go", execute on the confirmed list.

### Step 4: Execute

For each confirmed item:

**New skill:**
- Place it in `.claude/skills/<name>/SKILL.md` (project-local, checked in with the repo) unless the finding is genuinely cross-project (then it goes in `~/.claude/skills/<name>/SKILL.md`).
- Use this format:
  ```
  ---
  name: <name>
  description: <when-to-trigger and what-it-does, with concrete trigger phrases. Be slightly pushy — Claude tends to under-trigger skills, so spell out the cases where it *should* fire even if the user doesn't name it explicitly.>
  ---

  # <Title>

  ## When to use
  - <trigger phrases and contexts>

  ## Workflow
  ### Step 1: ...
  ### Step 2: ...

  ## Common failure modes (optional but valuable)
  | Symptom | Cause | Fix |

  ## Reference
  - <links to relevant docs, ADRs, memory files>
  ```
- Keep SKILL.md under 500 lines. If a section is large (e.g., a long table of options), put it in a sibling file under `references/` and link from SKILL.md.
- Mirror the patterns of any existing project-local skills (under `.claude/skills/`) for consistency.

**Skill update:**
- Read the existing SKILL.md first.
- Make the smallest surgical edit that captures the finding. Update the description if the scope materially expands. Add a step, a row to the failure-modes table, or a rule — don't rewrite working sections.

**Memory entry:**
- Follow the auto-memory system's two-step process — write the memory file, then add a one-line pointer to `MEMORY.md`. Don't put memory content directly in `MEMORY.md`.
- For feedback memories, structure the body as `<rule>` then `**Why:**` then `**How to apply:**` so future sessions can judge edge cases.

### Step 5: Report

Tell the user what you did:

```
Created:
- .claude/skills/<name>/SKILL.md — <one-line>

Updated:
- .claude/skills/<name>/SKILL.md — <one-line>
- memory/<file>.md — <one-line>

Discarded:
- <one-line>
```

If you created or updated a project-level skill, mention that it's checked in and will be available to anyone cloning the repo. If you wrote to memory, mention that it'll auto-load next session.

Don't commit anything yourself unless the user asks — these are working-tree changes that the user normally reviews before a feature commit.

## Anti-patterns

- **Don't capture the obvious.** "Use git to commit code" is not a finding.
- **Don't capture transient state.** "We're currently on branch feature/X" is not durable.
- **Don't write skills nobody will trigger.** A skill the description doesn't surface clearly is dead weight. If you can't write a description that names concrete trigger phrases, the finding probably isn't ready for a skill yet — keep it as memory or discard.
- **Don't fragment.** If you're tempted to make three skills that share a domain, make one skill with three sections instead.
- **Don't preempt the user.** Step 3 (draft proposals, ask) is non-negotiable on first run. After they've used the skill a few times, you can compress to a single confirmation step if they say so — but never silently write to disk.

## Reference

- Existing project-local skills under `.claude/skills/` (if any) — read them to match the project's skill style.
- Auto-memory system: described in the Claude Code harness instructions. Storage path is derived from the project's absolute path with `/` replaced by `-`: e.g. project `/Users/me/Projects/foo` → `~/.claude/projects/-Users-me-Projects-foo/memory/`.
- `skill-creator` (built-in): use this as a fallback if a finding genuinely needs the full draft → test → iterate loop. For most session findings, the lighter-weight flow above is enough.
