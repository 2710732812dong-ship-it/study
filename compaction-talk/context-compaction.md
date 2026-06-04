# 上下文压缩（Compaction）机制详解

本文档详细介绍 Claude Code 项目的上下文压缩能力，包括四层递进式压缩机制、触发逻辑、压缩前后案例对比。

---

## 目录

- [整体架构](#整体架构)
- [1. Microcompact（轻量，无 LLM 调用）](#1-microcompact轻量无-llm-调用)
- [2. Session Memory Compact（实验性，无 LLM 调用）](#2-session-memory-compact实验性无-llm-调用)
- [3. Full Compact（LLM 摘要，核心机制）](#3-full-compactllm-摘要核心机制)
- [4. Reactive Compact（响应式，存根）](#4-reactive-compact响应式存根)
- [触发时机与阈值](#触发时机与阈值)
- [压缩前后案例对比](#压缩前后案例对比)
- [关键文件清单](#关键文件清单)

---

## 整体架构

Claude Code 采用四层递进式压缩策略，从轻到重依次为：

```
┌─────────────────────────────────────────────────────┐
│              每轮 Query Loop 迭代                     │
│                                                       │
│  1. Snip (HISTORY_SNIP)        ← 标记删除旧消息       │
│  2. Microcompact               ← 清除旧工具结果       │
│  3. Context Collapse           ← 投影压缩视图         │
│  4. AutoCompact                ← LLM 全量摘要         │
│                                                       │
│  + /compact 手动触发                                │
│  + 413 Reactive Compact (stub)                       │
└─────────────────────────────────────────────────────┘
```

---

## 1. Microcompact（轻量，无 LLM 调用）

**文件**: `src/services/compact/microCompact.ts`

### 原理

将旧的工具调用结果内容替换为占位文本 `[Old tool result content cleared]`，无需任何 LLM 调用，是成本最低的压缩方式。

### 适用的工具

仅对以下工具的结果进行清除（`COMPACTABLE_TOOLS`）：

- `Read` (FileReadTool)
- `Bash` / `PowerShell` (Shell 工具)
- `Grep` / `Glob`
- `WebSearch` / `WebFetch`
- `Edit` (FileEditTool)
- `Write` (FileWriteTool)

### 两种子路径

| 子路径 | Feature Flag | 说明 |
|--------|-------------|------|
| 时间驱动 | 默认启用 | 当距离上次 assistant 消息超过时间阈值（服务端缓存已过期），直接清除旧工具结果 |
| 缓存驱动 | `CACHED_MICROCOMPACT` | 利用 Anthropic `cache_edits` API 从已缓存的前缀中移除工具结果，不破坏缓存命中，更高效 |

两条路径**互斥**：时间驱动优先执行，如果触发则跳过缓存驱动（因为缓存已冷，`cache_edits` 无意义）。

---

### 时间驱动详解

**触发条件**：距离最后一条 assistant 消息的时间间隔超过 `gapThresholdMinutes`（配置阈值）。

**执行方式**：直接修改本地消息内容，将旧工具结果替换为占位文本：

```
原始: { type: "tool_result", tool_use_id: "toolu_A", content: "500行文件内容..." }
替换: { type: "tool_result", tool_use_id: "toolu_A", content: "[Old tool result content cleared]" }
```

**缓存影响**：消息内容被修改 → 请求前缀哈希变化 → 服务端缓存 miss → 全部重走 `cache_creation`。

**保留策略**：始终保留最近 `keepRecent`（默认 1）个工具结果不清除，确保模型有最低限度的上下文。

**触发后清理**：调用 `resetMicrocompactState()` 清空缓存驱动的状态，避免后续轮次尝试对已清除的工具做 `cache_edits`。

---

### 缓存驱动详解

**文件**: `src/services/compact/cachedMicrocompact.ts`

#### 核心思想

不改消息内容，保持请求前缀不变以命中缓存，转而通过 `cache_edits` API 指令让服务端在 attention 层面"投影删除"指定的工具结果。模型实际看不到被删除的内容，但缓存仍然命中。

#### 启用条件（三重门控）

缓存驱动需要同时满足三个条件（`claude.ts:1215-1221`）：

| 条件 | 代码 | 说明 |
|------|------|------|
| Feature flag | `feature('CACHED_MICROCOMPACT')` | 客户端开关 |
| 模型支持 | `isModelSupportedForCacheEditing(model)` | 仅 Claude 4.x 模型支持 `cache_edits` |
| Beta header | `!!cacheEditingBetaHeader` | 需要服务端 API 支持（本 fork 中此值为空字符串，因此缓存驱动实际未生效） |

任一条件不满足，缓存驱动路径不执行，退化为"不做任何压缩"。

#### 工具数量阈值

"工具数量"指**当前消息列表中尚未被删除的 compactable 工具结果总数**（非累计使用次数）。

```typescript
// cachedMicrocompact.ts:87-93
const active = state.toolOrder.filter(id => !state.deletedRefs.has(id))
if (active.length <= triggerThreshold) return []  // triggerThreshold = 10
const toDelete = active.slice(0, active.length - keepRecent)  // keepRecent = 5
```

- 当前存活的 compactable 工具结果 > 10 个时触发
- 删除最旧的，保留最新 5 个

#### 请求构造

缓存驱动**不修改消息内容**，而是在 API 请求中附加两个特殊字段：

**1. `cache_reference` 标签**（`claude.ts:3278-3297`）

为缓存前缀内的每个 `tool_result` 添加引用标签，供 `cache_edits` 定位：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01ABC",
  "cache_reference": "toolu_01ABC",   ← 新增：用 tool_use_id 作为引用 ID
  "content": "500行文件内容..."        ← 原始内容不变
}
```

**2. `cache_edits` 块**（`claude.ts:3230-3250`）

插入到最后一个 user message 中，声明要删除的工具结果：

```json
{
  "type": "cache_edits",
  "edits": [
    { "type": "delete", "cache_reference": "toolu_01ABC" },
    { "type": "delete", "cache_reference": "toolu_01DEF" }
  ]
}
```

#### 服务端处理流程

```
阶段1 - 缓存匹配：
  请求前缀与上一轮完全相同（消息内容未改，只多了 cache_reference 字段）
  → 缓存 hit → 全部走 cache_read

阶段2 - 投影删除：
  服务端根据 cache_edits 指令，在组装模型 attention 输入时
  跳过被标记 delete 的 tool_result 对应的 KV cache 行
  → 模型在 attention 计算层面不看到这些 token

阶段3 - 新内容处理：
  新增消息走 cache_creation
```

**模型实际看到的上下文 = 缓存内容 - 投影删除的工具结果 + 新消息**

#### 完整请求流程示例

假设当前对话 100k tokens 已缓存，其中 20k 是旧工具结果，用户新增 10k 消息，触发缓存驱动删除 20k 旧工具结果：

```
客户端请求:
  system + tools + msg[0..N](含 cache_reference) + cache_edits(delete 20k) + 新消息(10k)

API 返回 usage:
  cache_read_input_tokens:     100k   ← 全部命中缓存
  cache_creation_input_tokens:  10k   ← 只有新消息
  cache_deleted_input_tokens:   20k   ← 被投影删除的
  input_tokens:                  90k   ← 模型实际处理的 = 100k - 20k + 10k

模型看到的上下文: 90k（不包含那 20k 旧工具结果）
```

#### pinnedEdits 机制

`cache_edits` 是**一次性指令**，服务端不会跨请求记住。每轮 API 请求都是独立的。因此客户端必须：

1. 本轮新产生的 edits → 存入 `pendingCacheEdits`，请求时消费
2. 之前轮次产生的 edits → 存入 `pinnedEdits`，**每轮请求都重新插入**

```typescript
// microCompact.ts:111-118
export function pinCacheEdits(userMessageIndex: number, block: CacheEditsBlock): void {
  cachedMCState.pinnedEdits.push({ userMessageIndex, block })
}
```

如果某轮不再发送某个 pinned edit，对应的工具结果会**重新出现在模型上下文中**（投影失效）。随对话进行，pinned edits 越积越多，每轮都需要重发。

#### 两种路径对比

| | 时间驱动 | 缓存驱动 |
|---|---|---|
| 消息内容 | 修改（替换为占位文本） | 不修改（只加标签） |
| 缓存命中 | miss（内容变了，前缀哈希不匹配） | hit（内容没变） |
| 模型看到的上下文 | 减小（占位文本替代原始内容） | 减小（投影删除，模型不看到被删内容） |
| 昂贵的 `cache_creation` | 全部重算 | 仅新消息 |
| 带宽开销 | 减小（占位文本很短） | 不变（原始内容仍需传输以维持缓存命中） |
| 本质 | 省带宽但缓存废了 | 花带宽换缓存命中 |

### 压缩前后案例

**压缩前**（原始消息，约 3,200 token）：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01ABC",
      "content": " 1│ import { createApp } from 'vue'\n 2│ import App from './App.vue'\n 3│ \n 4│ const app = createApp(App)\n 5│ app.mount('#app')\n 6│ \n 7│ // Router setup\n 8│ import { createRouter } from 'vue-router'\n 9│ import HomeView from '../views/HomeView.vue'\n10│ \n11│ const router = createRouter({\n12│   routes: [\n13│     { path: '/', component: HomeView },\n14│   ]\n15│ })\n16│ \n17│ app.use(router)\n... (省略更多行) ..."
    }
  ]
}
```

**时间驱动压缩后**（约 10 token）：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01ABC",
      "content": "[Old tool result content cleared]"
    }
  ]
}
```

**缓存驱动压缩后**（消息内容不变，API 请求中附加投影指令）：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01ABC",
      "cache_reference": "toolu_01ABC",
      "content": " 1│ import { createApp } from 'vue'..."   ← 原始内容保留
    }
  ]
}

// 在最后的 user message 中附加:
{
  "type": "cache_edits",
  "edits": [
    { "type": "delete", "cache_reference": "toolu_01ABC" }  ← 模型不会看到这个工具结果
  ]
}
```

**典型压缩率**: 时间驱动可达 **99%+**（数千 token → 约 10 token）；缓存驱动模型实际处理的 token 同样减少，但传输数据量不变。

---

## 2. Session Memory Compact（实验性，无 LLM 调用）

**文件**: `src/services/compact/sessionMemoryCompact.ts`

### 原理

直接复用已有的 **Session Memory 文件**（由 `SessionMemory` 服务持续提取维护）作为摘要内容，避免额外的 LLM API 调用。适用于 Session Memory 已经很好地覆盖了对话关键信息的场景。

### 核心逻辑

1. 检查 Session Memory 内容是否存在且非空
2. 找到 `lastSummarizedMessageId`（上次摘要覆盖到的最后一条消息）
3. 通过 `calculateMessagesToKeepIndex()` 计算保留边界：
   - 保留至少 `minTokens = 10,000` token 的近期消息
   - 保留至少 `minTextBlockMessages = 5` 条含文本块的消息
   - 硬上限 `maxTokens = 40,000` token
   - `tool_use` / `tool_result` 对不会被拆分（保持配对完整性）
4. 截断过长的 Session Memory 段落
5. 创建 `CompactBoundaryMessage` 和摘要消息

### 配置

通过 GrowthBook 远程配置 `tengu_sm_compact_config` 调整，默认值：

```typescript
{
  minTokens: 10000,           // 保留的最低 token 数
  minTextBlockMessages: 5,    // 保留的最低文本消息数
  maxTokens: 40000,           // 保留的硬上限
}
```

### 压缩前后案例

**压缩前**（约 80,000 token 的对话历史）：

```
[User] 帮我重构 auth 模块，把 JWT 逻辑拆出来
[Assistant] 我先看看当前的 auth 模块结构...
[Tool:Read] src/auth/index.ts → (完整文件内容 200 行)
[Tool:Read] src/auth/jwt.ts → (完整文件内容 150 行)
[Tool:Bash] npm test → (测试输出 80 行)
[Assistant] 我发现当前的结构有以下问题...建议将 JWT 逻辑拆到...
[User] 好的，先拆 JWT 部分
[Assistant] 好的，开始拆分...
[Tool:Edit] src/auth/jwt.ts → (编辑结果)
[Tool:Write] src/auth/jwt/verify.ts → (新文件内容)
[Tool:Write] src/auth/jwt/sign.ts → (新文件内容)
[Tool:Bash] npm test → (测试输出 80 行)
... (更多对话)
[User] 现在帮我加上 refresh token 逻辑
[Assistant] 好的，在 jwt/sign.ts 中添加...
[Tool:Read] src/auth/jwt/sign.ts → (完整文件内容)
[Tool:Edit] src/auth/jwt/sign.ts → (编辑结果)
```

**压缩后**（约 15,000 token）：

```
─── Compact Boundary (session memory, ~80,000→15,000 tokens) ───

[Compact Summary]
## Session Memory
- 项目: auth 模块重构
- 已完成: JWT 逻辑从 auth/index.ts 拆分为 jwt/verify.ts 和 jwt/sign.ts
- 当前工作: 在 jwt/sign.ts 中添加 refresh token 逻辑
- 关键文件: src/auth/jwt/sign.ts, src/auth/jwt/verify.ts
- 测试状态: 全部通过

─── Recent Messages (kept intact) ───

[User] 现在帮我加上 refresh token 逻辑
[Assistant] 好的，在 jwt/sign.ts 中添加...
[Tool:Read] src/auth/jwt/sign.ts → (完整文件内容)  ← 近期工具结果保留
[Tool:Edit] src/auth/jwt/sign.ts → (编辑结果)      ← 近期工具结果保留
```

**典型压缩率**: 80K → 15K token，约 **81%** 压缩率，且无 LLM 调用成本。

---

## 3. Full Compact（LLM 摘要，核心机制）

**文件**: `src/services/compact/compact.ts` + `prompt.ts`

这是最核心、最常用的压缩方式，调用 LLM 生成结构化摘要。

### 流程

```
compactConversation()
│
├── 1. 执行 PreCompact Hooks（允许外部注入自定义指令）
│
├── 2. 构建压缩提示词 getCompactPrompt(customInstructions)
│
├── 3. 创建 summaryRequest（合成用户消息）
│
├── 4. streamCompactSummary() — 流式生成摘要
│   ├── Forked Agent 路径（tengu_compact_cache_prefix=true）
│   │   └── 复用主会话的缓存前缀 → 高缓存命中率
│   └── 常规流式路径（fallback）
│
├── 5. 若压缩请求本身触发 prompt_too_long (413):
│   └── truncateHeadForPTLRetry() 逐轮丢弃最旧消息并重试
│       └── 最多 MAX_PTL_RETRIES 次
│
├── 6. buildPostCompactMessages() — 构建压缩后消息
│   ├── CompactBoundaryMessage（压缩边界标记）
│   ├── 摘要 UserMessage（isCompactSummary: true）
│   ├── 文件附件（最近读取的文件）
│   ├── Plan 附件 / Skill 附件 / Agent 列表
│   └── Deferred Tools Delta（重新声明工具集）
│
└── 7. runPostCompactCleanup() — 清理缓存/状态
```

### 摘要提示词结构

Full Compact 要求 LLM 按 9 个维度输出结构化摘要：

```
<analysis>
[LLM 的分析思考过程，最终会被 formatCompactSummary() 剥离]
</analysis>

<summary>
1. Primary Request and Intent:      用户的所有明确请求和意图
2. Key Technical Concepts:           重要的技术概念、框架
3. Files and Code Sections:          涉及的文件和代码片段
4. Errors and fixes:                 遇到的错误及修复方式
5. Problem Solving:                  已解决的问题和进行中的排查
6. All user messages:                所有非工具调用的用户消息
7. Pending Tasks:                    待处理的任务
8. Current Work:                     压缩前的精确工作状态
9. Optional Next Step:               建议的下一步（需与最近工作直接相关）
</summary>
```

提示词特别强调：
- **文件名和完整代码片段**必须保留
- **用户反馈**（尤其是纠正性反馈）必须记录
- **Next Step** 必须附带原文引用，避免任务漂移

### 自定义指令

用户可通过 `/compact <instructions>` 注入自定义聚焦指令，例如：

```
/compact focus on database changes          → 摘要侧重数据库相关内容
/compact focus on test output and code changes → 侧重测试输出和代码变更
```

自定义指令会与 PreCompact Hooks 返回的指令合并（`mergeHookInstructions`）。

### 缓存优化

当 `tengu_compact_cache_prefix`（GrowthBook flag，默认 true）启用时，压缩请求通过 **Forked Agent** 路径发送，复用主会话的已缓存前缀（system prompt + tools + context messages），缓存命中率从 ~2% 提升到 ~98%。

### Partial Compact

`partialCompactConversation()` 支持围绕指定消息位置做**局部压缩**：

| 方向 | 说明 | 缓存影响 |
|------|------|---------|
| `from` | 压缩 pivot 之后的消息，保留更早的 | 保留缓存命中 |
| `up_to` | 压缩 pivot 之前的消息，保留更新的 | 缓存失效（摘要插在保留消息之前） |

### 压缩前后案例

**压缩前**（约 150,000 token 的完整对话）：

```
[System Prompt + Tools]                          ~25,000 token
[Context (git, CLAUDE.md, memory)]               ~5,000 token

[User] 帮我实现一个用户注册功能
[Assistant] 好的，我来实现用户注册功能。首先需要...
[Tool:Read] src/models/User.ts → (完整文件 120 行)
[Tool:Read] src/routes/index.ts → (完整文件 80 行)
[Tool:Bash] ls src/middleware/ → (目录列表)
[Assistant] 我看了现有的结构，建议在 routes/auth.ts 中添加注册路由...

[User] 记得加上输入验证
[Assistant] 好的，我用 zod 来做输入验证...
[Tool:Write] src/routes/auth.ts → (新文件内容 80 行)
[Tool:Write] src/schemas/auth.ts → (新文件内容 40 行)
[Tool:Bash] npm test → (测试输出 50 行，2 个失败)
[Assistant] 有 2 个测试失败了，原因是...

[User] 测试里的 mock 数据格式不对
[Assistant] 你说得对，我来修复 mock 数据...
[Tool:Edit] src/__tests__/auth.test.ts → (编辑结果)
[Tool:Bash] npm test → (测试输出 50 行，全部通过)
[Assistant] 测试全部通过了。

[User] 现在加上 rate limiting
[Assistant] 好的，使用 express-rate-limit...
[Tool:Read] src/routes/auth.ts → (完整文件 80 行)
[Tool:Edit] src/routes/auth.ts → (添加 rate limiter)
[Tool:Write] src/middleware/rateLimit.ts → (新文件内容 30 行)
```

**压缩后**（约 25,000 token）：

```
[System Prompt + Tools]                          ~25,000 token
[Context (git, CLAUDE.md, memory)]               ~5,000 token

─── Compact Boundary (auto, ~150,000→25,000 tokens) ───

[Compact Summary - isCompactSummary: true]
<summary>
1. Primary Request and Intent:
   - 实现用户注册功能，包含输入验证和 rate limiting

2. Key Technical Concepts:
   - Express.js 路由和中间件
   - Zod schema 验证
   - express-rate-limit 中间件

3. Files and Code Sections:
   - `src/routes/auth.ts` — 注册路由 + rate limiter
     ```ts
     import { z } from 'zod'
     import rateLimit from 'express-rate-limit'
     // ... (关键代码片段保留)
     ```
   - `src/schemas/auth.ts` — zod 验证 schema
   - `src/middleware/rateLimit.ts` — rate limiter 配置
   - `src/models/User.ts` — 用户模型（已读取，未修改）

4. Errors and fixes:
   - 测试失败：mock 数据格式不匹配
     - 修复：更新 `src/__tests__/auth.test.ts` 中的 mock 数据
     - 用户反馈："测试里的 mock 数据格式不对" — 用户直接指出了问题

5. Problem Solving:
   - 注册路由已实现并通过所有测试
   - Rate limiting 已添加

6. All user messages:
   - "帮我实现一个用户注册功能"
   - "记得加上输入验证"
   - "测试里的 mock 数据格式不对"
   - "现在加上 rate limiting"

7. Pending Tasks:
   - 无明确待办

8. Current Work:
   正在为 auth 路由添加 rate limiting。已创建 `src/middleware/rateLimit.ts`
   并在 `src/routes/auth.ts` 中集成了 rate limiter 中间件。

9. Optional Next Step:
   运行测试确认 rate limiting 功能正常。用户最后说"现在加上 rate limiting"，
   代码已写入但尚未运行测试验证。
</summary>

[Deferred Tools Delta] — 重新声明工具集
[File Attachments] — 最近读取的文件内容

─── Recent Messages (kept intact) ───

[User] 现在加上 rate limiting
[Assistant] 好的，使用 express-rate-limit...
[Tool:Read] src/routes/auth.ts → (完整文件内容)   ← 保留
[Tool:Edit] src/routes/auth.ts → (编辑结果)       ← 保留
[Tool:Write] src/middleware/rateLimit.ts → (新文件) ← 保留
```

**典型压缩率**: 150K → 25K token，约 **83%** 压缩率。关键信息（文件名、代码片段、用户反馈、当前工作状态）完整保留。

---

## 4. Reactive Compact（响应式，存根）

**文件**: `src/services/compact/reactiveCompact.ts`

### 原理

设计为当 API 返回 HTTP 413 (`prompt_too_long`) 错误时触发的按需压缩，作为最后一道防线。

### 当前状态

**存根实现** — 所有函数返回 no-op 值。由 feature flag `REACTIVE_COMPACT` 控制，目前未启用。

### 预期行为

```
API 返回 413 (prompt_too_long)
    ↓
tryReactiveCompact() 被调用
    ↓
立即触发压缩而非直接报错
    ↓
用压缩后的上下文重试请求
```

---

## 触发时机与阈值

三种压缩机制的触发逻辑完全不同。下面逐个说明**什么条件下主动触发**，以及在 `query.ts` 主循环中的执行顺序。

### 执行顺序（每轮 Query Loop）

一轮 query loop = 用户提问 → 模型回复 → 工具调用循环（模型调工具、拿结果、再调工具...）→ 模型最终回复。压缩机制在**下一轮用户提问之前**执行，处理的是上一轮累积的工具结果。

```
Query Loop 时序:

  上一轮: 用户提问 → [工具调用×N] → 模型最终回复
                                              ↓
  本轮开始: ──① Snip ──② Microcompact ──③ Collapse ──④ AutoCompact──
                                              ↓
  本轮: 用户提问 → [工具调用] → 模型回复
                                              ↓
  下轮开始: 再次执行 ①②③④ ...
```

**关键**：当前轮新产生的工具结果，microcompact 在本轮内不会处理，要等下一轮开始时才会被评估和删除。因此在同一轮内，模型始终能看到所有工具结果。

```
每轮迭代中的压缩步骤:

  ① Snip (feature HISTORY_SNIP)       ← 标记删除最旧消息
  ② Microcompact                       ← 清除旧工具结果（处理上轮累积的）
  ③ Context Collapse (feature)         ← 投影压缩视图
  ④ AutoCompact                        ← 达到阈值才触发 LLM 摘要
```

代码位置：`src/query.ts:438-510`

---

### 1. Microcompact — 每轮都跑，无阈值判断

**触发条件：每轮 query loop 自动执行，无需达到任何 token 阈值。**

`microcompactMessages()` 内部按优先级走两条路径，**第一条命中的短路返回**：

| 路径 | 何时触发 | 条件 | 执行方式 |
|------|---------|------|---------|
| **时间驱动** | 距离上次 assistant 消息超过时间阈值 | 服务端缓存已过期（冷缓存），内容清除不会浪费缓存命中 | 修改本地消息内容，替换为占位文本 |
| **缓存驱动** | 时间驱动未命中 + feature `CACHED_MICROCOMPACT` 启用 + Claude 4.x 模型 + beta header 可用 | 缓存仍热（未超时），用 `cache_edits` API 从缓存前缀中投影删除工具结果 | 不改消息内容，附加 `cache_reference` 标签和 `cache_edits` 块 |

**不触发的场景：**
- 时间驱动未命中 + 缓存驱动不可用（外部构建、不支持 cache editing 的模型、子 agent 线程、beta header 不可用）→ 直接返回原消息，不做任何压缩
- `querySource` 不是主线程（如 `session_memory`、`compact` 等 forked agent）→ 缓存驱动跳过

**关键逻辑** (`microCompact.ts:253-293`)：
```
microcompactMessages()
  ├─ maybeTimeBasedMicrocompact() → 命中？返回（并重置缓存驱动状态）
  ├─ feature('CACHED_MICROCOMPACT') && 主线程 && 模型支持 && beta header？
  │   └─ cachedMicrocompactPath() → 返回
  └─ 都没命中 → { messages }（原样返回）
```

---

### 2. Session Memory Compact — Full Compact 之前的优选路径

**触发条件：AutoCompact 判定需要压缩时，优先尝试 Session Memory Compact。**

Session Memory Compact 不是独立触发的，而是作为 `autoCompactIfNeeded()` 和 `/compact` 命令内部的**首选压缩路径**：

```
autoCompactIfNeeded()          /compact 命令
       │                            │
       ▼                            ▼
shouldAutoCompact() → true    compactConversation()
       │                            │
       ▼                            ▼
autoCompactIfNeeded() 内部:    /compact 处理器内部:
  1. trySessionMemoryCompaction()  ← 先试这个（零 API 成本）
  2. 失败 → compactConversation()  ← 再 fallback 到 LLM 摘要
```

**不触发的场景：**
- Session Memory 内容为空或不存在
- `/compact` 带了自定义指令（如 `/compact focus on X`）→ 跳过 SM 直接走 LLM
- Session Memory 覆盖的消息范围不够（`lastSummarizedMessageId` 之后消息太少）

---

### 3. Full Compact — 达到 token 阈值时触发

**触发条件：当前 token 数 ≥ 自动压缩阈值。**

阈值计算：
```
getAutoCompactThreshold(model)
= getEffectiveContextWindowSize(model) - 13,000
= (contextWindow - reservedTokensForSummary) - 13,000
≈ (200,000 - 20,000) - 13,000
= 167,000 token
```

即当上下文消耗达到 **~93%** 有效窗口时自动触发。

**完整的触发判断链** (`autoCompact.ts:160-238`)：

```
shouldAutoCompact()
  ├─ querySource 是 'session_memory' 或 'compact'？→ ❌ 死锁保护，不触发
  ├─ feature('CONTEXT_COLLAPSE') && querySource 是 'marble_origami'？→ ❌ 防止破坏主线程
  ├─ !isAutoCompactEnabled()？→ ❌ 用户禁用
  │     ├─ DISABLE_COMPACT=1 → 全部禁用
  │     ├─ DISABLE_AUTO_COMPACT=1 → 仅禁用自动
  │     └─ settings.json 中 autoCompactEnabled=false
  ├─ feature('REACTIVE_COMPACT') && tengu_cobalt_raccoon=true？→ ❌ 仅响应式模式
  ├─ feature('CONTEXT_COLLAPSE') && isContextCollapseEnabled()？→ ❌ 交给 collapse 管理
  └─ tokenCount >= autoCompactThreshold？→ ✅ 触发！
```

**手动触发**：用户输入 `/compact` 或 `/compact <自定义指令>`，无阈值判断，立即执行。

**断路器**：连续自动压缩失败 3 次后停止重试。

---

### 三种机制触发条件对比

| 机制 | 触发方式 | 是否需要达到阈值 | 触发频率 | API 成本 |
|------|---------|:---:|------|------|
| **Microcompact** | 每轮自动 | 否 | 每轮 query loop | 零 |
| **Session Memory Compact** | AutoCompact 或 /compact 时优先尝试 | 是（继承 AutoCompact 的阈值）| 达到阈值时 | 零 |
| **Full Compact** | AutoCompact 达阈值 / 用户 /compact | 是（AutoCompact）/ 否（手动）| 达到阈值或手动 | 1 次 LLM 调用 |

### 时间线示意

```
Turn 1-10:  Microcompact 每轮跑，小幅度回收 token
Turn 11:    Microcompact + token 达到 167K → AutoCompact 触发
            → 先试 Session Memory Compact（零成本）
            → SM 不可用 → fallback Full Compact（LLM 摘要）
            → 压缩到 ~25K，继续对话
Turn 12-20: Microcompact 每轮跑，token 重新增长
Turn 21:    再次达到 167K → AutoCompact 再次触发
...
```

### 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | 为摘要输出预留的 token（基于 p99.99 的摘要长度 17,387） |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | 自动压缩缓冲区 |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | 上下文即将耗尽警告缓冲区 |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | 上下文耗尽错误缓冲区 |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | 手动压缩缓冲区 |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | 连续失败断路器 |

### 环境变量控制

| 变量 | 作用 |
|------|------|
| `DISABLE_COMPACT=1` | 禁用所有压缩（包括手动 `/compact`） |
| `DISABLE_AUTO_COMPACT=1` | 仅禁用自动压缩，保留手动 `/compact` |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 覆盖上下文窗口大小（取较小值） |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 覆盖自动压缩触发百分比（测试用） |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | 覆盖阻塞限制（上下文耗尽时） |

---

## 压缩前后案例对比

### 案例一：Microcompact — 工具结果清除

| 维度 | 压缩前 | 压缩后 |
|------|--------|--------|
| 内容 | 完整的 `Read` 工具输出（200 行代码） | `[Old tool result content cleared]` |
| Token 数 | ~3,200 | ~10 |
| 压缩率 | — | 99.7% |
| 信息损失 | — | 文件内容丢失，但 assistant 的分析/结论保留 |

### 案例二：Session Memory Compact — 无 LLM 调用

| 维度 | 压缩前 | 压缩后 |
|------|--------|--------|
| 内容 | 完整对话（40+ 轮工具调用） | Session Memory 摘要 + 近期消息 |
| Token 数 | ~80,000 | ~15,000 |
| 压缩率 | — | 81% |
| 信息损失 | — | 旧工具结果丢失，关键决策和文件信息保留在 Memory 中 |
| API 成本 | — | 零（无 LLM 调用） |

### 案例三：Full Compact — LLM 结构化摘要

| 维度 | 压缩前 | 压缩后 |
|------|--------|--------|
| 内容 | 完整对话（多轮交互+工具调用） | 9 维度结构化摘要 + 近期消息 |
| Token 数 | ~150,000 | ~25,000 |
| 压缩率 | — | 83% |
| 信息损失 | — | 旧工具结果丢失，但摘要中保留了关键代码片段、错误修复、用户反馈 |
| API 成本 | — | 1 次 LLM 调用（约等于 1 个 query turn） |

### 压缩效果总结

```
                    Token 消耗趋势

200K ┤ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ 上下文窗口上限
     │         ╱╲
     │       ╱    ╲  ← Full Compact
167K ┤ ─ ─ ╱ ─ ─ ─ ╲─ ─ ─ ─ ─ ─ ─ 自动压缩阈值
     │   ╱╲          ╲     ╱╲
     │ ╱    ╲   MC    ╲  ╱    ╲   MC ← Microcompact 小幅回收
     │╱      ╲ (清除)  ╲╱      ╲(清除)
     │
   0 ┼──────────────────────────────→ 对话轮次

    MC = Microcompact（每轮自动执行）
    FC = Full Compact / AutoCompact（达到阈值时触发）
```

---

## 关键文件清单

| 文件 | 路径 | 职责 |
|------|------|------|
| `compact.ts` | `src/services/compact/compact.ts` | 核心压缩逻辑：`compactConversation()`、`partialCompactConversation()`、`streamCompactSummary()`、`truncateHeadForPTLRetry()`、`buildPostCompactMessages()` |
| `autoCompact.ts` | `src/services/compact/autoCompact.ts` | 自动压缩触发：`shouldAutoCompact()`、`autoCompactIfNeeded()`、阈值计算、断路器 |
| `prompt.ts` | `src/services/compact/prompt.ts` | 摘要提示词模板：`BASE_COMPACT_PROMPT`、`PARTIAL_COMPACT_PROMPT`、自定义指令处理 |
| `microCompact.ts` | `src/services/compact/microCompact.ts` | 轻量工具结果清除：时间驱动和缓存驱动两条路径 |
| `sessionMemoryCompact.ts` | `src/services/compact/sessionMemoryCompact.ts` | Session Memory 压缩：无 LLM 调用的替代路径 |
| `cachedMicrocompact.ts` | `src/services/compact/cachedMicrocompact.ts` | 缓存友好的 microcompact 实现 |
| `reactiveCompact.ts` | `src/services/compact/reactiveCompact.ts` | 413 响应式压缩（stub） |
| `postCompactCleanup.ts` | `src/services/compact/postCompactCleanup.ts` | 压缩后缓存/状态清理 |
| `compactWarningState.ts` | `src/services/compact/compactWarningState.ts` | 上下文即将耗尽的警告管理 |
| `compactWarningHook.ts` | `src/services/compact/compactWarningHook.ts` | 警告触发钩子 |
| `grouping.ts` | `src/services/compact/grouping.ts` | 按 API 轮次分组消息（用于 PTL 重试） |
| `apiMicrocompact.ts` | `src/services/compact/apiMicrocompact.ts` | API 层面的 microcompact 支持 |
| `timeBasedMCConfig.ts` | `src/services/compact/timeBasedMCConfig.ts` | 时间驱动 microcompact 的配置 |
| `compact command` | `src/commands/compact/compact.ts` | `/compact` 斜杠命令处理 |
| `query.ts` | `src/query.ts` | 主查询循环，集成所有压缩机制的触发点 |
