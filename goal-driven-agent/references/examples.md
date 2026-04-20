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
Round 1: FIND WEAKEST
  npm test --coverage → auth/register.ts 只有 45% 覆盖
  eslint src/auth/ → 2 warnings

  ▸ Target: coverage (45%, worst)
  ▸ Oracle call: 写边界 case 测试 → npm test --coverage
  ▸ New info: coverage 从 45% → 82%。3 个新边界 case 全过。
  ✅ Round 1: new info = "3 edge cases pass, coverage 45%→82%" [b2c3d4e]

Round 2: FIND WEAKEST
  eslint → 2 warnings (unused import, any type)

  ▸ Target: lint (2 warnings)
  ▸ Oracle call: 修 lint → eslint
  ▸ New info: 0 warnings.
  ✅ Round 2: new info = "lint clean, 2→0 warnings" [c3d4e5f]

Round 3: FIND WEAKEST
  coverage 82%, all tests pass, lint clean.
  再跑 checker → 没有新问题。
  所有方面 SATURATED → CONVERGED.

## GDA Complete
Phase 1: 3 goals, all ✅
Phase 2: 2 rounds (2 keeps, 0 reverts)
Oracle: test_runner + linter
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
Round 1: FIND WEAKEST
  扫描文档 → 01-current-state.md 的局限分析全是推测，无真实用户反馈。

  ▸ Target: "当前产品局限缺乏实证"
  ▸ Oracle call: search "OpenAI deep research limitations user feedback reddit"
  ▸ New info: 找到 Reddit 帖 — 用户说"deep research 漏掉关键来源，
    只搜前 3 页结果"；HN 帖 — 用户说"报告质量不稳定，同一问题跑两次结果差很多"
  ▸ 纳入 01-current-state.md，新增"真实用户反馈"小节。
  ✅ Round 1: new info = "2 user reports: missing sources + inconsistency" [e5f6g7h]

Round 2: FIND WEAKEST
  扫描 → 02-patterns.md 只分析了 autoresearch 一个项目。

  ▸ Target: "可迁移模式只来自单一来源"
  ▸ Oracle call: search "autonomous AI experiment frameworks 2025 2026"
  ▸ New info: 找到 SWE-agent (Princeton)、OpenHands (AllHands)、
    AIDE (Weco AI)。它们也有类似的 "run → evaluate → keep/discard" 循环。
  ▸ 纳入对比分析。
  ✅ Round 2: new info = "3 similar frameworks: SWE-agent, OpenHands, AIDE" [f6g7h8i]

Round 3: FIND WEAKEST
  扫描 → 03-architecture.md 的评估器设计没有讨论失败模式。

  ▸ Target: "架构缺风险分析"
  ▸ Oracle call: search "Goodhart's law AI agents reward hacking examples"
  ▸ New info: 论文 "Reward Hacking in Autonomous AI" — 列举了 3 种
    agent 优化代理指标而非真实目标的案例。
  ▸ 纳入 03-architecture.md 风险小节。
  ✅ Round 3: new info = "reward hacking paper with 3 failure cases" [g7h8i9j]

Round 4: FIND WEAKEST
  扫描 → 结论中的开放问题"多 agent 协作"没有任何外部参考。

  ▸ Target: "多 agent 协作问题缺乏依据"
  ▸ Oracle call: search "multi-agent research collaboration AI 2026"
  ▸ New info: 找到 Google DeepMind 的 "FunSearch" 论文 — 多 agent
    搜索 + 评估架构确实优于单 agent。
  ✅ Round 4: new info = "FunSearch paper: multi-agent > single-agent" [h8i9j0k]

Round 5: FIND WEAKEST
  各方面扫描:
  ▸ 局限 → 有用户实证 ✓
  ▸ 模式 → 有多项目对比 ✓
  ▸ 架构 → 有风险分析 ✓
  ▸ 开放问题 → 有论文支撑 ✓
  ▸ Oracle call: search 3 个新关键词 → 没有显著新信息。
  所有方面 SATURATED → CONVERGED.

## GDA Complete
Phase 1: 4 goals, all ✅
Phase 2: 5 rounds (4 keeps, 1 saturated)
Oracle: web_search
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

## Example 4: goal-state.md during IMPROVE

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

## aspect saturation
| aspect | status |
|--------|--------|
| 局限实证 | ✅ saturated (round 1) |
| 多项目对比 | ✅ saturated (round 2) |
| 风险分析 | ✅ saturated (round 3) |
| 开放问题 | targeting |

## improve log
- round 1: search "deep research limitations user feedback" → 2 Reddit/HN posts → 01-current-state.md [e5f6g7h]
- round 2: search "autonomous AI experiment frameworks" → SWE-agent, OpenHands, AIDE → 02-patterns.md [f6g7h8i]
- round 3: search "Goodhart AI reward hacking" → paper with 3 cases → 03-architecture.md [g7h8i9j]
```

---

## Key: why this doesn't spin

Previous version: agent edits → agent scores → agent says "better" → keeps → repeat.
**No new information enters. Closed system. Spin.**

This version: agent finds weakest part → searches externally → either finds new info or doesn't.
**New information enters from outside. Open system. Converges when info exhausted.**

The test: "after this IMPROVE round, does the deliverable contain a fact/source/result
that it didn't contain before?" YES → real improvement. NO → stop.
