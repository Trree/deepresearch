---
name: goal-driven-agent
description: >-
  Goal-driven execution protocol with a bounded scout and external-oracle
  improvement loop. Phase 1: decompose + execute to baseline. Phase 2: scout
  the landscape once, build a problem map, then attack the weakest open problem
  with external oracles until all meaningful problems are improved or saturated.
  Use when: "目标驱动", "GDA", "帮我拆解", "按目标执行".
license: MIT
---

# Goal-Driven Agent

Read `references/examples.md` for worked examples.

## Core Principle

Every IMPROVE round must bring in information the agent did not previously have.
No new information for a problem means that problem is saturated, not that the
entire task is globally done.

Agent self-evaluation is not an oracle. External processes are.

## Two Phases

```text
Phase 1: EXECUTE
  Get a baseline that satisfies the explicit task.

Phase 2: IMPROVE
  Scout once -> build a problem map -> attack the weakest open problem ->
  keep or saturate -> repeat -> stop when all meaningful problems are done
  or saturated.
```

## State File: goal-state.md

Create at DECOMPOSE. Update at every transition. Read first on every turn.

```text
phase: IMPROVE
round: 3
branch: gda/<tag>
best_commit: d4e5f6g
oracle_type: test_runner | benchmark | linter | web_search | paper_read | code_read

| id | goal | verify_cmd | expect | type | status |
|----|------|------------|--------|------|--------|
| G1 | ...  | `cmd`      | ...    | hard | done   |

## scout log
- scan 1: broad checker/query -> what it revealed
- scan 2: broad checker/query -> what it revealed

## problem map
| rank | aspect | current | gap | status | notes |
|------|--------|---------|-----|--------|-------|
| 1 | ... | ... | ... | open | why it is rank 1 |
| 2 | ... | ... | ... | open | ... |
| 3 | ... | ... | ... | saturated | round 4, no new info |

## improve log
- round 1: target = "..."; new info = "..."; action = keep [commit]
- round 2: target = "..."; new info = "none"; action = saturate
```

Status values:
- `open`: not yet attacked
- `targeting`: current round
- `done`: gap is materially closed
- `saturated`: oracle produced no materially new information for this problem

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

Two oracle families:

```text
Family A: execution oracles
  Examples:
  - test suite
  - benchmark
  - linter
  - compiler

Family B: information oracles
  Examples:
  - web search
  - paper search
  - codebase reading
  - dataset inspection
```

### Why There Is A Scout Step

Weakest-part optimization only works after you have a usable map of the space.
Without that map, "weakest" is often just "most salient right now."

But scout is not exhaustive research.

Scout exists to:
- discover the main dimensions of the problem
- surface obvious blind spots
- rank candidate weaknesses

Scout does not exist to:
- collect everything
- read every source
- fully solve the task before ranking

Rule: `Scout for map, not for completeness.`

### Step 1: SCOUT ONCE, WITH A BUDGET

Run a bounded breadth-first sweep. Budget: at most 3 broad oracle calls.

Family A:
- Run the broadest checkers that expose the shape of the problem.
- Typical set: tests, coverage, linter, compiler, benchmark.
- Use up to 3 that give the widest picture.

Family B:
- First do an internal scan of the current deliverable:
  - claims with no evidence
  - shallow sections
  - missing counterarguments
  - missing comparable systems
- Then do up to 3 broad external queries to map the space.

Good scout queries are broad and structural, for example:
- timeline / history / canonical overview
- limitations / criticism / failures
- comparable systems / alternative approaches

Do not try to exhaust the literature in scout.

### Step 2: BUILD THE PROBLEM MAP

Rank the most important weaknesses found in scout.

Write to `goal-state.md`:

```text
## problem map
| rank | aspect | current | gap | status | notes |
|------|--------|---------|-----|--------|-------|
| 1 | {worst} | {metric/description} | {what is missing} | open | {why now} |
| 2 | ... | ... | ... | open | ... |
| 3 | ... | ... | ... | open | ... |
```

Ranking heuristics:
- severity: how much this weakens the deliverable
- tractability: whether an oracle can realistically improve it
- leverage: whether fixing it changes multiple downstream sections

Default attack order: highest severity first.
Override only if the top item is clearly blocked and the reason is explicit.

### Step 3: WEAKEST-PART LOOP

Maximum 10 rounds unless the user requests more.

For each round:

```text
1. Read goal-state.md.
2. Pick the highest-ranked OPEN problem.
3. Mark it TARGETING.
4. Run one focused oracle call against that problem.
5. Judge the result.
```

Focused oracle calls:

Family A:
- make one targeted change
- run the checker that measures that problem
- the changed checker output is the new information

Family B:
- search externally for evidence specific to that problem
- the found source, fact, quote, counterexample, or comparison is the new information

### Step 4: JUDGE THE ROUND

If the oracle produced materially new information:

```text
1. Incorporate it.
2. git add -A && git commit -m "improve round {n}: {what was found}"
3. Update best_commit.
4. Re-check the affected problem.
5. If the gap is materially closed -> mark DONE.
6. If the gap is smaller but still important -> keep it OPEN and re-rank.
7. Continue.
```

If the oracle produced no materially new information:

```text
1. Revert to best_commit if needed.
2. Mark this problem SATURATED.
3. Record why it saturated.
4. Continue to the next OPEN problem.
```

One miss does not stop the whole loop.
It only stops work on that problem.

### Step 5: STOP

Stop when any of these is true:

```text
1. All meaningful problems are DONE or SATURATED.
2. A confirmation re-scan finds no newly rankable problem.
3. Max rounds reached.
4. User interrupts.
```

Do not stop merely because one problem saturated.
Do stop when the map itself has stopped yielding actionable open problems.

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

### CONVERGED

Print:

```text
## GDA Complete
Phase 1: {n} goals, all done
Phase 2: {m} rounds
Oracle type: {type}

Scout summary:
- scan 1: ...
- scan 2: ...
- scan 3: ...

Problem map outcome:
| rank | aspect | result |
|------|--------|--------|
| 1 | ... | done (round X) |
| 2 | ... | saturated (round Y) |
| 3 | ... | done (round Z) |

New information incorporated:
- round 1: ...
- round 2: ...

Saturated problems:
- ...

Human review needed:
- ...
```

Update `goal-state.md`. Done.
