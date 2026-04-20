# 下一代 Deep Research 架构

## 核心命题

当前 deep research = 单向只读管道。
下一代 deep research = **自主迭代的研究闭环**。

区别不在于"搜得更多"或"综合得更好"，而在于架构范式的转变：从管道（pipeline）到循环（loop）。

## 整体架构

```
                    ┌─────────────────────────────┐
                    │      Research Loop           │
                    │                             │
 ┌──────────┐      │  ┌────────┐    ┌────────┐  │     ┌──────────┐
 │ 用户     │─────►│  │ 规划器  │───►│ 执行器  │  │────►│ 产出     │
 │ (问题+   │      │  │Planner │◄───│Executor│  │     │ (报告+   │
 │  偏好)   │      │  └───┬────┘    └────┬───┘  │     │  数据+   │
 └──────────┘      │      │              │      │     │  代码)   │
                    │      │    ┌─────┐   │      │     └──────────┘
                    │      └───►│评估器│◄──┘      │
                    │           │Judge│           │
                    │           └──┬──┘           │
                    │              │              │
                    │     keep/revert/pivot       │
                    │              │              │
                    │         ┌────┴────┐         │
                    │         │ 状态存储 │         │
                    │         │ State   │         │
                    │         └─────────┘         │
                    └─────────────────────────────┘
```

## 组件 1: 规划器（Planner）— 动态研究计划

**当前做法**：生成一次研究计划，执行到底。
**下一代**：计划是一个**持续更新的优先队列**。

```
research_queue = PriorityQueue()

初始化：
  从用户问题生成 3-5 个研究子问题
  每个子问题带有: 优先级 + 预估信息增益

每轮循环后：
  1. 从 queue 取出最高优先级的子问题
  2. 执行研究
  3. 根据结果：
     - 发现新子问题 → 入队
     - 子问题已回答 → 标记完成
     - 子问题方向错误 → 丢弃
  4. 重新计算剩余子问题的优先级
```

**与 autoresearch 的对应**：autoresearch 没有显式规划器，agent 隐式决定"接下来试什么"。下一代 deep research 把这个隐式过程显式化为优先队列——因为研究子问题比代码实验更需要方向管理。

### 优先级计算

```
priority(question) = information_gap × feasibility × user_relevance

information_gap:  当前报告中该方面的覆盖度有多低（0-1）
feasibility:     这个问题能通过可用工具回答吗（0-1）
user_relevance:  这个问题与用户核心关切的距离（0-1）
```

## 组件 2: 执行器（Executor）— 多模态研究行动

**当前做法**：只有一种行动——搜索网页+阅读。
**下一代**：多种研究行动，按需选择。

| 行动类型 | 描述 | 适用场景 |
|---------|------|---------|
| **web_search** | 搜索+阅读网页 | 事实查询、观点收集 |
| **paper_read** | 搜索+阅读学术论文 | 前沿技术、方法对比 |
| **code_run** | 写代码+运行+看结果 | 验证声明、生成数据 |
| **data_analyze** | 下载数据集+分析 | 趋势分析、统计验证 |
| **api_call** | 调用 API 获取实时数据 | 市场数据、指标对比 |

**关键区别**：`code_run` 和 `data_analyze` 是当前 deep research 完全没有的。这就是 autoresearch 的核心能力——通过实验产生新知识，而非只综合现有知识。

### 行动选择逻辑

```
对于每个研究子问题：
  1. 这个问题能通过搜索回答吗？→ web_search / paper_read
  2. 这个问题的答案有争议吗？→ code_run 验证
  3. 这个问题需要定量数据吗？→ data_analyze / api_call
  4. 以上都不行？→ 标记为"需要人类输入"
```

## 组件 3: 评估器（Judge）— 自动质量判定

**当前做法**：无。报告生成后直接交给用户，没有质量评估。
**下一代**：每轮循环后自动评估报告质量，决定 keep/revert/pivot。

### 评估指标体系

```
report_score = weighted_sum(
  coverage,        # 研究问题各方面的覆盖度 (0-1)
  citation_rate,   # 事实性断言中有引用支撑的比例 (0-1)
  consistency,     # 报告内部是否有矛盾 (0-1)
  freshness,       # 引用来源的时效性 (0-1)
  depth            # 关键论点的论证层次 (0-1)
)
```

### 自动计算方法

