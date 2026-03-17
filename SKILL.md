---
name: dispatching-programmers
description: Use when you have an implementation plan to execute, dispatching subagents per task - triggers on plan execution, task dispatch, subagent workflow
---

# Subagent-Driven Development

Execute plan by dispatching fresh subagent per task, with two-stage review after each: spec compliance first, then code quality. Every step commits, so session interruption loses at most one task's work.

**Core principles:**
- Fresh subagent per task + two-stage review = high quality, fast iteration
- Every atomic unit of work commits immediately = session recovery possible
- TodoWrite tracks all progress = new session can pick up where old one stopped

## When to Use

- Have implementation plan with mostly independent tasks
- Want to stay in current session (no context switch)
- Tasks can be understood and implemented by a subagent with provided context

## The Process

```
Read plan → Extract all tasks with full text → Create TodoWrite → Record base SHA

For each task:
  1. Dispatch implementer subagent (worktree-isolated)
     - Subagent loads /implement + appropriate language skill
     - /implement runs TDD loop: tests → code → validate → /audit-code (4 agents) → repeat
     - Loop continues until /audit-code returns clean
  2. Answer questions if asked
  3. Merge worktree branch back to feature branch
  4. Mark task complete in TodoWrite

After all tasks:
  5. Dispatch /audit-code across entire implementation (cross-task integration audit)
  6. Verify all tests pass
  7. Present branch completion options
```

## Step 1: Load Plan and Establish Recovery State

1. Read plan file once
2. Extract ALL tasks with full text and context
3. Create TodoWrite with all tasks
4. Note any cross-task dependencies
5. **Record base SHA**: `git rev-parse HEAD` — this is the recovery checkpoint
6. **Check for prior progress**: If resuming after a session interruption, check TodoWrite state and git log to identify which tasks already completed. Skip completed tasks.

**Recovery detection:** If the plan file exists and TodoWrite has tasks marked complete, this is a resumed session. Verify completed tasks by checking git log for their commits, then continue from the first incomplete task.

## Step 2: Per-Task Cycle

**Dispatch implementer subagent** with `isolation: "worktree"`:
- Full task text from plan (don't make subagent read plan file)
- Scene-setting context (where this task fits in the project)
- Clear goal and constraints
- **Skill loading**: subagent MUST load `/implement` + the appropriate language skill from the dispatch table in `/programming`
- **Explicit commit requirement**: subagent MUST commit before returning

```markdown
Implement Task N: [name]

Context: [brief description of where this fits]

[Full task text from plan, verbatim]

Requirements:
- Load /implement and follow its TDD loop
- Load [language skill] for patterns and conventions
- The /implement loop will dispatch /audit-code after each round
- Loop continues until the auditor returns a clean report
- Commit when audit passes with message: "feat(task-N): [description]"
- Write implementation decisions to statement-mcp as handoff messages
- Return summary of what you built, audit rounds needed, and the commit SHA
```

**Worktree isolation:** Each implementer runs in its own worktree. This means:
- Partial work from a crashed subagent doesn't pollute the feature branch
- The worktree can be discarded if the subagent fails
- Successful work is merged back explicitly

**After subagent returns:** Merge its worktree branch back to the feature branch. This is the durable checkpoint — if the session dies after this merge, the task's work is preserved.

**If implementer asks questions:** Answer clearly and completely. Provide additional context. Don't rush them into implementation.

**After merge:** Mark task complete in TodoWrite immediately. The `/implement` loop already ran `/audit-code` until clean — no additional review dispatch needed per task.

## Session Recovery

If the session is interrupted at any point, the next session can recover:

| Interrupted during...        | State on disk                                     | Recovery action                             |
| ---------------------------- | ------------------------------------------------- | ------------------------------------------- |
| Subagent implementing        | Worktree has partial work, feature branch clean   | Discard worktree, re-run task               |
| Merge back to feature branch | Feature branch has commit, TodoWrite not updated  | Check git log, mark task complete, continue |
| Spec/quality review          | Feature branch has commit, task incomplete        | Re-run review only                          |
| Fix subagent after review    | Worktree has partial fix                          | Discard worktree, re-run fix                |
| Between tasks                | TodoWrite accurate, all commits on feature branch | Continue from next incomplete task          |

**Key invariant:** The feature branch only receives complete, tested, committed work. Partial work lives in worktrees that can be safely discarded.

## Step 3: Branch Completion

After all tasks complete:

1. **Cross-task integration audit** — dispatch `/audit-code` across the ENTIRE implementation (all tasks combined). This catches interaction effects between independently-implemented tasks that per-task audits couldn't see.
2. **If audit finds issues** — dispatch fix subagent (worktree-isolated) → fix → re-audit until clean
3. **Verify tests pass** — run full suite, read output, confirm zero failures
4. **If tests fail** — fix before offering options, do not proceed

Present exactly these options:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work
```

**Option 1 — Merge locally:** checkout base, pull latest, merge feature, verify tests on merged result, delete feature branch.

**Option 2 — Create PR:** push branch, `gh pr create` with summary and test plan.

**Option 3 — Keep as-is:** report branch name and path, preserve everything.

**Option 4 — Discard:** require typed "discard" confirmation. Show what will be deleted (branch, commits, worktree). Only then delete.

## Handling Review Feedback

When receiving review feedback from any source:

- **Verify before implementing** — check suggestion against codebase reality
- **Push back if wrong** — technical correctness over social comfort. If suggestion breaks things, lacks context, or violates YAGNI, say so with reasoning.
- **Clarify all items first** — if any feedback is unclear, ask before implementing anything. Partial understanding = wrong implementation.
- **One fix at a time, test each** — don't batch fixes without testing between them
- **No performative agreement** — don't say "Great point!" or "You're absolutely right!" Just fix it or push back.

## Red Flags

**Never:**
- Start implementation on main/master without explicit user consent
- Skip the /implement audit loop (every task gets audited until clean)
- Skip the cross-task integration audit after all tasks complete
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make subagent read plan file (provide full text instead)
- Proceed with failing tests
- Trust agent success reports without independent verification
- Force-push without explicit request
- Leave a task unmarked in TodoWrite after its work is merged
- Let a subagent return without committing (uncommitted work = unrecoverable work)
- Let a subagent skip loading /implement or the language skill

**If subagent asks questions:** Answer before letting them proceed.

**If subagent fails:** Dispatch fix subagent with specific instructions. Don't fix manually (context pollution). Use worktree isolation.

**If cross-task audit finds issues:** Dispatch fix subagent (worktree-isolated) → fix subagent commits → merge back → re-audit. Don't skip re-audit.

**If session may be interrupted:** Every commit on the feature branch is a recovery checkpoint. TodoWrite state plus git log are sufficient to resume from any point.

## When NOT to Use

- Tasks are tightly coupled and can't be understood independently
- No written plan exists (use brainstorming → writing-plans first)
- Single small task (just do it directly)
- Exploratory debugging (use systematic-debugging instead)

## Real-World Impact

From production sessions: 18 tasks across 3 projects (xnn, OnShape, focus_stacking) executed via this workflow. Spec compliance review caught over-building (extra flags not in spec) and under-building (missing progress reporting). Code quality review caught magic numbers and duplicate code. Two-stage review found issues that single-pass review missed — spec compliance ensures correctness, quality review ensures craftsmanship.
