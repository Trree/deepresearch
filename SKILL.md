---
name: goal-driven-agent
description: >-
  Goal-driven execution protocol with external-oracle improvement loop.
  Phase 1: decompose + execute to baseline.
  Phase 2: run all checkers first, then attack the weakest problem with external
  oracles, forced-switch on miss, rescan to stop.
  Use when: "目标驱动", "GDA", "帮我拆解", "按目标执行".
license: MIT
---

# Goal-Driven Agent

Read `references/examples.md` for worked examples.

## Core Principle

Two kinds of agent judgment:

```text
1. CAN be outsourced to oracle -> MUST outsource.
   (code quality -> run tests; report length -> wc -l)

2. CANNOT be outsourced -> allowed, but with safety net.
   (information relevance; which direction to explore next)
   Safety net: consistency check every 3 rounds + evidence tiebreaker.
```

Every IMPROVE round must bring in information the agent did not previously have.
No new information for a problem means forced switch, not global stop.

Agent self-evaluation is not an oracle. External processes are.

## Seven Core Concepts

| Concept | Meaning |
|---------|---------|
| phase | execute / improve |
| goal | Phase 1 sub-goal |
| verify | goal's verification command |
| oracle | external info source (execution / information) |
| round | IMPROVE loop iteration |
| current_target | the problem attacked this round |
| budget | total round limit (default 10) |

## Two Phases

```text
Phase 1: EXECUTE
  Get a baseline that satisfies the explicit task.

Phase 2: IMPROVE
  Round 1: run all checkers / scan all aspects -> sort worst-first.
  Then loop: attack weakest problem -> keep or forced-switch ->
  rescan after miss -> stop when no actionable target remains or budget hit.
```

## State File: goal-state.md

Create at DECOMPOSE. Update at every transition. Read first on every turn.

```text
phase: IMPROVE
round: 3
branch: gda/<tag>
best_commit: d4e5f6g
oracle_type: test_runner | benchmark | linter | web_search | paper_read | code_read
budget: 10

| id | goal | verify_cmd | expect | type | status |
|----|------|------------|--------|------|--------|
| G1 | ...  | `cmd`      | ...    | hard | done   |

## improve log
- round: 1
  target: "coverage 45% in auth module"
  oracle_call: "add 3 edge-case tests -> npm test --coverage"
  new_info: "coverage 45% -> 82%, 3 edge cases pass"
  action: keep [commit abc1234]
  next_constraint: none

- round: 2
  target: "lint 2 warnings"
  oracle_call: "fix warnings -> eslint"
  new_info: "none"
  action: miss
  next_constraint: must not target "lint 2 warnings" next round
```

---

## Phase 1: EXECUTE

### CLARIFY

1. Restate the task in one sentence.
2. List assumptions.
3. If ambiguity is high, ask the user. If clear, DECOMPOSE.

### DECOMPOSE

For each sub-goal:

```text
G{n}: {what to do}
  expect_diff: {files that change}
  verify_cmd:  {one shell command}
  expect:      {expected output}
  type:        hard | soft | human
```

Verify patterns:
- `exitcode`: `cmd` -> `$? = 0`
- `numeric`: `cmd | extract` -> threshold
- `string`: `cmd` -> output contains `X`

Setup:
- `git checkout -b gda/<tag>`
- write `goal-state.md`
- confirm with user if needed

For research tasks, verify_cmd checks existence, not quality:
- file exists and has content (`wc -l > N`)
- required sections present (`grep` for headings)
Quality control belongs entirely to Phase 2.

### EXECUTE

```text
1. Print: G{n}: {goal}
2. Do the work.
3. git add -A && git commit -m "G{n}: {description}"
4. Run {verify_cmd} and capture output.
5. Judge.
6. PASS -> next goal. FAIL -> REPLAN.
```

### REPLAN

```text
1. Read the verification log.
2. Syntax/typo -> fix, retry. (max 2)
3. Logic error -> analyze, fix, retry. (max 2)
4. Retries exhausted -> revert to best_commit and try a different approach. (max 2 strategies)
5. Strategies exhausted -> BLOCKED. Report to user.
```

