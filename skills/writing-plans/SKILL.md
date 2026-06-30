---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code. Mandatory after approved spec/triage for substantial implementation work; do not skip straight to coding.
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

## Pre-requisites Check

Before writing the implementation plan, you MUST verify:
1. **Triage State:** The target features/bugs are fully triaged with an active, approved **Agent Brief** (see `triage` skill) detailing category, acceptance criteria, and out-of-scope boundaries. If no brief exists, stop and run the `triage` skill first.
2. **Architecture Decisions (ADRs):** Any major architectural decisions (e.g. storage engine, frameworks, protocol changes) are recorded as ADRs (Architecture Decision Records) in `docs/adr/`. If missing but required, write the ADR first.

**Context:** If working in an isolated worktree, it should have been created via the `superpowers:using-git-worktrees` skill at execution time.

**Save plans to:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (User preferences for plan location override this default)

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## The Planning Process

### Step 1: Read-Only Mode

Before writing any tasks, operate in read-only mode:

- Read the spec and relevant codebase sections
- Identify existing patterns and conventions
- Map dependencies between components
- Note risks and unknowns

**Do NOT write code during planning.** The output is a plan document, not implementation.

### Step 2: Identify the Dependency Graph

Map what depends on what before defining tasks. This determines implementation order:

```
Database schema
    │
    ├── API models/types
    │       │
    │       ├── API endpoints
    │       │       │
    │       │       └── Frontend API client
    │       │               │
    │       │               └── UI components
    │       │
    │       └── Validation logic
    │
    └── Seed data / migrations
```

Implementation order follows the dependency graph bottom-up: build foundations first.

If you skip this step, you'll order tasks by perceived importance instead of by what must exist before what, and later tasks will reference things that haven't been built yet.

### Step 3: Slice Vertically

Instead of building all the database, then all the API, then all the UI — build one complete feature path at a time:

**Bad (horizontal slicing):**
```
Task 1: Build entire database schema
Task 2: Build all API endpoints
Task 3: Build all UI components
Task 4: Connect everything
```

**Good (vertical slicing):**
```
Task 1: User can create an account (schema + API + UI for registration)
Task 2: User can log in (auth schema + API + UI for login)
Task 3: User can create a task (task schema + API + UI for creation)
Task 4: User can view task list (query + API + UI for list view)
```

Each vertical slice delivers working, testable functionality. The system is never in a broken intermediate state.

### Step 3.5: Build a Requirements Traceability Map

Before writing tasks, map the approved request to the plan so execution cannot drift:

```markdown
## Requirements Traceability

| Requested behavior / surface | Planned task(s) | Verification |
|-----------------------------|-----------------|--------------|
| `/login` is the primary entrypoint | Tasks 1, 2 | Playwright login flow |
| Admin portal can manage teams | Tasks 3, 4 | Integration test + browser check |
| Cards can be created and moved | Tasks 5, 6 | Unit + e2e drag/drop |
```

If you cannot fill in this table, the plan is not ready. Missing rows are a stop-the-line issue.

For user-facing apps, include a route/surface map:

```markdown
## Route And Surface Map
- `/login` — real auth surface, not a marketing page
- `/admin` — real management surface, not a static preview
- `/boards/:id` — real board interactions, not decorative cards
```

Do not let "polish", "modern UI", or "production-like" replace this mapping.

### Step 4: Map File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Task Sizing

| Size | Files | Scope | Example |
|------|-------|-------|---------|
| **XS** | 1 | Single function or config change | Add a validation rule |
| **S** | 1-2 | One component or endpoint | Add a new API endpoint |
| **M** | 3-5 | One feature slice | User registration flow |
| **L** | 5-8 | Multi-component feature | Search with filtering and pagination |
| **XL** | 8+ | **Too large — break it down further** | — |

If a task is L or larger, break it into smaller tasks. Agents perform best on S and M tasks.

**When to break a task down further:**
- It would take more than one focused session (roughly 2+ hours of agent work)
- You cannot describe the acceptance criteria in 3 or fewer bullet points
- It touches two or more independent subsystems (e.g., auth and billing)
- You find yourself writing "and" in the task title (a sign it is two tasks)
- More than 5 steps are needed to implement it

## Bite-Sized Step Granularity

Within each task, steps are one action (2-5 minutes):

- "Write the failing test" — step
- "Run it to make sure it fails" — step
- "Implement the minimal code to make the test pass" — step
- "Run the tests and make sure they pass" — step
- "Commit" — step

Every step has the actual code or command. No "then implement the feature" — show the code.

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Acceptance criteria:**
- [ ] [Specific, testable condition]
- [ ] [Specific, testable condition]

