# Examples

## Example 1: Coding task with scout and saturation

User: "Fix the login bug where an empty email crashes the app."

### Phase 1: EXECUTE

```text
G1: write a failing test     -> npm test -> failure reproduced
G2: fix the bug              -> npm test -> target test passes
G3: regression check         -> npm test -> all pass
BASELINE reached.

Oracle check:
tests and lints can produce outputs the agent cannot predict
-> oracle_type = execution oracle
-> Phase 2
```

### Phase 2: IMPROVE

```text
STEP 1: SCOUT
  - npm test --coverage -> auth/register.ts coverage 45%
  - eslint src/auth/ -> 2 warnings
  - tsc --noEmit -> 0 errors

STEP 2: PROBLEM MAP
  | rank | aspect   | current | gap                  | status | notes          |
  |------|----------|---------|----------------------|--------|----------------|
  | 1    | coverage | 45%     | edge cases untested  | open   | highest risk   |
  | 2    | lint     | 2 warn  | any + unused import  | open   | easy cleanup   |

STEP 3: LOOP

Round 1:
  target = coverage
  oracle call = add 3 edge-case tests -> rerun coverage
  new info = coverage 45% -> 82%
  action = keep
  status = done

Round 2:
  target = lint
  oracle call = fix warnings -> rerun eslint
  new info = warnings 2 -> 0
  action = keep
  status = done

Round 3:
  confirmation re-scan
  - coverage still 82%; remaining uncovered lines are generated code
  - eslint 0
  - tsc 0
  no newly rankable problem
  -> stop
```

## Example 2: Research task with broad scout first

User: "What should the next generation of deep research systems look like?"

### Phase 1: EXECUTE

```text
G1: summarize current products     -> heading exists
G2: extract recurring patterns     -> heading exists
G3: propose next-gen architecture  -> heading exists
G4: produce a final report         -> conclusion exists
BASELINE reached.

Oracle check:
web search and paper search can produce new evidence
-> oracle_type = information oracle
-> Phase 2
```

### Phase 2: IMPROVE

```text
STEP 1: SCOUT
  internal scan:
  - current-state section lacks user evidence
  - patterns section has too few comparables
  - architecture section lacks failure modes

  broad query 1: "deep research product overview 2025 2026"
  -> confirms main players and feature surface

  broad query 2: "deep research limitations criticism user feedback"
  -> suggests reliability and citation quality are recurring complaints

  broad query 3: "autonomous research agent frameworks papers"
  -> surfaces comparable frameworks outside the product layer

STEP 2: PROBLEM MAP
  | rank | aspect            | current         | gap                         | status | notes              |
  |------|-------------------|-----------------|-----------------------------|--------|--------------------|
  | 1    | user evidence     | mostly inference| lacks real reports          | open   | high credibility   |
  | 2    | comparables        | too narrow      | missing adjacent frameworks | open   | high leverage      |
  | 3    | failure analysis   | thin            | no failure taxonomy         | open   | important for trust|

STEP 3: LOOP

Round 1:
  target = user evidence
  oracle call = focused search for user reports
  new info = 2 real reports about missing sources and inconsistency
  action = keep
  status = done

Round 2:
  target = comparables
  oracle call = focused search for related frameworks
  new info = 3 external frameworks with similar loops
  action = keep
  status = done

Round 3:
  target = failure analysis
  oracle call = focused search for reward hacking / failure cases
  new info = paper with concrete failure modes
  action = keep
  status = done

Round 4:
  confirmation re-scan
  no newly rankable open problem
  -> stop
```

## Example 3: Saturating one problem does not stop everything

User: "Improve this benchmark report."

```text
SCOUT -> problem map
  1. missing competitor data
  2. weak methodology section
  3. missing limitations

Round 1:
  target = missing competitor data
  search -> no trustworthy public benchmark found
  action = mark saturated

Round 2:
  target = weak methodology section
  search -> finds official benchmark protocol
  action = keep, mark done

Round 3:
  target = missing limitations
  search -> finds two known caveats from vendor docs
  action = keep, mark done

Round 4:
  re-scan -> no newly rankable problem
  stop
```

Key point:
one miss saturates one problem.
It does not imply global convergence.

## Example 4: No oracle, so stop at Phase 1

User: "Refactor this auth module into smaller functions."

```text
Phase 1:
  G1: split handleAuth into 5 functions
  G2: tests still pass
  BASELINE reached

Oracle check:
  - tests already pass and further test runs are unlikely to create new information
  - web search is not relevant to a local structural refactor
  -> no meaningful oracle

Stop at Phase 1.
```

## Why This Does Not Spin

Bad loop:

```text
agent edits -> agent scores -> agent says "better" -> repeat
```

Good loop:

```text
scout -> problem map -> focused oracle call -> keep or saturate -> re-rank -> stop
```

The test is simple:

```text
After this round, does the deliverable contain a fact, source, result,
or checker output it did not contain before?
```

If yes, the round was real.
If no, the current problem is saturated.
