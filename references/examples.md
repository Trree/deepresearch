# Examples

## Example 1: Coding task — round 1 runs all checkers, then loop

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
Round 1 (scan by dimension):
  oracle calls:
    - [correctness] npm test -> all pass
    - [coverage] npm test --coverage -> auth/register.ts coverage 45%
    - [lint] eslint src/auth/ -> 2 warnings (unused import, any type)
    - [type safety] tsc --noEmit -> 0 errors
  sorted worst-first: coverage 45% > lint 2 warnings > tsc 0 errors
  target: "coverage 45% in auth module"
  oracle_call: "add 3 edge-case tests -> npm test --coverage"
  new_info: "coverage 45% -> 82%, 3 edge cases pass"
  action: keep [commit abc1234]
  next_constraint: none

Round 2:
  target: "lint 2 warnings"
  oracle_call: "fix unused import + any type -> eslint"
  new_info: "warnings 2 -> 0"
  action: keep [commit def5678]
  next_constraint: none

Round 3:
  rescan: coverage 82% (remaining = generated code), eslint 0, tsc 0
  no actionable target -> stop
```

## Example 2: Research task — round 1 scans all aspects

User: "What should the next generation of deep research systems look like?"

### Phase 1: EXECUTE

```text
G1: summarize current products     -> wc -l > 20 ✓
G2: extract recurring patterns     -> grep "## Patterns" ✓
G3: propose next-gen architecture  -> grep "## Architecture" ✓
G4: produce a final report         -> grep "## Conclusion" ✓
BASELINE reached.

Oracle check:
web search and paper search can produce new evidence
-> oracle_type = information oracle
-> Phase 2
```

### Phase 2: IMPROVE

```text
Round 1 (scan by dimension):
  internal scan of deliverable:
    - [evidence] current-state section lacks user evidence
    - [coverage] patterns section has too few comparables (only 2)
    - [boundary] architecture section lacks failure modes
    - [counter] no counterarguments anywhere
  broad queries:
    - "deep research product overview 2025 2026" -> confirms main players
    - "deep research limitations criticism user feedback" -> reliability and citation quality are recurring complaints
    - "autonomous research agent frameworks papers" -> surfaces 3 comparable frameworks
  sorted worst-first: user evidence (none) > comparables (too few) > failure modes (none)
  target: "no user evidence"
  oracle_call: search "deep research user experience reports problems"
  new_info: 2 real user reports about missing sources and hallucinated citations
  action: keep [commit aaa1111]
  next_constraint: none

Round 2:
  target: "too few comparables"
  oracle_call: search "research agent framework architecture comparison"
  new_info: 3 external frameworks (STORM, chain-of-research, autoresearch) with loop structures
  action: keep [commit bbb2222]
  next_constraint: none

Round 3:
  consistency check triggered (every 3 rounds, information oracle):
    re-read full deliverable
    contradiction found: report claims "all products use single-pass generation"
      but round 2 added frameworks that use iterative loops
    -> resolving contradiction becomes next target
  target: "contradictory claim about generation approach"
  oracle_call: search "deep research iterative vs single pass architecture"
  new_info: primary source (official docs) confirms iterative approach in 3 of 5 products
  evidence classification: primary > anecdotal (original claim was from a blog post)
  -> fix deliverable to reflect iterative approach
  action: keep [commit ccc3333]
  next_constraint: none

Round 4:
  target: "no failure modes"
  oracle_call: search "autonomous research agent failure taxonomy reward hacking"
  new_info: paper with 4 concrete failure modes (confirmation bias, source fabrication, loop divergence, reward hacking)
  action: keep [commit ddd4444]
  next_constraint: none

Round 5:
  rescan: all identified gaps now have evidence
  no actionable target -> stop
```

Key points in this example:
- Round 1 scans ALL aspects before picking a target
- Round 3 shows consistency check triggering at the 3-round mark
- Evidence tiebreaker resolves the contradiction (primary > anecdotal)
- Phase 1 verify_cmd checks existence only (`wc -l`, `grep`), not quality

## Example 3: Forced switch + miss then rescan finds another target

User: "Improve this benchmark report."

```text
Round 1 (scan by dimension):
  internal scan:
    - [evidence] missing competitor data
    - [boundary] weak methodology section
    - [boundary] missing limitations
  sorted worst-first: competitor data > methodology > limitations
  target: "missing competitor data"
  oracle_call: search for public benchmark data on competitors
  new_info: none — no trustworthy public benchmark found
  action: miss
  next_constraint: must not target "missing competitor data" next round
  gap marked in deliverable: "## Competitor Benchmarks\n> Data gap: no trustworthy
    public benchmark found as of this date. Numbers below are partial,
    sourced from vendor press releases only."

  fresh rescan: methodology and limitations are still actionable
  -> continue

Round 2:
  forced-switch active: cannot target "missing competitor data"
  target: "weak methodology section"
  oracle_call: search "standard benchmark methodology protocol"
  new_info: official benchmark protocol document found
  action: keep [commit eee5555]
  next_constraint: none

Round 3:
  target: "missing limitations"
  oracle_call: search "benchmark comparison caveats known issues"
  new_info: 2 known caveats from vendor docs
  action: keep [commit fff6666]
  next_constraint: none

Round 4:
  target: "missing competitor data" (forced-switch expired, re-eligible)
  oracle_call: search with different query strategy "competitor performance claims press release"
  new_info: vendor press release with partial benchmark numbers
  action: keep [commit ggg7777]
  next_constraint: none

Round 5:
  rescan: no actionable target -> stop
```

Key points:
- Round 1 miss triggers forced switch (cannot re-target immediately)
- Miss explicitly marks the gap in the deliverable (reader sees what's missing and why)
- After the miss, fresh rescan confirms other targets exist -> continue
- Round 4: the previously missed target becomes attackable again with new angle
- Forced switch is temporary — a missed target can be re-attacked later with a new angle

## Example 4: Miss then rescan finds NO other target -> stop

User: "Polish this CLI help text."

```text
Round 1 (all checkers first):
  - wc -l -> 45 lines, reasonable
  - grep for inconsistent flags -> 1 inconsistency found
  - spell check -> 0 typos
  target: "inconsistent flag format"
  oracle_call: fix --verbose vs -verbose -> run help formatter
  new_info: formatter now shows consistent output
  action: keep [commit hhh8888]
  next_constraint: none

Round 2:
  rescan: no obvious issue
  target: "could add examples section"
  oracle_call: check similar CLIs for example patterns -> search
  new_info: none — help text already follows standard conventions
  action: miss
  next_constraint: must not target "examples section" next round

  fresh rescan: no actionable target OTHER THAN "examples section"
  -> stop
```

Key point:
- After a miss, rescan finds nothing else actionable -> immediate stop
- No wasted rounds cycling through dead targets

## Example 5: No oracle, so stop at Phase 1

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
round 1: run all checkers -> sort worst-first ->
  focused oracle call -> keep or forced-switch ->
  rescan after miss -> stop when nothing actionable
```

The test is simple:

```text
After this round, does the deliverable contain a fact, source, result,
or checker output it did not contain before?
```

If yes, the round was real.
If no, forced switch — and if nothing else to target, stop.
