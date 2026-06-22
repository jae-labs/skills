---
name: grill-me
description: Stress-test a design or plan through relentless one-question-at-a-time interrogation. Walk down each branch of the decision tree, resolving dependencies one-by-one. Use when the user wants to stress-test a plan, get grilled on their design, or mentions "grill me".
---

# Grill Me

Interview the user relentlessly about every aspect of a design or plan until every branch of the decision tree has a resolution. This skill is for **after** brainstorming has produced a design — it stress-tests what exists, it does not create something new.

## When to Use

- The user has a design, plan, or spec and wants it challenged
- Before committing to implementation — catching ambiguity now is cheaper than rework later
- The user invokes "grill me", "stress-test this", "challenge this plan", "are we sure?"
- After brainstorming, when the user says they want deeper challenge (the "Need deeper challenge?" branch in the AIgile flow)

**When NOT to use:**

- The ask is still underspecified and no design exists yet — use brainstorming instead
- The user hasn't produced any artifact to grill against
- Pure information requests ("how does X work?")
- Mechanical operations where requirements are unambiguous
- The user has explicitly asked for speed over verification

## The Process

### Step 1: State Your Opening Hypothesis

Before asking anything, write down your current read of the design/plan's strength and your confidence in it:

```
HYPOTHESIS: This design handles the happy path well but has three unresolved edge cases around auth state transitions.
CONFIDENCE: ~50% — haven't examined: error recovery flow, concurrent access, data migration path
```

The number forces honesty. If you can't articulate what might be wrong, you're not ready to grill — read the design again.

When confidence is below ~70%, append what's still unresolved. This tells the user exactly where the gaps are.

### Step 2: Ask One Question at a Time, Each with a Guess

Format:

```
Q:     What happens when the payment service is down mid-transaction?
GUESS: You probably have a retry queue with exponential backoff, because that's the standard pattern — but I don't see it in the design.
```

Wait for the user to react before asking the next question.

**Why one at a time:**
- The user can't react to your hypotheses if you bury them in a list
- Batches encourage skim-reading and surface answers
- Later questions often depend on earlier answers
- The user's energy for careful thinking is finite; spend it one question at a time

**Why attach a guess:**
- The user reacts faster to a wrong guess than they generate an answer from scratch
- It commits you to a hypothesis you can be visibly wrong about
- It surfaces *your* assumptions about the design, which may reveal gaps the designer didn't notice

### Step 3: Walk the Decision Tree

Don't jump between topics. Follow one branch of the decision tree to its conclusion before moving to the next. For each branch:

1. **Open the branch** — state the decision point and your guess for how it resolves
2. **Listen** — the user's answer may open sub-branches you didn't anticipate
3. **Resolve** — don't move on until this branch has a concrete resolution (not "we'll figure it out later")
4. **Close the branch** — briefly restate the resolution before moving on

Example tree for an API design:
```
Authentication
├── Token format → resolved: JWT with RS256
│   ├── Refresh strategy → resolved: rotating refresh tokens
│   └── Token expiry → resolved: 15min access, 7day refresh
├── Rate limiting → resolved: per-user, sliding window
└── Error responses → UNRESOLVED — what code for expired vs invalid?
```

### Step 4: Probe for "Want vs Should-Want"

The most dangerous answers are where the user says what a thoughtful answer *sounds like* rather than what they actually decided. Watch for:

- Answers that pattern-match best-practice talk without specifics ("it should be idempotent", "use clean architecture")
- Answers that defer to convention ("the way most APIs do it", "the standard approach")
- Phrases like "I should probably…", "I think we're supposed to…"
- Buzzwords as answers — "resilient", "scalable", "production-grade" without concrete meaning in this context

When you hear these, ask:

> *"If you didn't have to justify this to anyone, what would you actually decide?"*

### Step 5: Codebase Grounding

If a question can be answered by exploring the codebase, **explore the codebase instead of asking**. This is faster for the user and often more accurate — the code doesn't rationalize.

When you find code that contradicts the design, flag it:

