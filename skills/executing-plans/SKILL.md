---
name: executing-plans
description: Use when you have a written implementation plan to execute. Follows the plan task-by-task with incremental delivery, review checkpoints, requirements traceability, and discipline about scope and verification. Do not use without a written plan.
---

# Executing Plans

## Overview

Load plan, review critically, execute tasks incrementally with verification at each step, report when complete.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

**Note:** Tell your human partner that Superpowers works much better with access to subagents. The quality of its work will be significantly higher if run on a platform with subagent support (such as Claude Code or Codex). If subagents are available, use superpowers:subagent-driven-development instead of this skill.

## The Increment Cycle

Each task in the plan follows this cycle:

```
┌──────────────────────────────────────┐
│                                      │
│   Implement ──→ Test ──→ Verify ──┐  │
│       ▲                           │  │
│       └───── Commit ◄─────────────┘  │
│              │                       │
│              ▼                       │
│          Next task                  │
│                                      │
└──────────────────────────────────────┘
```

For each task:
1. **Implement** the smallest complete piece from the plan
2. **Test** — run the test suite (or write a test if none exists)
3. **Verify** — confirm the task works as expected (tests pass, build succeeds, manual check)
4. **Commit** — save progress with a descriptive message
5. **Move to the next task** — carry forward, don't restart

## The Process

### Step 1: Load and Review Plan

