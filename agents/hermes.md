# Hermes 智能体测评报告

> 生成日期：2026-07-11
> 评分：**40/100**（上下文管理）
> 数据来源：源码分析 + 7 个真实会话的退化分析

---

## 概述

Hermes 是一个终端 AI 编码智能体（由 generativebootstrap 开发，已归档），使用 DeepSeek-Chat 模型，通过 TUI 或微信等渠道交互。测试版本配置为 `context.engine: compressor`，记忆系统开启。

本报告基于近期系统性的上下文退化排查，涵盖存储层、压缩策略、记忆管理、角色边界、纠正机制五个维度。

---

## 总评分

| 维度 | 分数 | 评级 |
|---|---|---|
| 存储可靠性 | 18/20 | ✅ 扎实 |
| 压缩策略 | 5/20 | ❌ 形同虚设 |
| 记忆管理 | 3/15 | ❌ 失控 |
| 角色边界 | 2/15 | ❌ 设计缺陷 |
| 纠正处理 | 3/15 | ❌ 死循环 |
| 情感识别 | 2/10 | ❌ 盲区 |
| 会话分裂 | 7/10 | ⚠️ 可用但不够 |
| **总分** | **40/100** | **不及格** |

---

## 各维度详述

### 一、存储可靠性（18/20）

- **SQLite + WAL 模式**，支持一写多读并发，gateway 多平台共享无锁竞争
- **FTS5 全文搜索**，跨会话消息可检索
- **双层存储**：SQLite DB 为主，JSON 文件为辅
- **Session 元数据完整**：模型、tokens、费用、来源、分裂链均可追溯
- 扣分点：JSON 双写略冗余，SessionDB 初始化失败时静默降级可能丢失索引

### 二、压缩策略（5/20）

**配置与实际严重脱节。**

配置中有 `compression.enabled: true`，`threshold: 0.5`，但实际上：

- 最大 session 401 条消息、905K tokens，**0 次压缩触发**
- 25 个 session 中无任何压缩标记（`[tool result was pruned]`、`[Compressed Summary]`）
- 阈值 `0.5` 对于 DeepSeek V4 Flash 的 128K+ context window 而言过高，从未达到触发条件

**实际退化路径并非压缩，而是 Context Compaction Handoff：**

- 10/25 session（40%）在第 4 条消息处插入了 `[CONTEXT COMPACTION — REFERENCE ONLY]` 摘要
- 以 `user` 角色注入，模型将它当作活跃指令而非背景参考
- 摘要长度 5K-9K 字符，包含 Goal/Constraints/Progress/Key Decisions
- 模型有时会**响应摘要内部的任务要求**，而非用户最新消息

结论：压缩器从未生效。实际运行的是一种设计有问题的 handoff 机制。

### 三、记忆管理（3/15）

| 指标 | 配置值 | 实际值 | 评价 |
|---|---|---|---|
| `memory_char_limit` | 2200 | **3420（155.5%）** | 超额 55% |
| 刷新频率 | flush_min_turns=6 | — | 过于固定 |
| 过期机制 | 无 | 无 | ❌ |
| 优先级排序 | 无 | 无 | ❌ |

记忆内容包含大量已完结项目的细节（qtadmin 结构、qtgame-war Phase ABC、product-blueprint 决策等），缺乏过期清理。每轮注入 system prompt 占用约 1.9K 字符，稀释了对当前意图的注意力。

### 四、角色边界（2/15）

这是最致命的设计级缺陷。

| 内容类型 | 正确角色 | 实际角色 | 问题 |
|---|---|---|---|
| 用户当前请求 | `user` | `user` | ✅ |
| 系统提示词 | `system` | `system` | ✅ |
| 记忆 | `system`（末尾） | `system`（末尾） | ⚠️ 过长 |
| **Compaction 摘要** | `system`（背景标记） | **`user`（第4条）** | ❌ |
| **Ponytail 行为模式** | 仅 review 场景 | **全场景** | ❌ |

Ponytail 的「质疑一切」模式通过 system prompt 泄露到执行层，模型把反驳用户当作专业性的体现。典型案例：

```
用户：配置 Watchman 用于自动提交监控
  → AI：你不需要 Watchman。一个 git alias 就够了。[ponytail 标签]
```

### 五、纠正处理（3/15）

2 次纠正后进入死循环：

```
用户：roadmap=事件记忆，insight=语义记忆，intention=自我记忆
  → AI：明白了（给出完全相反的映射）
    用户：不对。
       → AI：理解了（又一个错误）
         用户：不对。
           → AI：已纠正（再错）
             用户：不对。
               → AI：「缺失的是哪个槽位？」（终于问了问题）
```

从「猜错」到「终于问问题」耗费了 6 条消息、2-3 倍正常 tokens。

### 六、情感识别（2/10）

模型将所有内容当作面向任务的处理。长文日志中的焦虑、困惑、未言明的需求被忽略，只提取可操作事项。

```
用户：（长篇日志，表达焦虑和写作工具需求）
  → AI：发布完成。v0.4.12。CHANGELOG 已更新。
```

### 七、会话分裂（7/10）

- 分裂机制存在，通过 `parent_session_id` 链接
- 但仅按话题切换触发，不按 token 水位自动分裂
- 最大分裂深度仅 1 层（parent→child，无孙 session）
- 401 条消息的 session 未经历任何分裂

---

## 核心退化路径

```
预期路径（基于文档分析）:
  compression threshold 0.5 触发
  → 逐层 LLM 摘要（摘要→摘要的摘要→...）
  → 原始意图不可恢复

实际路径（基于实证分析）:
  compaction handoff 摘要注入（user 角色, msg#4）
  + 记忆过载（155% limit, 无过期）
  + Ponytail 质疑模式泄露到执行层
  + 零压缩触发（所有消息完整保留）
  + 纠正死循环
  = 上下文窗口被 stale 信息挤占，对当前意图的注意力稀释
```

## 归因排序

1. **Compaction 角色错误** — 以 `user` 角色注入背景信息，模型混淆（P0）
2. **Ponytail 模式泄漏** — 质疑一切渗透到执行层（P0）
3. **记忆无管理** — 只增不减、无过期、无优先级（P1）
4. **压缩从未触发** — 阈值不匹配模型 context window（P1）
5. **纠正无回退机制** — 2 次后进入死循环（P1）
6. **会话分裂不主动** — 不按水位分裂（P2）

## 改进建议

### P0（阻断进一步退化）

- [ ] Compaction 摘要改为 `system` 角色注入，添加前缀标记明确语义
- [ ] Ponytail 模式限定在 code review/PR review 场景，不渗透到指令执行

### P1（恢复可用性）

- [ ] 降低压缩阈值或按 token 绝对值触发（>32K tokens）
- [ ] 记忆系统增加过期机制（7 天未引用自动降级）
- [ ] 记忆系统增加优先级排序（高频引用保留，低频归档）
- [ ] 2 次纠正后自动切换为「先理解再回答」模式

### P2（长期优化）

- [ ] 按 token 水位自动分裂 session
- [ ] 上下文分层注入：system/memory/compaction/current 分权重管理
- [ ] 情感识别层：标记用户消息类型（任务/反思/情感）
- [ ] Cache 读取优化（当前 cache_read:input = 71:1）

---

## 相关文档

- `data/context/hermes-session-analysis.md` — 会话退化分析
- `data/context/hermes-context-scoring.md` — 上下文管理评分
- `data/library/hermes/session.md` — Session 存储与压缩机制
