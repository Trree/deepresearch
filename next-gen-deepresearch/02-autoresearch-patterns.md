# autoresearch 的可迁移设计模式

从 Karpathy 的 program.md 中提炼出的设计模式，这些模式不依赖于"训练 LLM"这个具体场景，可以直接迁移到 deep research 领域。

## 模式 1: 单一标量指标（Single Scalar Metric）

**autoresearch 中的实现**：`val_bpb`，一个数字，越低越好。所有决策（keep/discard）归结为这一个数字的比较。

**为什么有效**：
- 消除判断歧义——不需要讨论"这算不算改进"
- 使自主循环成为可能——agent 不需要理解"为什么好"，只需要比较数字
- 实验可排序——100 个实验按 val_bpb 排序，一眼看出最佳

**迁移到 deep research**：
当前 deep research 没有任何指标衡量"这份研究报告有多好"。如果能定义一个标量指标（或少数几个），deep research 就能像 autoresearch 一样自主迭代。

可能的指标：
- **覆盖度**：研究问题的各个子方面是否都被覆盖（可通过预定义的 checklist 自动检查）
- **引用密度**：每个事实性断言是否有来源支撑（可自动计数）
- **可验证声明比例**：报告中有多少断言可以被独立验证（可自动分类）
- **信息增益**：相比用户已知信息，报告提供了多少新信息（需要基线对比）

不需要一个完美指标——一个粗糙但可计算的指标，胜过没有指标。

## 模式 2: 改进即保留，否则回滚（Keep-or-Revert）

**autoresearch 中的实现**：
```
if val_bpb improved → git commit (advance branch)
if val_bpb same or worse → git reset (revert to last good state)
```

**为什么有效**：
- 棘轮机制——质量只升不降
- 失败的代价极低——revert 是一条命令，2 秒
- 鼓励冒险——既然 revert 这么容易，不如大胆尝试

**迁移到 deep research**：
每轮 deep research 产出一个版本的报告。新版本与旧版本对比指标：
- 变好了 → 保留新版本（keep）
- 没变好 → 丢弃，恢复旧版本（revert）

这需要报告版本化（git 管理）和自动指标对比。当前没有任何 deep research 产品这么做。

## 模式 3: 无限自主循环（LOOP FOREVER）

**autoresearch 中的实现**：
```
LOOP FOREVER:
  1. 想一个实验
  2. 做实验
  3. 看结果
  4. keep or discard
  5. 回到 1
NEVER STOP.
```

**为什么有效**：
- 人类睡觉，agent 跑 100 个实验
- 不需要人类决定"接下来做什么"——agent 自己从结果中推导下一步
- 时间变成了 agent 的资源而非瓶颈

**迁移到 deep research**：
当前 deep research 是一次性的。迁移后应该是：
```
LOOP FOREVER:
  1. 读当前报告，找到最薄弱/最不确定的部分
  2. 针对该部分做深入研究（搜索、阅读、综合）
  3. 更新报告
  4. 如果指标改善 → keep，否则 → revert
  5. 回到 1
```

每轮循环让报告在最薄弱处加厚。人类第二天醒来看到一份经过 100 轮迭代打磨的报告。

## 模式 4: 状态外化到文件系统（Externalized State）

**autoresearch 中的实现**：
- `results.tsv` — 实验历史
- git branch — 代码快照
- `run.log` — 当前实验输出
- `train.py` — 唯一的可变文件

**为什么有效**：
- Agent 的 context window 会被清空，但文件不会
- 任何时候中断，任何时候恢复——只需读 results.tsv + 当前 train.py
- 人类可以随时 `cat results.tsv` 看进度

**迁移到 deep research**：
当前 deep research 的中间过程（搜了什么、读了什么、为什么选这个来源）全在 context window 里，对话结束就丢。迁移后：
- `research-state.md` — 当前研究进度、已覆盖/未覆盖的方面
- `sources.tsv` — 已读来源列表 + 可信度评分 + 关键信息摘要
- `report.md` — 当前版本的报告（git 管理，可 diff 可 revert）
- `questions.md` — 尚未回答的子问题队列

## 模式 5: 固定资源预算（Fixed Budget）

**autoresearch 中的实现**：每次实验固定 5 分钟。不管你怎么改模型，预算不变。这使得不同实验直接可比。

**为什么有效**：
- 公平比较——大模型和小模型在同样时间预算下对比
- 防止过度投入——不会在一个死胡同里花 3 小时
- 可预测——12 个实验/小时，用户知道醒来能看到多少结果

**迁移到 deep research**：
每轮研究迭代分配固定预算：
- 搜索预算：最多 N 次搜索查询
- 阅读预算：最多 M 个页面
- 时间预算：最多 T 分钟
- token 预算：最多 K tokens

超预算 → 停止当前轮，评估指标，keep or revert，进入下一轮。这防止 agent 在一个方向上无限深挖。

## 总结

五个模式的关系：

```
模式 1 (标量指标)  →  使 模式 2 (keep/revert) 成为可能
                          ↓
模式 2 (keep/revert) →  使 模式 3 (无限循环) 安全
                          ↓
模式 4 (状态外化)   →  使 模式 3 (无限循环) 可中断可恢复
                          ↓
模式 5 (固定预算)   →  使 模式 3 (无限循环) 可预测
```

缺任何一个，自主循环都跑不起来。