1. Read plan file
2. Review critically — identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed
5. Load relevant context: spec sections and source files for the first 2-3 tasks (don't flood context with the entire spec)
6. Create a live traceability checklist from the plan's acceptance criteria and route/surface map. Keep it visible during execution.

### Step 2: Execute Tasks

For each task:
1. Mark as in_progress
2. Restate which requested behaviors, routes, or acceptance criteria this task covers
3. Follow each step exactly (plan has bite-sized steps)
4. Run verifications as specified
5. Mark the corresponding traceability items as completed only after evidence exists
6. Mark as completed
7. Commit with a descriptive message after each completed task

### Step 3: Checkpoint Review

After every 2-3 tasks (or at plan checkpoint markers):
- Run the full test suite
- Confirm the build is clean
- Review what was completed vs what remains
- Discuss any concerns with your human partner before proceeding

### Step 4: Complete Development

After all tasks complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## Implementation Rules

### Rule 0: Simplicity First

Before writing any code, ask: "What is the simplest thing that could work?"

After writing code, review it against these checks:
- Can this be done in fewer lines?
- Are these abstractions earning their complexity?
- Would a staff engineer look at this and say "why didn't you just..."?
- Am I building for hypothetical future requirements, or the current task?

Three similar lines of code is better than a premature abstraction. Implement the naive, obviously-correct version first. Optimize only after correctness is proven with tests.

### Rule 0.5: Scope Discipline

Touch only what the task requires.

Do NOT:
- "Clean up" code adjacent to your change
- Refactor imports in files you're not modifying
- Remove comments you don't fully understand
- Add features not in the plan because they "seem useful"
- Modernize syntax in files you're only reading

If you notice something worth improving outside your task scope, note it — don't fix it:

```
NOTICED BUT NOT TOUCHING:
- src/utils/format.ts has an unused import (unrelated to this task)
- The auth middleware could use better error messages (separate task)
→ Want me to create tasks for these?
```

### Rule 0.75: No Substitute Surfaces

Do not satisfy a requested functional surface with a different, easier, or more decorative one.

- If the request says login page, don't ship a landing page.
- If the request says admin portal, don't ship an admin preview.
- If the request says working board, don't ship static cards with fake drag/drop.

If the approved plan says a route or interaction is functional in the current slice, build that route or interaction. If you need to reduce scope, stop, say exactly which slice you can deliver, and get approval before continuing.

### Rule 1: One Thing at a Time

Each increment changes one logical thing. Don't mix concerns:

**Bad:** One commit that adds a new component, refactors an existing one, and updates the build config.

**Good:** Three separate commits — one for each change.

### Rule 2: Keep It Compilable

After each increment, the project must build and existing tests must pass. Don't leave the codebase in a broken state between tasks.

### Rule 3: Feature Flags for Incomplete Features

If a feature isn't ready for users but you need to merge increments:

```typescript
// Feature flag for work-in-progress
const ENABLE_TASK_SHARING = process.env.FEATURE_TASK_SHARING === 'true';

if (ENABLE_TASK_SHARING) {
  // New sharing UI
}
```

This lets you merge small increments to the main branch without exposing incomplete work.

### Rule 4: Safe Defaults

New code should default to safe, conservative behavior:

```typescript
// Safe: disabled by default, opt-in
export function createTask(data: TaskInput, options?: { notify?: boolean }) {
  const shouldNotify = options?.notify ?? false;
  // ...
}
```

### Rule 5: Rollback-Friendly Commits

Each task should be independently revertable:
- Additive changes (new files, new functions) are easy to revert
- Modifications to existing code should be minimal and focused
- Database migrations should have corresponding rollback migrations
- Avoid deleting something in one commit and replacing it in the same commit — separate them

## Slicing Strategy

The plan should already contain vertically-sliced tasks (from the writing-plans skill). If it doesn't, or if a task feels too large:

**Vertical slicing** means building one complete feature path at a time:

**Bad (horizontal):**
- Task 1: Build entire database schema
- Task 2: Build all API endpoints
- Task 3: Build all UI components

**Good (vertical):**
- Task 1: User can create an account (schema + API + UI)
- Task 2: User can log in (auth + API + UI)
- Task 3: User can create a task (schema + API + UI)

Each vertical slice delivers working, testable functionality.

## Context Management

Don't load the entire spec and all source files at once. Load what's needed for the current task:

1. At session start, load the plan header + first 2-3 tasks
2. Before each task, load relevant spec sections and source files
3. After completing a task, context relevant to that task can be dropped
4. When switching between unrelated tasks, consider a fresh session

When the work moves between design-heavy and implementation-heavy phases, reload the spec objective, success criteria, out-of-scope, and route/surface map before continuing. Context drift on these items causes the most expensive failures.

See `context-engineering` skill for detailed guidance on managing agent context.

## Handling Doubt

When you encounter a non-trivial decision during execution (architectural choice, correct approach under uncertainty, irreversible change), consider using the `doubt-driven-development` skill to subject it to adversarial review before committing.

Non-trivial decisions include:
- Choices that introduce or modify branching logic
- Changes that cross module or service boundaries
- Assertions the type system cannot verify
- Irreversible operations (production deploys, data migrations, public API changes)

For routine decisions that are clearly specified in the plan, just follow the plan.

## Framework-Specific Code

When implementing code that depends on a specific framework or library version, use the `source-driven-development` skill to verify patterns against official documentation instead of relying on training data.

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly (3+ attempts)
- You notice something outside task scope that needs attention
- You realize the current implementation path no longer matches the approved route/surface contract
- You are about to replace a requested behavior with a placeholder, preview, or decorative shell

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 1) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking
- Verification failures reveal a plan-level issue

**Don't force through blockers** — stop and ask.

## Git Discipline

Every code change flows through git. Treat commits as save points, branches as sandboxes, and history as documentation.

### Commit Messages

Format: `<type>: <short description>` followed by an optional body explaining *why*, not *what*.

**Types:** `feat`, `fix`, `refactor`, `test`, `docs`, `chore`.

```
# Good: Explains intent
feat: add email validation to registration endpoint

Prevents invalid email formats from reaching the database.
Uses Zod schema validation at the route handler level,
consistent with existing validation patterns in auth.ts.

# Bad: Describes what's obvious from the diff
update auth.ts
```

### Pre-Commit Hygiene

Before every commit:

1. `git diff --staged` — review what you're about to commit
2. Check for secrets: `git diff --staged | grep -i "password\|secret\|api_key\|token"`
3. Run test suite
4. Run linting and type checking

