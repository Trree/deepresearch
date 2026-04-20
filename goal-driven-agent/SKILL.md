---
name: goal-driven-agent
description: >-
  Goal-driven execution protocol with external-oracle improvement loop.
  Phase 1: decompose + execute to baseline.
  Phase 2: each round gets NEW external information (run tests, search web,
  read papers), incorporate, keep if better else revert, stop when no new
  info found. State in goal-state.md, checkpoints via git.
  Use when: "目标驱动", "GDA", "帮我拆解", "按目标执行".
license: MIT
---

# Goal-Driven Agent

Read `~/.claude/skills/goal-driven-agent/references/examples.md` for worked examples.

## Core principle

Every IMPROVE round must bring in information the agent did not previously have.
No new information = convergence = stop.
Agent self-evaluation is not an oracle. External processes are.

## Two phases

```
Phase 1: EXECUTE  (get it working — baseline)
Phase 2: IMPROVE  (get new info → incorporate → keep/revert → repeat)
```

## State file: goal-state.md

Create at DECOMPOSE. Update at every transition. Read first on every turn.

```
phase: IMPROVE
round: 3
branch: gda/<tag>
best_commit: d4e5f6g
oracle_type: test_runner | benchmark | linter | web_search | paper_read | code_read

| id | goal | verify_cmd | expect | type | status |
|----|------|-----------|--------|------|--------|
| G1 | ... | `cmd` | `output` | hard | ✅ |

## improve log
- round 1: new info = "Reddit thread: users report missing sources" → added to limitations → keep [commit]
- round 2: new info = "found Agent-E framework" → added comparison → keep [commit]
- round 3: searched 3 queries, nothing new → converged
```

---

## Phase 1: EXECUTE

### CLARIFY

1. Restate task in one sentence.
2. List assumptions.
3. Ambiguity → ask user. Clear → DECOMPOSE.

### DECOMPOSE

For each sub-goal:

```
G{n}: {what to do}
  expect_diff: {files that change}
  verify_cmd:  {one shell command}
  expect:      {expected output}
  type:        hard | soft | human
```

Verify patterns:
- **exitcode**: `cmd` → `$? = 0`
- **numeric**: `cmd | extract` → `< threshold`
- **string**: `cmd` → output contains `"X"`

Setup: `git checkout -b gda/<tag>`. Write goal-state.md. Confirm with user.

### EXECUTE

```
1. Print: ▶ G{n}: {goal}
2. Do the work.
3. git add -A && git commit -m "G{n}: {description}"
4. {verify_cmd} > .gda-verify.log 2>&1; echo "EXIT:$?" >> .gda-verify.log
5. Judge.
6. ✅ → next goal.  ❌ → REPLAN.
```

### REPLAN

```
1. Read .gda-verify.log.
2. Syntax/typo → fix, retry. (≤ 2)
3. Logic error → analyze, fix, retry. (≤ 2)
4. Retries exhausted → git reset --hard {best_commit}, different approach. (≤ 2 strategies)
5. Strategies exhausted → BLOCKED. Report to user.
```

### BASELINE

All goals ✅. Now determine: does this task have an external oracle?

```
Oracle check:

1. Can I run tests/benchmarks/linters that produce output I can't predict?
   → YES → oracle_type = test_runner / benchmark / linter
   → Go to Phase 2.

2. Can I search the web / read papers / read codebases to find
   information I don't currently have?
   → YES → oracle_type = web_search / paper_read / code_read
   → Go to Phase 2.

3. Neither?
   → No oracle. Phase 1 is the final result. Stop. Hand to user.
```

---

## Phase 2: IMPROVE

### The rule

**Every round MUST execute an external action that can return information the agent does not already have.** The external action is the oracle. Without it, the loop is self-talk.

Two oracle families:

```
Family A: EXECUTION oracles (for coding tasks)
  The agent changes code. An external process judges.
  Oracle examples:
    - test suite: npm test → pass/fail count
    - benchmark: bench.sh → latency number
    - linter: eslint → warning count
    - compiler: tsc → error count
  Agent cannot fake these numbers.

Family B: INFORMATION oracles (for research/analysis tasks)
  The agent has a document. External sources provide new knowledge.
  Oracle examples:
    - web search → new articles, user feedback, data
    - paper search → new research, methods, evidence
    - codebase reading → new projects, implementations, patterns
  Agent cannot invent these sources.
```

### IMPROVE loop

```
LOOP (max 10 rounds):

  1. READ goal-state.md. Recover context.

  2. FIND WEAKEST PART of current deliverable:

     Family A (code): run all checkers.
       → Pick the checker with worst result.
       → Example: "coverage lowest at 62% in auth module"

     Family B (research): scan document for:
       → Claims with no external evidence
       → Aspects with shallow coverage
       → Missing perspectives or counterarguments
       → Example: "limitation analysis has no real user feedback"

  3. GO GET NEW INFORMATION (the oracle call):

     Family A: make a change, then run the checker.
       → The checker result IS the new information.
       → Example: add tests → npm test → coverage now 78%

     Family B: search externally for info about the weakest part.
       → The search result IS the new information.
       → Example: search "deep research user complaints" → find Reddit thread

  4. DID YOU GET NEW INFORMATION?

     → YES (found something agent didn't know before):
       Incorporate into deliverable.
       git add -A && git commit -m "improve round {n}: {what was found}"
       Update goal-state.md.
       Print: ✅ Round {n}: new info = "{summary}" → incorporated [commit]
       Go to 1.

     → NO (search returned nothing new / checker didn't improve):
       git reset --hard {best_commit} (if changes were made).
       Print: ↩️ Round {n}: no new information for "{target}"
       Mark this aspect as SATURATED.
       Try next weakest part.
       If ALL aspects saturated → CONVERGED.

  5. CONVERGE CHECK:
     All aspects saturated → stop.
     OR max rounds reached → stop.
     OR user interrupts → stop.
```

### Key: what counts as "new information"

```
IS new information:
  - A fact, data point, or source the agent didn't have before
  - A test result that changed after a code modification
  - A benchmark number from running actual code
  - A real user quote, paper finding, or project example found via search
  - A counterexample or edge case revealed by a checker

IS NOT new information:
  - Agent rephrasing existing content
  - Agent "deciding" the score is higher
  - Agent reorganizing sections
  - Agent adding commentary without new sources
  - Agent running the same test without changing code
```

### CONVERGED

```
Print:
  ## GDA Complete
  Phase 1: {n} goals, all ✅
  Phase 2: {m} improve rounds ({keeps} keeps, {reverts} reverts, {saturated} saturated)
  Oracle type: {type}

  New information incorporated:
  - Round 1: {what was found}
  - Round 2: {what was found}
  - ...

  Saturated aspects (no more new info available):
  - {aspect 1}
  - {aspect 2}

  Human review needed: {list}
```

Update goal-state.md. Done.
