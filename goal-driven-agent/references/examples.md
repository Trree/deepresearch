# Examples

## Example 1: Coding task — Family A oracle (test runner)

User: "修复登录 bug，空邮箱会崩溃"

### Phase 1: EXECUTE
```
G1: 写复现测试    → npm test → exit ≠ 0 (bug exists)     ✅
G2: 修复 bug     → npm test → exit 0                     ✅
G3: 回归         → npm test (all) → exit 0               ✅
BASELINE reached.

Oracle check: 有测试套件 → oracle_type = test_runner → Phase 2.
```

### Phase 2: IMPROVE (oracle = test runner + linter)
```
STEP 1: WIDE SCAN
  npm test --coverage → auth/register.ts 只有 45% 覆盖
  eslint src/auth/ → 2 warnings
  tsc --noEmit → 0 errors

STEP 2: PROBLEM MAP
  | rank | aspect | current | gap | status |
  |------|--------|---------|-----|--------|
  | 1 | coverage | 45% auth/register.ts | 边界 case 未测 | open |
  | 2 | lint | 2 warnings | unused import, any type | open |

STEP 3: WEAKEST-PART LOOP

  Round 1: target rank 1 (coverage 45%)
    ▸ Oracle call: 写 3 个边界 case 测试 → npm test --coverage
    ▸ New info: coverage 45% → 82%。3 个新 case 全过。
    ✅ Round 1: new info = "3 edge cases pass, coverage 45%→82%" [b2c3d4e]
    → rank 1 DONE. Re-rank: lint 成为新的 rank 1.

  Round 2: target rank 1 (lint 2 warnings)
    ▸ Oracle call: 修 lint → eslint
    ▸ New info: 0 warnings.
    ✅ Round 2: new info = "lint clean, 2→0 warnings" [c3d4e5f]
    → rank 1 DONE. Problem map 全部 DONE.

  Round 3: target — 无 open 问题，再跑一次 wide scan 确认
    ▸ npm test --coverage → 82%, all pass
    ▸ eslint → 0 warnings
    ▸ 没有新问题。

STEP 4: STOP — 最弱点无新信息 → CONVERGED.

## GDA Complete
Phase 1: 3 goals, all ✅
Phase 2: 3 rounds (2 keeps, 1 miss → stop)
Oracle: test_runner + linter

Problem map:
| rank | aspect | result |
|------|--------|--------|
| 1 | coverage 45% | ✅ improved → 82% (round 1) |
| 2 | lint 2 warnings | ✅ improved → 0 (round 2) |
| 3 | (none found) | ↩️ no new info → stop |
```

---

## Example 2: Research task — Family B oracle (web search)

User: "得出 deepresearch 的下一代应该是怎样的"

### Phase 1: EXECUTE
```
G1: 研究当前产品局限   → wc -l > 50                      ✅
G2: 提炼可迁移模式     → grep -c "^##" >= 3               ✅
G3: 设计下一代架构     → grep -c "^##" >= 4               ✅
G4: 产出结论文档       → grep "结论" → found              ✅ (human review)
BASELINE reached.

Oracle check: 可以搜索 web 找到文档中没有的信息 → oracle_type = web_search → Phase 2.
```

