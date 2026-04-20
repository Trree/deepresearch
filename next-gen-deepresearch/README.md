# 下一代 Deep Research 应该是怎样的

> 当前的 deep research 做的是文献综述（literature review），不是研究（research）。
> 下一代应该能像 autoresearch 一样自主迭代——搜索、验证、实验、评估、保留或回滚——循环到收敛。

## 背景

2024-2025 年，OpenAI、Google、Perplexity 相继推出 "Deep Research" 产品。它们的共同架构是**单向只读管道**：规划 → 搜索 → 阅读 → 综合 → 输出报告。一次性执行，无迭代，无验证，无状态持久化。

与此同时，Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch) 展示了一种截然不同的范式：agent 自主循环修改代码 → 训练 → 看指标 → keep/discard → 重复。一夜跑 100 个实验，人类醒来看结果。

**核心洞见**：autoresearch 的五个设计模式（标量指标、keep/revert、无限循环、状态外化、固定预算）可以直接迁移到 deep research 领域。

## 当前产品的六大局限

详见 [01-current-state.md](01-current-state.md)。

1. **只读不写**——不能通过实验验证结论
2. **无物理闭环**——不能自主迭代加深
3. **无状态持久化**——跨 session 遗忘
4. **无验证机制**——幻觉传播
5. **规划不可修正**——计划锁死后只能执行
6. **单一输出模态**——只产出文本，不产出代码/数据/实验

## autoresearch 的五个可迁移模式

详见 [02-autoresearch-patterns.md](02-autoresearch-patterns.md)。

1. **单一标量指标** → 使自主判断"是否改进"成为可能
2. **Keep-or-Revert** → 棘轮机制，质量只升不降
3. **无限自主循环** → 人类不在时也能持续研究
4. **状态外化到文件系统** → 可中断、可恢复、可审计
5. **固定资源预算** → 每轮可预测、可对比

## 下一代架构

详见 [03-architecture.md](03-architecture.md)。

五个核心组件：

```
┌─────────────────────────────────────────────┐
│                                             │
│  1. 规划器: 动态优先队列（非一次性计划）       │
│     → 每轮根据结果重排优先级                  │
│                                             │
│  2. 执行器: 多模态行动                       │
│     → web_search + paper_read + code_run    │
│       + data_analyze + api_call             │
│     → 不只是搜索阅读，还能跑代码验证          │
│                                             │
│  3. 评估器: 自动标量指标                     │
│     → coverage × citation_rate × consistency │
│     → 粗糙但可计算 > 没有指标                │
│     → keep / revert / pivot 判定             │
│                                             │
│  4. 状态存储: 全持久化                       │
│     → report.md + sources.tsv + results.tsv │
│     → questions.md + experiments/           │
│     → git 版本控制                           │
│                                             │
│  5. 资源预算: 每轮固定                       │
│     → 5 次搜索 / 20 页 / 50K tokens / 3 min │
│     → 超预算即停，评估，下一轮               │
│                                             │
└─────────────────────────────────────────────┘
```

## 结论

下一代 deep research 的核心转变是三个词：**从管道到循环**。

| | 当前 | 下一代 |
|---|------|--------|
| 范式 | 管道（一次性） | 循环（迭代到收敛） |
| 知识来源 | 只读已有信息 | 读 + 实验产生新知识 |
| 质量保证 | 无 | 标量指标 + keep/revert |
| 状态 | 易失（context window） | 持久（文件系统 + git） |
| 人类角色 | 启动者 + 等待者 | 方向设定者 + 审查者 |

**一句话**：下一代 deep research = autoresearch 的迭代循环 + 当前 deep research 的信息获取能力。不是搜得更多，而是**搜完能验证、验证完能迭代、迭代到收敛**。

## 开放问题

1. **标量指标够用吗？** autoresearch 的 val_bpb 是客观物理量。研究报告的质量指标（coverage、citation_rate 等）是近似的。这个近似够驱动自主循环吗？还是会导致 Goodhart 问题（指标变好但报告实际变差）？

2. **code_run 的安全边界在哪？** 让 agent 自主运行代码来验证研究结论，权限怎么控？沙箱能覆盖所有风险吗？

3. **收敛判据是什么？** autoresearch 靠人类手动停。下一代 deep research 应该能自动判断"继续研究的边际收益已经低于成本"——这个判据怎么定？

4. **多 agent 协作**：一个 agent 做 deep research，另一个做 autoresearch 实验，第三个做评审——这种分工能产出更好的结果吗？还是单 agent 循环就够了？

## 文件索引

| 文件 | 内容 |
|------|------|
| [01-current-state.md](01-current-state.md) | 当前产品架构与六大局限 |
| [02-autoresearch-patterns.md](02-autoresearch-patterns.md) | autoresearch 五个可迁移设计模式 |
| [03-architecture.md](03-architecture.md) | 下一代五组件架构设计 |
