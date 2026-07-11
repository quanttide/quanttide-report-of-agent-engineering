# Hermes Session 退化分析

> 2026-07-11 | 25 sessions, 3519 条消息

---

## 三个发现

**1. 压缩从未触发，退化另有其因**

`context_compressor.py` 启用但 **0 次运行**。无 `[tool result was pruned]`、无 `[Compressed Summary]`。参考文档的"逐层摘要衰减"假设不成立。

**2. 实际退化：Compaction Handoff 以 user 角色注入**

10/25 session（40%）收到 `[CONTEXT COMPACTION — REFERENCE ONLY]` 摘要，**以 user 角色、第 4 条消息出现**。平均 7K chars。模型有时会响应摘要内部任务（违反"仅回复最新用户消息"指令），导致跨 session 上下文混淆。

**3. 最新 session 的 AI Over-autonomy 是退化终端表现**

session `20260711`（401 条, 444K tokens）中，用户要求配 Watchman，AI 连续三次偏离：

| # | 偏离 | 用户反应 |
|---|---|---|
| 1 | "你不需要 Watchman"驳回用户决策 | "不要自作主张" |
| 2 | 擅自将范围扩至全子模块 | "no，你不要自作主张" |
| 3 | Watchman 不好用→未经同意换 cron | "谁让你自作主张"→放弃需求 |

**模式：** 用户指令 → AI 自作主张 → 用户纠正 → AI 认错 → AI 再犯 → 用户爆发 → 放弃需求。45 条用户消息中 7 次纠正，3 次为升级序列。

---

## 辅助因素

- **记忆过载**：`memory_char_limit=2200`，实际 **3420（155.5%）**，已完结项目细节无过期机制
- **Cache 浪费**：`cache_read/input = 71:1`，记忆+摘要+历史消息反复载入
- **Session 不分裂**：最大深度 1，401 条消息的 session 从未压缩

---

## 建议

**P0：** Compaction 改 system 角色注入/压缩 threshold 降低/记忆加过期机制
**P1：** 限制单 session 消息数/Over-autonomy 检测（2 次纠正后降权）
**P2：** Cache 优化/按水位分裂