### Phase 2: IMPROVE (oracle = web search)
```
STEP 1: WIDE SCAN (≤ 3 次搜索，只摸底)
  扫描四份文档，列出所有弱点：
  ▸ 01-current-state.md: 局限分析全是推测，无真实用户反馈
  ▸ 02-patterns.md: 只分析了 autoresearch 一个项目，无对比
  ▸ 03-architecture.md: 评估器设计没有讨论失败模式
  ▸ README.md: 开放问题"多 agent 协作"没有任何外部参考

STEP 2: PROBLEM MAP
  | rank | aspect | current | gap | status |
  |------|--------|---------|-----|--------|
  | 1 | 局限分析 | 全推测 | 无真实用户反馈 | open |
  | 2 | 可迁移模式 | 单一来源 | 只有 autoresearch | open |
  | 3 | 架构风险 | 无风险分析 | 缺失败模式讨论 | open |
  | 4 | 多 agent | 无引用 | 无外部证据 | open |

STEP 3: WEAKEST-PART LOOP

  Round 1: target rank 1 (局限分析无实证)
    ▸ Oracle call: search "OpenAI deep research limitations user feedback reddit"
    ▸ New info: Reddit 帖 — "漏掉关键来源，只搜前 3 页"；
      HN 帖 — "报告质量不稳定，跑两次结果差很多"
    ▸ 纳入 01-current-state.md，新增"真实用户反馈"小节。
    ✅ Round 1: new info = "2 user reports: missing sources + inconsistency" [e5f6g7h]
    → rank 1 DONE. Re-rank: 可迁移模式成为新 rank 1.

  Round 2: target rank 1 (可迁移模式单一来源)
    ▸ Oracle call: search "autonomous AI experiment frameworks 2025 2026"
    ▸ New info: SWE-agent (Princeton)、OpenHands (AllHands)、AIDE (Weco AI)
      都有 "run → evaluate → keep/discard" 循环。
    ▸ 纳入对比分析。
    ✅ Round 2: new info = "3 similar frameworks: SWE-agent, OpenHands, AIDE" [f6g7h8i]
    → rank 1 DONE. Re-rank: 架构风险成为新 rank 1.

  Round 3: target rank 1 (架构缺风险分析)
    ▸ Oracle call: search "Goodhart's law AI agents reward hacking examples"
    ▸ New info: 论文 "Reward Hacking in Autonomous AI" — 3 种 agent
      优化代理指标而非真实目标的案例。
    ▸ 纳入 03-architecture.md 风险小节。
    ✅ Round 3: new info = "reward hacking paper with 3 failure cases" [g7h8i9j]
    → rank 1 DONE. Re-rank: 多 agent 成为新 rank 1.

  Round 4: target rank 1 (多 agent 协作无依据)
    ▸ Oracle call: search "multi-agent research collaboration AI 2026"
    ▸ New info: Google DeepMind "FunSearch" — 多 agent 搜索 + 评估 > 单 agent。
    ✅ Round 4: new info = "FunSearch paper: multi-agent > single-agent" [h8i9j0k]
    → rank 1 DONE. Problem map 全部 DONE.

  Round 5: target — 无 open 问题，oracle 搜 3 个新关键词
    ▸ 没有显著新信息。

STEP 4: STOP — 最弱点无新信息 → CONVERGED.

## GDA Complete
Phase 1: 4 goals, all ✅
Phase 2: 5 rounds (4 keeps, 1 miss → stop)
Oracle: web_search

Problem map:
| rank | aspect | result |
|------|--------|--------|
| 1 | 局限无实证 | ✅ user feedback (round 1) |
| 2 | 模式单一来源 | ✅ 3 frameworks (round 2) |
| 3 | 架构缺风险 | ✅ reward hacking paper (round 3) |
| 4 | 多 agent 无引用 | ✅ FunSearch paper (round 4) |
| 5 | (none found) | ↩️ no new info → stop |

New information incorporated:
  - Round 1: real user feedback (Reddit + HN)
  - Round 2: 3 comparable frameworks
  - Round 3: reward hacking failure cases
  - Round 4: FunSearch multi-agent evidence
Human review: G4 (README.md)
```

---

## Example 3: Task with no oracle → stop at Phase 1

User: "重构 auth 模块，把大函数拆小"

```
Phase 1: EXECUTE
  G1: 拆 handleAuth → 5 个小函数     ✅
  G2: 测试不回归 → npm test exit 0   ✅

Oracle check:
  1. 能跑测试？→ 能，但测试已经全过了，进一步跑不会产生新信息。
  2. 能搜 web 找新信息？→ 不适用，这是纯代码结构任务。
  → 无有效 oracle。Phase 1 完成即停。

"重构完成。测试全过。请审查 diff。"
```

---