```
coverage:
  预定义研究问题的 N 个子方面（在规划阶段生成）
  检查每个子方面是否在报告中被提及
  coverage = 被提及数 / N

citation_rate:
  计数报告中的事实性断言（非观点、非推理的陈述）
  计数其中有引用标注的
  citation_rate = 有引用的 / 总断言数

consistency:
  提取报告中所有 claim
  检查是否有两个 claim 互相矛盾
  consistency = 1 - (矛盾对数 / 总 claim 对数)
```

这些指标不完美，但**一个粗糙可计算的指标胜过没有指标**。

### 判定逻辑（对应 autoresearch 的 keep/discard）

```
new_score = evaluate(new_report)
old_score = evaluate(old_report)

if new_score > old_score + epsilon:
    keep(new_report)        # 提交新版本
elif new_score < old_score - epsilon:
    revert(old_report)      # 回退到旧版本
else:
    keep_simpler(new, old)  # 质量相近，保留更简洁的
```

## 组件 4: 状态存储（State）— 全持久化

**当前做法**：状态在 context window 中，对话结束即丢失。
**下一代**：所有状态外化到文件系统。

```
research-workspace/
├── report.md            # 当前最佳版本的报告 (git tracked)
├── research-state.md    # 当前阶段、进度、队列状态
├── sources.tsv          # 已读来源: URL | 可信度 | 关键摘要 | 引用次数
├── questions.md         # 研究子问题队列 + 状态
├── experiments/         # code_run 的实验代码和结果
│   ├── exp001/
│   └── exp002/
├── results.tsv          # 每轮迭代记录: 轮次 | 分数 | 状态 | 描述
└── .git/                # 版本控制——可 diff、可 revert、可审计
```

### results.tsv 格式（直接对应 autoresearch）

```
round	score	status	action	description
1	0.35	keep	web_search	initial research on 5 core subtopics
2	0.42	keep	paper_read	added academic sources for claim validation
3	0.40	discard	code_run	experiment failed to reproduce cited benchmark
4	0.51	keep	web_search	found authoritative source resolving contradiction
5	0.51	discard	paper_read	new paper added noise, no new insight
```

## 组件 5: 资源预算（Budget）— 每轮固定

**对应 autoresearch 的 5 分钟固定预算。**

```
per_round_budget:
  max_searches: 5        # 每轮最多 5 次搜索
  max_pages: 20          # 每轮最多读 20 个页面
  max_tokens: 50000      # 每轮最多消耗 50K tokens
  max_time: 3 min        # 每轮最多 3 分钟

超出任一限制 → 停止当前轮 → 评估 → keep/revert → 下一轮
```

这保证了：
- 每轮可预测——用户知道 20 轮大概要多久
- 防止在死胡同里无限深挖
- 不同策略在同等预算下公平对比

## 完整执行循环

```
SETUP:
  1. 解析用户问题
  2. 生成初始研究子问题队列
  3. 初始化 workspace（创建文件结构）
  4. 用户确认方向

LOOP FOREVER:
  1. 读 research-state.md 恢复上下文
  2. 从 questions.md 取最高优先级子问题
  3. 选择行动类型（web_search / paper_read / code_run / ...）
  4. 在预算内执行行动
  5. 更新 report.md
  6. 计算 report_score
  7. if improved → git commit (keep)
     if not → git reset (revert)
  8. 更新 results.tsv、questions.md、sources.tsv
  9. 如果 coverage > 0.9 且 score 连续 3 轮未提升 → 收敛，进入 DONE
  10. 否则 → 回到 1

DONE:
  1. 最终报告质量检查
  2. 标注所有 type: human 的内容
  3. 输出报告 + results.tsv + 研究过程摘要
```

## 与当前产品的关键差异汇总

| 维度 | 当前 deep research | 下一代 |
|------|-------------------|--------|
| 架构 | 单向管道 | 迭代循环 |
| 行动 | 只搜索+阅读 | 搜索+阅读+跑代码+分析数据 |
| 评估 | 无 | 标量指标自动评估 |
| 状态 | context window（易丢失） | 文件系统（持久化） |
| 计划 | 一次性制定 | 动态优先队列 |
| 版本 | 无 | git 管理（可 diff/revert） |
| 预算 | 无（可能无限搜索） | 每轮固定预算 |
| 自主性 | 一次性（做完就停） | 无限循环（直到收敛或被中断） |