### BASELINE

All goals done. Now determine whether an external oracle exists.

```text
1. Can I run tests, benchmarks, linters, compilers, or other checkers that
   produce outputs I cannot predict?
   -> YES -> oracle_type = execution oracle -> Phase 2

2. Can I search the web, read papers, inspect codebases, or gather outside
   evidence that I do not already have?
   -> YES -> oracle_type = information oracle -> Phase 2

3. Neither?
   -> No oracle. Phase 1 is the final result. Stop.
```

---

## Phase 2: IMPROVE

### The Rule

Every round must execute an external action that can return information the
agent did not already have.

### Oracle Families

```text
Family A: execution oracles
  - test suite, benchmark, linter, compiler

Family B: information oracles
  - web search, paper search, codebase reading, dataset inspection
```

### The Loop

Maximum 10 rounds unless the user specifies a different budget.

**Round 1 rule**: run ALL available checkers / scan ALL aspects before making
any changes. Sort results worst-first. Pick the top as the first target.

For each round:

```text
1. Read goal-state.md + improve log. Recover context.
2. Rescan: pick the weakest problem (respecting forced-switch constraint).
3. Run one focused oracle call against that problem.
4. Judge the result (see below).
5. Log to improve log with all 6 fields.
```

Focused oracle calls:

```text
Family A:
  - make one targeted change
  - run the checker that measures that problem
  - the changed checker output is the new information

Family B:
  - search externally for evidence specific to that problem
  - the found source, fact, quote, counterexample, or comparison is the new information
```

### Judge The Round

If the oracle produced materially new information:

```text
1. Incorporate it.
2. git add -A && git commit -m "improve round {n}: {what was found}"
3. Update best_commit.
4. Log: action = keep [commit hash]
5. Log: next_constraint = none
6. Continue.
```

If the oracle produced no materially new information:

```text
1. Revert to best_commit if needed.
2. Log: action = miss
3. Log: next_constraint = must not target "{problem}" next round
4. Do a fresh rescan of the deliverable.
5. If no actionable target exists OTHER THAN last_miss_target -> stop.
6. Otherwise continue with the next best target.
```

### Forced Switch

One rule, zero extra concepts:

```text
If the last round targeted problem X and got no new info,
this round MUST NOT target X.
```

The `next_constraint` field in the improve log makes this explicit and
auditable. A missed target may become actionable again in a later round
if new information from another target changes the landscape.

### Consistency Check

For information oracles only. Every 3 rounds:

```text
Re-read the full deliverable and check for contradictory claims.
If found, resolving the contradiction becomes the next target.
```

Evidence tiebreaker for resolving contradictions:

```text
Classify evidence as:
  primary    - peer-reviewed paper, official documentation, raw data
  benchmark  - reproducible experiment result, public benchmark
  vendor     - vendor white paper, product announcement, corporate blog
  anecdotal  - forum post, blog opinion, social media

When claims contradict, prefer higher-trust evidence.
This is not a gate — all evidence enters. It is a tiebreaker
for consistency check resolution.
```

### What Counts As New Information

New information:
- a fact, data point, or source the agent did not have before
- a changed test, benchmark, lint, or compile result
- a newly discovered counterexample
- a real user report, paper finding, implementation pattern, or dataset fact
- a source that materially changes confidence, ranking, or design

Not new information:
- rephrasing existing text
- subjective rescoring by the agent
- reorganizing sections without new evidence
- searching and finding only what the document already said
- rerunning the same checker without a targeted change

### Stop Conditions

```text
1. After a miss, fresh rescan finds no actionable target
   other than last_miss_target -> stop.
2. Budget (max rounds) reached -> stop.
3. User interrupts -> stop.
```

### CONVERGED

Print:

```text
## GDA Complete
Phase 1: {n} goals, all done
Phase 2: {m} rounds, budget was {budget}
Oracle type: {type}

Improve log:
- round 1: target = "..."; action = keep/miss
- round 2: ...

Human review needed:
- ...
```

Update `goal-state.md`. Done.