**Dependencies:** [Task numbers this depends on, or "None"]

**Estimated scope:** [XS/S/M]

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## Checkpoints

Insert review checkpoints between major phases of the plan:

```markdown
## Checkpoint: After Tasks 1-3
- [ ] All tests pass: `npm test`
- [ ] Application builds without errors: `npm run build`
- [ ] Core user flow works end-to-end
- [ ] Review with human before proceeding
```

Arrange tasks so that:

1. Dependencies are satisfied (build foundation first, per the dependency graph)
2. Each task leaves the system in a working state (vertical slicing ensures this)
3. Checkpoints occur after every 2-3 tasks
4. High-risk tasks are early (fail fast)

## Risks and Mitigations

After the task list, include a risks section:

```markdown
## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| [Risk description] | High/Med/Low | [Strategy to address it] |
| [Risk description] | High/Med/Low | [Strategy to address it] |

## Open Questions
- [Question needing human input before implementation starts]
- [Assumption that needs validation]
```

If there are no open questions, say "None" — don't skip the section.

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task
- User-facing placeholders disguised as scope: "landing page for now", "preview admin portal", "board shell", or "static mock" when the approved request called for a functional route or interaction

## Remember
- Exact file paths always
- Complete code in every step — if a step changes code, show the code
- Exact commands with expected output
- DRY, YAGNI, TDD, frequent commits
- Vertical slices over horizontal layers
- Build foundations first (follow the dependency graph)
- Each task leaves the system in a working, testable state

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll figure it out as I go" | That's how you end up with a tangled mess and rework. 10 minutes of planning saves hours. |
| "The tasks are obvious" | Write them down anyway. Explicit tasks surface hidden dependencies and forgotten edge cases. |
| "Planning is overhead" | Planning is the task. Implementation without a plan is just typing. |
| "I can hold it all in my head" | Context windows are finite. Written plans survive session boundaries and compaction. |
| "This is too small to need a plan" | Small tasks don't need *long* plans, but they still need acceptance criteria and a verification step. A 3-step plan is fine. |
| "I'll add the tricky parts later" | The tricky parts are exactly what needs planning. If you're skipping them, the plan is incomplete. |
| "Horizontal slicing is easier to plan" | Easier to plan, harder to execute. Horizontal slices leave the system broken until the last task connects everything. Vertical slices deliver working software at every step. |

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.

**1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

**2. Placeholder scan:** Search your plan for red flags — any of the patterns from the "No Placeholders" section above. Fix them.

**3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

**4. Dependency order:** Does every task reference only things built in earlier tasks (or things that already exist in the codebase)? If a task depends on something from a later task, the order is wrong.

**5. Vertical slice check:** Does each task deliver working, testable functionality on its own? If a task only makes sense after 3 other tasks are done, it's not a vertical slice.

**6. Sizing check:** Are any tasks L or larger? If so, break them down. Are there tasks with more than 5 acceptance criteria? If so, they're doing too much.

**7. Traceability check:** Can every requested route, surface, and behavior be pointed to in the traceability table? If not, add or fix tasks.

**8. Surface integrity check:** For user-facing apps, does the route map prove that the primary entrypoint and requested surfaces are functional in the planned slice? If not, the plan is drifting.

If you find issues, fix them inline. No need to re-review — just fix and move on. If you find a spec requirement with no task, add the task.

## Red Flags

- Starting implementation without a written task list
- Tasks that say "implement the feature" without acceptance criteria
- No verification steps in the plan
- All tasks are XL-sized
- No checkpoints between tasks
- Dependency order isn't considered
- Tasks that reference types, functions, or modules not defined in earlier tasks
- A task that only makes sense after multiple other tasks are complete (not a vertical slice)
- No requirements traceability table for substantial product work
- No route/surface map for a user-facing app
- Missing risks and mitigations section
- Placeholder text in any step

## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review

## Verification

Before starting implementation, confirm:

- [ ] Every task has acceptance criteria
- [ ] Every task has a verification step (test command, build, manual check)
- [ ] Every step contains actual code or commands — no placeholders
- [ ] Task dependencies are identified and ordered correctly (follows dependency graph)
- [ ] No task touches more than ~5 files
- [ ] Tasks are vertically sliced — each delivers working, testable functionality
- [ ] Requirements traceability covers every requested route, surface, and behavior
- [ ] Route/surface map exists for user-facing work and contains no preview substitutions unless approved
- [ ] Checkpoints exist between major phases
- [ ] Risks and mitigations are documented
- [ ] Open questions are listed (or explicitly "None")
- [ ] The human has reviewed and approved the plan
- [ ] Type and function names are consistent across tasks