```
The design says "all errors return structured JSON," but `src/handlers/errors.ts` has three places that throw raw strings. Which is the source of truth?
```

### Step 6: Confirm Resolutions

As you resolve branches, maintain a running summary. After every 3-5 resolved branches, restate what was decided:

```
Resolved so far:
- Auth: JWT with RS256, rotating refresh tokens
- Rate limiting: per-user sliding window, 100 req/min
- Error format: structured JSON with error code + message + request_id

Still unresolved:
- Error codes for expired vs invalid tokens
- Whether rate limit headers are included in 429 responses
```

This prevents drift — the user can spot contradictions between early and late resolutions.

### The Stop Condition

You're done when you can answer yes to **both**:

1. *Can I predict the user's answer to the next three questions I would ask?* (shared understanding)
2. *Does every branch of the decision tree have a concrete resolution?* (completeness)

If you've gone several rounds and still can't predict, stop and say: *"I've asked X questions and I still can't predict your reactions on [topic]. Something foundational is missing. Want to step back and reframe?"*

Don't grind past the point of diminishing returns. If the remaining branches are genuinely low-risk, say so and move on.

## Interaction with Other Skills

- **brainstorming**: upstream. Grill-me stress-tests a design that brainstorming produced. If grilling reveals fundamental problems, go back to brainstorming to rework the design.
- **triage**: downstream. Once grilling resolves all branches, the design is ready for triage to produce the agent-ready brief.
- **writing-plans**: two steps downstream (after triage). Don't skip triage after grilling.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The design is solid, we don't need to grill it" | If you can't predict the user's answer to the next three questions about it, there are gaps you haven't found. |
| "Asking too many questions wastes their time" | Time wasted by 4-6 targeted questions is small. Time wasted by implementing an ambiguous design is enormous. |
| "We'll figure it out during implementation" | "We'll figure it out later" means "we'll figure it out under time pressure, in code, with switching costs." That's the expensive version. |
| "They said 'it depends,' so I should move on" | "It depends" means the decision tree has an unresolved branch. Push for a concrete answer or flag it as an open risk. |
| "I don't want to be annoying" | You're not being annoying — you're being thorough. The user asked to be grilled. One question at a time with a guess attached is respectful, not aggressive. |
| "The codebase contradicts the design, but it's probably fine" | Contradictions between code and design are never "probably fine." They're ambiguity that will bite during implementation. |
| "We've covered enough, I'll just note the rest as open questions" | Open questions are fine if they're genuinely low-risk. If they're in the critical path, they're not open questions — they're unmade decisions. |
| "I'll attach my guess but keep it safe" | Safe guesses don't surface gaps. Be visibly willing to be wrong — that's where the value is. |

## Red Flags

- Three or more questions in a single message: that's batching, not grilling
- A question without your hypothesis attached: that's surveying, not committing
- Accepting "it depends" or "we'll figure it out later" as a resolution
- Jumping between decision branches without resolving any of them
- The user gives a sophistication-signaling answer ("scalable", "idempotent", "clean") and you accept it without probing
- Three or more rounds without resolving a single branch: you're circling, not progressing
- A confidence number below ~70% with no reason attached
- Not maintaining a running summary of resolved vs unresolved branches
- Skipping codebase exploration when the answer might already be in the code
- Moving on from a branch where the resolution is vague or conditional

## Verification

After completing grill-me:

- [ ] An explicit hypothesis with a confidence number was stated at the start
- [ ] Every confidence number below ~70% was accompanied by a one-line reason
- [ ] Questions were asked one at a time, each with the agent's guess attached
- [ ] At least one "what would you actually decide if you didn't have to justify it?" probe ran when the user gave a convention-signaling answer
- [ ] Decision branches were followed to conclusion before moving to new topics
- [ ] A running summary of resolved vs unresolved branches was maintained
- [ ] Codebase exploration was used when answers might already exist in the code
- [ ] Every branch of the decision tree has a concrete resolution (or is explicitly flagged as a low-risk open question)
- [ ] The user confirmed the final state of all resolutions with an explicit yes
- [ ] At the stop point, the agent could predict the user's answer to the next three questions