### Branch Discipline

- Keep `main` always deployable (trunk-based development)
- Feature branches should be short-lived (merge within 1-3 days)
- Delete branches after merge
- Prefer feature flags over long-lived branches for incomplete features

### Change Summaries

After any modification, provide a structured summary:

```
CHANGES MADE:
- src/routes/tasks.ts: Added validation middleware to POST endpoint
- src/lib/validation.ts: Added TaskCreateSchema using Zod

THINGS I DIDN'T TOUCH (intentionally):
- src/routes/auth.ts: Has similar validation gap but out of scope
- src/middleware/error.ts: Error format could be improved (separate task)

POTENTIAL CONCERNS:
- The Zod schema is strict — rejects extra fields. Confirm this is desired.
```

### Handling Generated Files

- Commit generated files only if the project expects them (e.g., `package-lock.json`, Prisma migrations)
- Don't commit build output (`dist/`, `.next/`), environment files (`.env`), or IDE config
- Have a `.gitignore` that covers: `node_modules/`, `dist/`, `.env`, `.env.local`, `*.pem`

### Using Git for Debugging

```bash
# Find which commit introduced a bug
git bisect start
git bisect bad HEAD
git bisect good <known-good-commit>

# View what changed recently
git log --oneline -20
git diff HEAD~5..HEAD -- src/

# Find who last changed a specific line
git blame src/services/task.ts

# Search commit messages
git log --grep="validation" --oneline
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll test it all at the end" | Bugs compound. A bug in Task 1 makes Tasks 2-5 wrong. Test each task. |
| "It's faster to do it all at once" | It *feels* faster until something breaks and you can't find which of 500 changed lines caused it. |
| "These changes are too small to commit separately" | Small commits are free. Large commits hide bugs and make rollbacks painful. |
| "I'll squash it all later" | Squashing destroys the development narrative. Prefer clean incremental commits from the start. |
| "I'll add the feature flag later" | If the feature isn't complete, it shouldn't be user-visible. Add the flag now. |
| "This refactor is small enough to include" | Refactors mixed with features make both harder to review and debug. Separate them. |
| "I'll just quickly add this too" | Scope creep is the enemy of incremental delivery. Note it, don't do it. |
| "The plan is wrong, I'll just fix it" | If the plan has an issue, stop and discuss. Don't silently deviate. |
| "I'll commit when the feature is done" | One giant commit is impossible to review, debug, or revert. Commit each slice. |

## Red Flags

- More than 100 lines of code written without running tests
- Multiple unrelated changes in a single commit
- "Let me just quickly add this too" scope expansion
- Skipping the test/verify step to move faster
- Build or tests broken between tasks
- Large uncommitted changes accumulating
- Building abstractions before the third use case demands it
- Touching files outside the task scope "while I'm here"
- Creating new utility files for one-time operations
- Running the same build/test command twice in a row without any intervening code change
- Deviating from the plan without discussing with your human partner

## Remember
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- One thing at a time, commit after each
- Keep it compilable between tasks
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent
- Simplicity first — naive and correct beats clever and broken
- Scope discipline — only touch what the task requires

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** - Complete development after all tasks

**On-demand skills (triggered by context):**
- **context-engineering** - When starting a new session or switching tasks
- **source-driven-development** - When writing framework-specific code
- **doubt-driven-development** - When encountering non-trivial decisions
- **frontend-ui-engineering** - When building or modifying user-facing interfaces
- **api-and-interface-design** - When designing APIs or module boundaries

## Verification

After completing all tasks:

- [ ] Each task was individually tested and committed
- [ ] Checkpoint reviews were done every 2-3 tasks
- [ ] The full test suite passes
- [ ] The build is clean
- [ ] The feature works end-to-end as specified
- [ ] No uncommitted changes remain
- [ ] No scope creep (only plan-specified work was done)
- [ ] No refactors mixed with feature work