## Example 4: Early convergence — weakest point hits wall

User: "给项目加 TypeScript 类型"

```
Phase 1: EXECUTE
  G1: tsconfig.json 配置     ✅
  G2: 核心模块加类型          ✅
  G3: tsc --noEmit exit 0    ✅
BASELINE reached.

Oracle check: tsc + eslint 可以产生新信息 → Phase 2.

STEP 1: WIDE SCAN
  tsc --noEmit → 0 errors
  eslint --ext .ts src/ → 5 warnings (3 any type, 2 missing return type)
  npm test → all pass, coverage 71%

STEP 2: PROBLEM MAP
  | rank | aspect | current | gap | status |
  |------|--------|---------|-----|--------|
  | 1 | coverage | 71% | utils/ 未覆盖 | open |
  | 2 | lint | 5 warnings | 3 any + 2 missing return | open |

STEP 3: WEAKEST-PART LOOP

  Round 1: target rank 1 (coverage 71%)
    ▸ 为 utils/ 写测试 → npm test --coverage
    ▸ New info: coverage 71% → 89%
    ✅ Round 1: new info = "coverage 71%→89%" [a1b2c3d]

  Round 2: target rank 1 (lint 5 warnings)
    ▸ 修 any type + return type → eslint
    ▸ New info: 5 → 0 warnings
    ✅ Round 2: new info = "lint 5→0" [b2c3d4e]

  Round 3: target — 再跑 wide scan
    ▸ tsc → 0, eslint → 0, coverage 89%
    ▸ 尝试把 coverage 推到 90%+ → 写测试 → coverage 还是 89%（剩余是 generated code）
    ↩️ Round 3: no new info for "coverage ceiling"

STEP 4: STOP → CONVERGED.

## GDA Complete
Phase 1: 3 goals, all ✅
Phase 2: 3 rounds (2 keeps, 1 miss → stop)
```

---

## Example 5: goal-state.md during IMPROVE

```
phase: IMPROVE
round: 3
branch: gda/next-gen-deepresearch
best_commit: g7h8i9j
oracle_type: web_search

| id | goal | verify_cmd | expect | type | status |
|----|------|-----------|--------|------|--------|
| G1 | 当前产品局限 | `wc -l 01-current-state.md` | > 50 | hard | ✅ |
| G2 | 可迁移模式 | `grep -c "^##" 02-patterns.md` | >= 3 | hard | ✅ |
| G3 | 下一代架构 | `grep -c "^##" 03-architecture.md` | >= 4 | hard | ✅ |
| G4 | 结论文档 | `grep "结论" README.md` | found | human | ✅ |

## problem map
| rank | aspect | current | gap | status |
|------|--------|---------|-----|--------|
| 1 | 局限实证 | 全推测 | 无用户反馈 | done (round 1) |
| 2 | 多项目对比 | 单一来源 | 只有 autoresearch | done (round 2) |
| 3 | 风险分析 | 无 | 缺失败模式 | done (round 3) |
| 4 | 多 agent | 无引用 | 无外部证据 | targeting |

## improve log
- round 1: search "deep research limitations user feedback" → 2 Reddit/HN posts → 01-current-state.md [e5f6g7h]
- round 2: search "autonomous AI experiment frameworks" → SWE-agent, OpenHands, AIDE → 02-patterns.md [f6g7h8i]
- round 3: search "Goodhart AI reward hacking" → paper with 3 cases → 03-architecture.md [g7h8i9j]
```

---

## Key: why this doesn't spin

Previous version: agent edits → agent scores → agent says "better" → keeps → repeat.
**No new information enters. Closed system. Spin.**

This version: wide scan → problem map → target weakest → search externally → new info or stop.
**New information enters from outside. Open system. Converges when external info exhausted.**

The test: "after this IMPROVE round, does the deliverable contain a fact/source/result
that it didn't contain before?" YES → real improvement. NO → stop.

Stopping rule: first miss on the current weakest problem = stop.
If the weakest part can't be improved, trying lesser problems won't help.
