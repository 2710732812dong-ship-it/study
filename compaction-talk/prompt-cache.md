# Anthropic Prompt Cache 机制详解

本文档详细介绍 Anthropic Prompt Cache 的原理、在 Claude Code 中的实现、以及不同 Provider 的缓存差异。

---

## 目录

- [1. 什么是 Prompt Cache](#1-什么是-prompt-cache)
- [2. 缓存的基本原理](#2-缓存的基本原理)
- [3. 上下文、缓存与模型 Input/Output 的关系](#3-上下文缓存与模型-inputoutput-的关系)
- [4. cache_control 断点机制](#4-cache_control-断点机制)
- [5. cache_control 与 tokenize 的关系](#5-cache_control-与-tokenize-的关系)
- [6. 请求文本拼接过程](#6-请求文本拼接过程)
- [7. scope 缓存共享范围](#7-scope-缓存共享范围)
- [8. 缓存匹配逻辑](#8-缓存匹配逻辑)
- [9. 缓存失效场景](#9-缓存失效场景)
- [10. cache_edits 读取时投影](#10-cache_edits-读取时投影)
- [11. SYSTEM_PROMPT_DYNAMIC_BOUNDARY](#11-system_prompt_dynamic_boundary)
- [12. promptCacheBreakDetection 缓存中断检测](#12-promptcachebreakdetection-缓存中断检测)
- [13. 不同 Provider 的缓存差异](#13-不同-provider-的缓存差异)
- [14. 缓存与压缩的协作关系](#14-缓存与压缩的协作关系)
- [关键文件清单](#关键文件清单)

---

## 1. 什么是 Prompt Cache

Prompt Cache 是 Anthropic API 的一项基础设施能力。当多个 API 请求共享相同的输入前缀时，服务端可以复用之前处理该前缀时计算出的中间结果（KV cache），避免重复计算。

### 计费差异

| 计费类型 | 价格（Sonnet） | 说明 |
|---------|---------------|------|
| `cache_creation_input_tokens` | $3.75/M token | 首次处理，写入缓存（1.25x input） |
| `cache_read_input_tokens` | $0.30/M token | 命中缓存，直接读取（0.1x input） |
| `input_tokens` | $3.00/M token | 无缓存的正常输入 |

缓存命中的价格仅为正常输入的 **1/10**，cache 写入比正常输入贵 **25%**。一个长会话中缓存命中可节省大量成本。

---

## 2. 缓存的基本原理

### Transformer 的 KV Cache

```
模型处理流程:
  输入 token → Transformer 层逐层计算 → 每层产出 K/V 矩阵 → 最终输出

  首次请求: 全部从头计算，存储 KV 矩阵（这就是"缓存"）
  再次请求: 前缀相同时，直接加载已存的 KV 矩阵，跳过重复计算
```

缓存的是**模型处理输入时的中间计算结果（KV cache）**，不是原始文本。

#### KV Cache 存储的是什么

Prompt Cache 存储的是每个 token 在**每一层**的 K 向量和 V 向量：

- **K（Key）**：当前 token 的"索引/标签"，用于被其他 token 查询匹配
- **V（Value）**：当前 token 的"实际内容"，用于被其他 token 提取信息

```
2 个 token（"猫" "坐"）、3 层、4 维的 KV Cache 示例:

Layer 0:
  K (2×4):              V (2×4):
       d0    d1  d2  d3      d0    d1    d2    d3
  猫: [0.23, -0.71, ...]  猫: [0.91, 0.22, -0.33, 0.65]
  坐: [1.05, 0.33, ...]   坐: [-0.17, 0.88, 0.44, -0.56]

Layer 1:  (值与 Layer 0 完全不同，因为每层输入不同)
  K (2×4):              V (2×4):
       d0    d1  d2  d3      d0    d1    d2    d3
  猫: [0.55, 0.12, ...]  猫: [-0.55, 0.44, 0.67, 0.33]
  坐: [0.31, -0.67, ...] 坐: [0.89, -0.11, 0.23, -0.55]

Layer 2:  (值又不同)
  ...
```

**矩阵行数 = token 数**，每行是一个 token 在该层的 K 或 V 向量。

#### 层和维度的含义

**层（Layer）**：Transformer 是串叠的多层结构，每层逐步提取不同级别的语义：

```
Layer 0:  提取基础特征（词性、局部关系）
Layer 1:  提取词间关系（主谓、修饰）
Layer 2:  提取句法结构
  ...
Layer N:  提取高级语义（意图、推理）
```

越深的层看到越抽象的关系。不同层的 K/V 值完全不同，因为每层的输入是上一层的输出。

**维度（Dimension）**：每个 token 的 K/V 向量有多个分量，每个分量编码一种语义特征：

```
dim_0   dim_1   dim_2  ... dim_127
[0.23,  -0.71,  0.55,  ... 0.33]
```

维度越高，能编码的信息越丰富。不同层中相同维度的含义不同。

#### K 和 V 在 Attention 中的作用

生成新 token 时，用 Q 查询所有旧 token 的 K，按匹配度提取 V：

```
新 token "在" 生成 Q → 查 "猫" 和 "坐" 的 K:
  "猫" 的 K: [0.23, -0.71, ...] → 匹配度 0.82 (高)
  "坐" 的 K: [1.05, 0.33, ...]  → 匹配度 0.45 (中)

按匹配度加权取 V:
  0.82 × "猫" 的 V + 0.45 × "坐" 的 V → "在" 的输出
```

- **K**：回答"我能不能帮到你？"（匹配阶段）
- **V**：回答"我能给你什么？"（提取阶段）

#### 为什么只缓存 K 和 V

- **Embedding**：不需要缓存，token → 查表即可重建
- **Q（Query）**：不需要缓存，Q 是当前新 token 产生的，每次都不同
- **Attention 分数**：不需要缓存，每次用 Q × K 实时计算
- **K 和 V**：计算成本高（要跑完整个前向传播）、反复需要（每生成一个新 token 都要查）、内容不变（同样前缀下永远相同）

#### 存储量估算

实际模型中每个向量是 128 维 float，80+ 层：

```
1 token 的缓存 = 80 layers × 2 (K+V) × 128 dim × 4 bytes = ~80 KB
100k token 的缓存 ≈ 8 GB
```

缓存存储在 **Anthropic 服务端 GPU 显存**，不在客户端。这也是缓存有 TTL 过期和计费策略的原因。

#### 每个新 token 依赖之前所有 token

Self-Attention 的核心特性：每个新 token 的 Q 会查询之前所有 token 的 K，按匹配度提取 V。匹配分数不均匀分布，相关的 token 权重高，无关的接近 0。

这意味着 KV Cache 必须保留全部 token 的 K 和 V，少一行都不行。

### 缓存存储位置

**Anthropic API 服务端的内存/SSD 中**，不是客户端，也不是模型内部。客户端无法直接读取或操作缓存，只能通过请求参数影响缓存的创建和命中。

### 缓存 TTL

| TTL | 条件 | 说明 |
|-----|------|------|
| 5 分钟 | 默认 | 5 分钟内无访问则淘汰 |
| 1 小时 | `ttl: '1h'` | 需满足特定条件（1P 用户、非 overage 等） |

代码中 `should1hCacheTTL()` 判断是否启用 1 小时 TTL。时间驱动 microcompact 用 60 分钟作为阈值，对应此 TTL。

---

## 3. 上下文、缓存与模型 Input/Output 的关系

理解缓存机制的关键是区分四个概念：**上下文**、**缓存**、**模型 Input**、**模型 Output**。它们并不等同。

### 四个概念

| 概念 | 是什么 | 在哪 | 谁控制 |
|------|--------|------|--------|
| **上下文（Context）** | 客户端发给 API 的全部内容：system_prompt + tools + messages + cache_control + cache_edits | 客户端构造，服务端接收 | 客户端 |
| **缓存（Cache）** | 服务端处理上下文时的中间计算结果（KV 矩阵） | 服务端内存/SSD | 服务端，客户端通过 `cache_control` 影响 |
| **模型 Input** | 模型实际参与 attention 计算的 token 集合 | 服务端，在缓存读取之后 | 服务端，受 `cache_edits` 影响 |
| **模型 Output** | 模型生成的回复 token | 服务端 | 模型 |

**关键区分**：上下文 ≠ 模型 Input。上下文是你发送的，模型 Input 是模型实际看到的，中间有缓存匹配和 `cache_edits` 投影两个环节。

### 完整数据流

```
客户端构造                    服务端处理
┌─────────────┐              ┌──────────────────────────────────┐
│             │   HTTP请求    │                                  │
│  上下文      │ ──────────→ │  1. 缓存匹配：前缀哈希比对         │
│  (100k)     │              │     命中 → cache_read (100k)      │
│             │              │     未命中 → cache_creation       │
│  + cache_control           │                                  │
│  + cache_edits │           │  2. 投影组装：                     │
│  + cache_reference         │     cache_edits 删除 -20k        │
│             │              │     → 模型 Input = 100k - 20k    │
│             │              │                                  │
│             │              │  3. 模型推理：                     │
│             │              │     attention(Input) → Output     │
│             │  ←────────── │                                  │
│             │   流式响应    │  4. 返回：                        │
│             │              │     output_tokens (2k)            │
│             │              │     usage: cache_read/creation/   │
│             │              │            deleted/input_tokens   │
└─────────────┘              └──────────────────────────────────┘
```

### 具体数字示例

对话第 5 轮，上下文已缓存 100k，其中 20k 旧工具结果，用户新增 10k 消息，触发缓存驱动压缩：

```
客户端发送的上下文:
  system_prompt (5k) + tools (20k) + messages (80k) = 105k
  其中 messages 含 cache_reference 标签 + cache_edits(delete 20k)

         ↓

服务端阶段1 - 缓存匹配:
  前 100k 与上次请求前缀相同 → cache_read: 100k
  新增 5k (新消息的一部分) → cache_creation: 5k

         ↓

服务端阶段2 - 投影组装:
  cache_edits 指令: 删除 20k 旧工具结果
  → cache_deleted: 20k
  → 模型 Input = 100k - 20k + 5k = 85k

         ↓

服务端阶段3 - 模型推理:
  attention(KV_cache[85k有效行] + 新KV[5k]) → output

         ↓

API 返回 usage:
  cache_read_input_tokens:    100k   ← 从缓存读了多少
  cache_creation_input_tokens:  5k   ← 新写入缓存多少
  cache_deleted_input_tokens:  20k   ← 读了但被投影删除
  input_tokens:                 85k   ← 模型实际处理的
  output_tokens:                 2k   ← 模型输出的
```

### 不同场景下四者的关系

#### 场景 A：纯追加消息（最常见）

上下文 = 模型 Input = 缓存读取 + 新增。三者一致。

```
上下文: 追加 10k 新消息
缓存:   前 100k 命中 → cache_read
模型Input: 100k + 10k = 110k（模型看到全部历史 + 新消息）
```

#### 场景 B：时间驱动压缩（修改消息内容）

上下文变小了，但缓存废了。模型 Input = 新上下文。

```
上下文: 旧工具结果替换为占位文本 → 内容变了 → 80k
缓存:   前缀哈希不匹配 → miss → 全部 cache_creation
模型Input: 80k
```

#### 场景 C：缓存驱动压缩（cache_edits）

上下文 ≠ 模型 Input。差值 = cache_deleted。

```
上下文: 100k（原始内容不变）+ cache_edits 指令
缓存:   前缀哈希不变 → hit → cache_read (100k)
模型Input: 100k - 20k = 80k（投影删除后）
```

#### 场景 D：Full Compact（LLM 摘要）

上下文大幅缩小，缓存也废了，但后续几轮缓存又可以逐步建立。

```
上下文: 被 LLM 摘要替换 → 150k → 25k
缓存:   前缀完全变了 → miss → cache_creation (25k)
模型Input: 25k
```

### 全局关系图

```
              上下文（客户端发送的）
              ┌───────────────────────┐
              │ system + tools + msgs  │
              │ + cache_control        │
              │ + cache_reference      │
              │ + cache_edits          │
              └───────────┬───────────┘
                          │
                    前缀哈希匹配？
                  ╱              ╲
               hit                miss
                │                  │
         cache_read          cache_creation
                │                  │
                ╲                ╱
            缓存 KV 矩阵（服务端存储）
                    │
            cache_edits 投影？
              ╱           ╲
           yes             no
            │               │
     删除指定 KV 行     保留全部 KV 行
            │               │
            ╲             ╱
          模型 Input（模型实际看到的）
                │
          attention 计算
                │
          模型 Output
```

---

## 4. cache_control 断点机制

### 断点的作用

没有 `cache_control` 标记的位置，服务端**不缓存任何东西**。断点是告诉服务端"请把到此处为止的内容存入缓存"的指令。

### 断点位置

Claude Code 在每个 API 请求中放置最多 **3 个断点**：

```
请求结构:

  ┌─ system_prompt 块1 ────────────────┐ [cache_control]  ← 断点1
  ├─ system_prompt 块2 ────────────────┤ [cache_control]  ← 断点2（可选）
  ├─ tools（最后一个 tool）─────────────┤ [cache_control]  ← 断点3
  ├─ message[0] ──────────────────────┤
  ├─ ...                               │
  └─ message[N]（最后一个消息的最后一个内容块）┘ [cache_control]  ← 断点4
```

**每个请求恰好 1 个 message 级 `cache_control` 标记**（`addCacheBreakpoints()` 函数控制）。Forked agent 场景下标记在倒数第二条消息（避免为 fork 的独特尾部创建缓存条目）。

### cache_control 的结构

```typescript
// 基础（5min TTL, org scope）
{ type: 'ephemeral' }

// 1 小时 TTL
{ type: 'ephemeral', ttl: '1h' }

// 全局共享
{ type: 'ephemeral', scope: 'global' }

// 全局共享 + 1 小时
{ type: 'ephemeral', scope: 'global', ttl: '1h' }
```

### 断点为什么不能更多？

Anthropic API 限制每个请求最多 4 个 `cache_control` 断点。且每个断点都有管理开销，多加断点在"前缀中间某段变化"的场景下**不会节省任何缓存**（因为缓存键是累积前缀，详见第 5 节）。

---

## 5. cache_control 与 tokenize 的关系

### `cache_control` 不是文本内容

`cache_control` 是 API 协议层的**元数据标记**，不是 prompt 内容。模型永远不会"看到" `cache_control`。

```json
// 请求中的结构
{ "type": "text", "text": "You are Claude Code...", "cache_control": {"type": "ephemeral"} }
                                         ↑                    ↑
                                    会被 tokenize          不会被 tokenize
                                    → 变成 token IDs      → API 服务端读取后剥离
                                    → 计算 embedding       → 不进入模型
                                    → 生成 KV cache
```

### API 服务端的处理流程

```
1. 解析 JSON 请求
2. 提取所有 text 字段的内容，拼接成完整文本
3. 读取 cache_control 位置，记录"在第 N 个 token 处存储 KV"
4. 剥离 cache_control，将纯文本送入模型
5. 模型 tokenize → embedding → 计算 → 到断点位置时，将 KV 持久化存储
```

### 从文本到 KV 的完整流程

```
文本 → tokenize → token IDs → embedding → Transformer layers → KV cache
       (BPE分词)  (整数序列)   (查表得向量)   (自注意力计算)     (键值对)

示例:
  "You are Claude Code"
       ↓ tokenize
  [745, 527, 12868, 3742]   ← token IDs
       ↓ embedding
  [[0.12,...], [0.55,...], [0.33,...], [0.88,...]]   ← 每个token 4096维向量
       ↓ Transformer 80+ 层
  每层产出 K 矩阵和 V 矩阵   ← 这就是被缓存的对象
```

**`cache_control` 只在 API 基础设施层面存在，模型看到的是纯 token 序列，没有任何"缓存标记"的概念。**

---

## 6. 请求文本拼接过程

### 结构化数据如何变成 token 流

API 服务端将请求中的 system blocks、tools、messages 按固定顺序排列，插入角色分隔符，形成一条连续的 token 流。

以一个实际的 Claude Code 请求为例：

```json
{
  "system": [
    { "type": "text", "text": "x-anthropic-billing-header: cc_version=2.1.133..." },
    { "type": "text", "text": "You are Claude Code, Anthropic's official CLI for Claude.", "cache_control": {"type": "ephemeral"} },
    { "type": "text", "text": "\nYou are an interactive agent...", "cache_control": {"type": "ephemeral"} }
  ],
  "tools": [
    { "name": "Bash", "description": "Run a shell command...", "input_schema": {...} },
    { "name": "Read", "description": "Read a file...", "input_schema": {...} }
  ],
  "messages": [
    { "role": "user", "content": "帮我修复这个 bug" },
    { "role": "assistant", "content": [{ "type": "text", "text": "让我先看看代码。" }, { "type": "tool_use", "id": "toolu_1", "name": "Read", "input": {"file_path": "/src/main.ts"} }] },
    { "role": "user", "content": [{ "type": "tool_result", "tool_use_id": "toolu_1", "content": "// file contents here..." }, { "type": "text", "text": "这是文件内容", "cache_control": {"type": "ephemeral"} }] }
  ]
}
```

模型实际看到的 token 序列（加入特殊 token 标记角色边界）：

```
[系统部分]
<system>
x-anthropic-billing-header: cc_version=2.1.133...
</system>
<system>
You are Claude Code, Anthropic's official CLI for Claude.
</system>
<system>
You are an interactive agent...
</system>

[工具部分]
<tools>
{"name": "Bash", "description": "Run a shell command...", ...}
{"name": "Read", "description": "Read a file...", ...}
</tools>

[对话部分]
<user>
帮我修复这个 bug
</user>
<assistant>
让我先看看代码。
<tool_use>
{"name": "Read", "input": {"file_path": "/src/main.ts"}}
</tool_use>
</assistant>
<user>
<tool_result>
// file contents here...
</tool_result>
这是文件内容
</user>
```

> 注：实际 Anthropic 使用的特殊 token 是 `<|begin|>`、`<|end|>`、`<|header_start|>` 等内部格式，这里用 XML 风格简化表示。

### `cache_control` 在拼接时如何起作用

```
token 位置:  0    1    2   ...  800  801  802  ...  5000  5001 ...  8200
             ↓    ↓    ↓        ↓    ↓    ↓         ↓     ↓         ↓
内容:       [system_1......] [system_2..........] [system_3...............]
             billing header   "You are Claude.."   "You are an interactive.."
                              ↑ cache_control      ↑ cache_control
                              断点1: 存KV[0..800]  断点2: 存KV[0..5000]
```

- 断点 1 处：API 计算完前 800 个 token 的 KV，**持久化存储 KV[0..800]**
- 断点 2 处：计算完前 5000 个 token 的 KV，**持久化存储 KV[0..5000]**（包含断点1的范围，但断点1仍独立可命中）

断点放在 block 边界的原因：前面稳定的内容可以跨请求复用，后面变化的内容不影响前面的缓存命中。如果把断点放在整个请求末尾，后面 tools 或 messages 任何变化都会导致全部缓存失效。

---

## 7. scope 缓存共享范围

`scope` 控制谁能命中这份缓存，由 API key 自动关联的组织 ID 决定，客户端无需配置。

### scope 类型

| scope | 谁能命中 | 匹配条件 | 典型用途 |
|-------|---------|---------|---------|
| `org`（默认） | 同组织用户 | 前缀内容相同 + org_id 相同 | tools、messages |
| `global` | 所有用户 | 前缀内容相同（不限组织） | 静态 system prompt |

### org_id 的来源

```
API key 创建时 → 绑定到组织
请求时: x-api-key: sk-ant-xxxxx → 服务端解析出 org_id
缓存匹配时: 前缀内容相同 + org_id 相同 → 命中
```

**不需要在代码或请求中配置 org_id**，API key 本身就包含了组织归属。

### 个人用户 vs 组织用户

| 用户类型 | global 缓存 | org 缓存 | 实际效果 |
|---------|:---:|:---:|------|
| 个人用户 | 可用 | 仅自己 | org 等同 user 级别，收益来自 global |
| 组织用户 | 可用 | 组织内共享 | 跨用户 + 跨会话收益 |

### 代码中的 scope 决策

`splitSysPromptPrefix()` 根据 MCP tools 是否存在决定 scope：

| 场景 | system prompt scope | 原因 |
|------|---|------|
| 有 MCP tools | `org` | MCP tools 是 per-user 的，global 会导致不同用户错误命中别人的 MCP 定义 |
| 无 MCP tools，1P 用户，有 boundary | `global` | 静态部分所有用户相同，可跨组织共享 |
| 3P provider 或无 boundary | `org` | 保守策略，避免跨组织问题 |

---

## 8. 缓存匹配逻辑

### 核心规则：累积前缀匹配

缓存匹配是**从前往后逐段匹配累积前缀**，不是独立的逐段匹配。

```
每个断点的缓存键 = 从请求开头到该断点的所有内容

断点1 key: hash(system_prompt)
断点2 key: hash(system_prompt + tools)
断点3 key: hash(system_prompt + tools + messages)
```

### 匹配机制：前缀哈希查找

缓存匹配**不是逐条消息对比**，而是通过**前缀哈希一次查找**完成。

API 服务端维护一个有序的前缀哈希索引表：

```
缓存索引:
┌──────────────────────┬──────────┐
│ hash(system)          │ KV[0..S] │  ← 断点1
│ hash(system+tools)    │ KV[0..T] │  ← 断点2
│ hash(system+tools+M)  │ KV[0..M] │  ← 断点3 (M=所有消息)
└──────────────────────┴──────────┘
```

新请求到达时：

```
1. 拼接 token 流，计算前缀哈希
2. 查缓存索引 → 从最长前缀开始尝试匹配
3. 命中 → 从缓存加载 KV，直接注入模型的 KV 状态
4. 模型只对新增部分开始计算

模型不会对 0-msgn 先算一遍再发现不命中——
缓存匹配发生在模型计算之前，是纯基础设施层的操作
```

**没有 `cache_control` 断点的位置**，即使前缀内容相同，API 也**不会**去查索引表，不会尝试匹配。断点的作用是"触发创建缓存条目"，而非"标记可匹配的位置"。

### 匹配过程示例

```
Turn N 缓存:
  断点1: hash(静态A)                        → 存入
  断点2: hash(静态A + 动态X + toolsV1)       → 存入
  断点3: hash(静态A + 动态X + toolsV1 + msgs) → 存入

Turn N+1（动态从 X 变为 Y，其余不变）:
  断点1: hash(静态A)                         → 命中 ✓
  断点2: hash(静态A + 动态Y + toolsV1)        → 未命中 ✗（累积前缀包含动态Y）
  断点3: hash(静态A + 动态Y + toolsV1 + msgs) → 未命中 ✗（同上）
```

**关键**：中间某段变化后，后续所有段即使内容没变，也因为累积前缀变了而 cache miss。**多加断点不能解决这个问题**。

### 消息增长时的匹配

```
Turn 1: system + tools + msg[0..1] [断点]
  → 缓存: 到 msg[1] 的完整前缀

Turn 2: system + tools + msg[0..3] [断点]
  → 匹配: msg[0..1] 的缓存命中 → cache_read
  → msg[2..3] → cache_creation

Turn 3: system + tools + msg[0..5] [断点]
  → 匹配: msg[0..3] 的缓存命中 → cache_read
  → msg[4..5] → cache_creation
```

这是最常见的场景：每轮只新增几条消息，旧消息全部命中缓存。

### 多轮对话的缓存演进

```
轮次  缓存命中范围            读取(0.1x)    新计算(1x)    写入(1.25x)
─────────────────────────────────────────────────────────────────
1     无                     0            全部          全部
2     system+tools+msg1      S+T+M1       msg2          全部
3     system+tools+msg1+2    S+T+M1+M2    msg3          全部
4     system+tools+msg1+2+3  S+T+M1+M2+M3 msg4          全部
```

注意：`cache_creation_input_tokens` 报告的是"到断点位置的整个前缀"的 token 数，不是只算新增部分。但实际 GPU 计算量只有新增部分——API 内部只对新增部分做了计算，缓存写入时是把旧缓存 + 新 KV 合并存储。

### 中间消息修改时的缓存影响

缓存是前缀匹配——前缀中任何位置的内容变化，从该位置往后的缓存全部失效。

```
操作                          缓存命中情况
──────────────────────────────────────────────
新增 msg4（前缀不变）          msg1-msg3 全部命中 ✓
修改 msg3（前缀断裂）          system+tools 命中，msg1-msg2 重算 ✗
删除 msg3+msg4，加 msg3_0     system+tools 命中，msg1-msg2 重算 ✗
compact 压缩                  system+tools 命中，消息全部重算 ✗
```

### 为什么不在每条消息末尾都放断点

1. **API 限制最多 4 个断点**，必须精打细算
2. **多断点浪费 KV 存储空间**：Mycro 的 local-attention eviction 机制会保护断点位置的 KV pages 不被回收，但中间断点位置永远不会被恢复使用——只有最长前缀才有匹配价值
3. 消息级缓存价值不大——同一会话中，用户几乎不会发出"前 N-1 条消息完全相同、只改最后一条"的请求

### 4 个断点的分配策略

```
断点1: system prompt 块末尾   — 最稳定，几乎不变（scope: global/org）
断点2: system prompt 末尾块   — 次稳定（scope: global/org）
断点3: tools 末尾             — 较稳定，只在增删工具时变
断点4: 最后一条消息末尾        — 覆盖整个对话历史，每轮都变
```

这是在 4 个断点限额下的最优分配：把断点放在最稳定的内容边界，确保即使对话历史被修改，最昂贵的 system prompt 缓存仍然可以命中。

---

## 9. 缓存失效场景

| 变更类型 | 缓存影响 | 失效范围 |
|---------|---------|---------|
| 新增消息 | 无影响 | 仅新消息 cache_creation |
| 修改旧消息内容 | 该消息起全部失效 | 从变更处到末尾重算 |
| `cache_edits` 删除工具结果 | 无影响 | 零（读时投影，不修改内容） |
| 新增/删除 tool | tools 断点起全部失效 | tools + messages 重算 |
| CLAUDE.md 变更（有 boundary） | 仅动态部分 | 动态部分 + tools + messages 重算 |
| CLAUDE.md 变更（无 boundary） | system 全部失效 | system + tools + messages 重算 |
| 超过 TTL（5min/1h） | 缓存被淘汰 | 全部重算 |
| Beta header 变更 | 累积前缀变化 | 从变更处重算 |

### 保护措施

代码中采用多种策略防止不必要的缓存失效：

- **Beta header sticky-on 锁定**：一旦发送过某个 beta header，后续请求都带上，防止中途增减导致失效
- **1h TTL 锁定**：会话内保持一致，防止 overage 状态变化导致 TTL 切换
- **`notifyCompaction()`**：压缩后重置 `prevCacheReadTokens` 基线，避免误报缓存中断
- **`notifyCacheDeletion()`**：标记 cache_edits 导致的预期 token 下降

---

## 10. cache_edits 读取时投影

### 原理

`cache_edits` 不修改客户端消息内容，而是在请求中附加指令，让服务端**在读取缓存后、喂给模型前**执行删除操作。

```
请求:
  messages: [...100 个工具结果...]    ← 原样发送
  cache_edits: [删除 80 个工具结果]

服务端处理:
  1. 匹配缓存键 → 命中 → cache_read
  2. 读取缓存的完整内容
  3. 在内存中按 cache_edits 删除指定 tool_result
  4. 把删减后的内容喂给模型
  5. 缓存本身不修改，下次仍可使用完整版本
```

### 类比

```
数据库表 = 缓存存储（原始数据，不变）
SELECT ... WHERE ... = cache_edits（查询时的过滤条件）

过滤掉 80 行不影响表中存了 100 行
下次换个 WHERE 还能再查，表还是那个表
```

### 投影删除的 Attention 层面原理

`cache_edits` 的"删除"不是修改 KV Cache 数据，而是在 Attention 计算时跳过被标记的 token 行。

#### KV Cache 数据不变

```
原始 KV Cache (5 个 token):

Layer 0 的 K:           Layer 0 的 V:
tok_0: [0.23, ...]     tok_0: [0.91, ...]    ← 被标记删除
tok_1: [1.05, ...]     tok_1: [-0.17, ...]   ← 正常
tok_2: [-0.48, ...]    tok_2: [0.44, ...]    ← 被标记删除
tok_3: [0.67, ...]     tok_3: [0.88, ...]    ← 正常
tok_4: [0.31, ...]     tok_4: [0.23, ...]    ← 正常

缓存数据: 完整保留，5 行全在，没有删除任何一行
```

#### Attention 计算时的跳过

```
无投影删除:                              有投影删除:

Q × 所有 K:                              Q × 未删除的 K:
  tok_0: 0.82    ← 正常参与               tok_0: 跳过 ✗
  tok_1: 0.45                                  tok_1: 0.45
  tok_2: 0.15    ← 正常参与               tok_2: 跳过 ✗
  tok_3: 0.67                                  tok_3: 0.67
  tok_4: 0.33                                  tok_4: 0.33

归一化: [0.35, 0.19, 0.06, 0.28, 0.12]   归一化: [0.28, 0.42, 0.30]

加权取 V: 全部 5 行参与                    加权取 V: 只取 tok_1, 3, 4
```

被标记删除的 token：K 仍存在但匹配度强制归零，V 仍存在但因为匹配度为零不会被提取。KV 数据本身完好无损，所以缓存可以原样复用。

#### 间接信息的局限

投影删除是**近似删除**，不是精确删除。原因：

```
tok_0 → tok_1 → tok_2 → tok_3 → tok_4 → tok_5 (新)

假设 tok_3 被标记删除:

tok_4 的 K/V 是在生成时"看过" tok_3 后计算出来的
→ tok_4 的向量中间接包含了 tok_3 的信息
→ 投影删除只阻止新 token 直接关注 tok_3
→ 但 tok_3 对 tok_4 的影响已经"烙印"在 tok_4 的 K/V 中
```

实际效果：新 token 不能**直接**看到被删除的 token，但可能通过其他 token **间接**保留部分痕迹。不过直接关注的权重远大于间接影响，投影删除仍然有效。

**类比**：像把一本书的某页撕掉，读者无法直接翻到那一页，但其他页可能引用了那页的内容。直接信息被切断，间接痕迹可能残留。

### 为什么不改消息内容？

| 方案 | 缓存命中？ | 成本 |
|------|:---:|------|
| 直接改消息内容 | miss | 全部重算（cache_creation，贵） |
| `cache_edits` | hit | 只删除指定部分（cache_read，便宜） |

### 限制

- 仅 Anthropic Claude 4.x 模型支持（`/claude-[a-z]+-4[-\d]/`）
- 需要环境变量 `CLAUDE_CACHED_MICROCOMPACT=1` 启用
- 仅主线程使用，子 agent 不使用（防止全局状态冲突）

### 与时间驱动 microcompact 的对比

| 维度 | 时间驱动 | 缓存驱动（cache_edits） |
|------|---------|------|
| 触发条件 | 距上次 assistant ≥ 60 分钟 | 活跃工具结果 ≥ 10 个 |
| 缓存状态 | 冷（已过期） | 热（未过期） |
| 客户端消息 | **修改**（内容替换为占位文本） | **不变** |
| 缓存影响 | miss（缓存已冷，改了也不浪费） | hit（缓存仍热，不能破坏） |
| 互斥关系 | 优先执行，命中则短路 | 时间驱动未命中才走到这里 |

---

## 11. SYSTEM_PROMPT_DYNAMIC_BOUNDARY

### 问题

System prompt 中既有所有用户相同的静态指令，也有每会话不同的动态内容（CLAUDE.md 等）。如果合并缓存，动态部分一变就导致整个 system prompt 缓存失效。

### 解决方案

`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`（`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`）是一个分界线，将 system prompt 分成两段：

```
┌─ attribution header ──────────────┐ 无缓存标记
├─ 静态部分 [cache_control, global] │ ← 所有用户相同，跨组织共享
├─ DYNAMIC_BOUNDARY ────────────────┤ ← 分界线
├─ 动态部分（无缓存标记）────────────│ ← CLAUDE.md 等，每会话不同
```

### 效果

```
有 boundary:
  静态部分 → global 缓存，跨用户共享，几乎永不过期
  动态部分 → 无缓存，每次重算
  动态变了 → 静态部分缓存不受影响

无 boundary:
  整个 system prompt 作为一个缓存段
  CLAUDE.md 变了 → 整个 system 缓存失效 → tools + messages 全部重算
```

### 仅 1P 用户生效

3P provider 的 system prompt 结构不同，boundary 机制不适用，回退到 `org` scope 的整段缓存。

---

## 12. promptCacheBreakDetection 缓存中断检测

**文件**: `src/services/api/promptCacheBreakDetection.ts`
**Feature Flag**: `PROMPT_CACHE_BREAK_DETECTION`

### 两阶段监控

#### 阶段一：请求前（`recordPromptState()`）

计算当前请求的多个 hash，与上次比较：

- `systemHash` — system prompt 内容 hash
- `toolsHash` — tools schema hash
- `cacheControlHash` — cache_control 字段 hash（捕获 scope/TTL 变化）
- `perToolHashes` — 每个 tool 的独立 hash
- 其他：model、beta headers、fastMode、effortValue 等

任何 hash 变化 → 记录 `PendingChanges` 对象（包含具体差异）。

#### 阶段二：响应后（`checkResponseForCacheBreak()`）

检查 API 响应中的 `cache_read_input_tokens` 是否显著下降：

```typescript
const tokenDrop = prevCacheRead - cacheReadTokens
// 缓存中断判定：下降超过 5% 且绝对值超过 2,000 token
if (cacheReadTokens >= prevCacheRead * 0.95 || tokenDrop < 2000) {
  // 非显著下降，忽略
}
```

检测到缓存中断时：
1. 记录 `tengu_prompt_cache_break` 分析事件
2. 写 diff 文件到 `$CLAUDE_TEMP_DIR/cache-break-XXXX.diff`
3. 判断是 TTL 过期还是客户端变更导致

---

## 13. 不同 Provider 的缓存差异

| Provider | 缓存机制 | cache_control | cache_edits | scope | 隔离方式 |
|----------|---------|:---:|:---:|:---:|------|
| Anthropic 直接 | 原生 Prompt Cache | 支持 | 支持（4.x） | global/org | 按 org_id |
| AWS Bedrock | AWS 侧缓存 | 部分支持 | 不支持 | org | 按 AWS account |
| Google Vertex | Google 侧缓存 | 部分支持 | 不支持 | org | 按 GCP project |
| OpenAI 官方 | 自动前缀缓存（无需标记） | 不需要 | 不适用 | 自动 | 按 API key |
| OpenAI 兼容 | 看端点（vLLM 有，Ollama 部分） | 不支持 | 不适用 | 无 | 无 |
| Gemini | Context Caching API | 不兼容 | 不适用 | 按 project | 按 GCP project |
| Grok | 未公开 | 不支持 | 不适用 | 无 | 无 |

### 关键区别

**Anthropic 的缓存是显式的**：客户端用 `cache_control` 标记断点，服务端按标记缓存。

**OpenAI 的缓存是隐式的**：不需要标记，服务端自动检测前缀匹配并缓存，客户端无感知。

### 代码中的降级处理

```typescript
// 只有 Anthropic 原生 API 才使用完整的 cache_control 体系
const enablePromptCaching = isFirstParty || isBedrock || isVertex

// OpenAI 兼容层：不发送 cache_control
// Gemini 层：使用 Gemini 自己的 Context Caching API
// cache_edits：仅 Claude 4.x 模型支持
```

### 非 Anthropic Provider 的压缩策略

| 压缩机制 | Anthropic | 其他 Provider |
|---------|:---:|:---:|
| 时间驱动 microcompact（改消息内容） | 缓存冷时使用 | 始终有效（无缓存可保护） |
| 缓存驱动 microcompact（cache_edits） | 缓存热时使用 | 不适用 |
| Session Memory Compact | 可用 | 可用 |
| Full Compact（LLM 摘要） | 可用 | 可用 |

---

## 14. 缓存与压缩的协作关系

缓存优化和上下文压缩是互补的：

```
缓存优化的目标: 让尽可能多的 token 走 cache_read（便宜）
压缩优化的目标: 让模型实际消费的 token 尽可能少（省上下文窗口）

两者冲突的场景:
  压缩改了消息内容 → 缓存失效 → 更贵的 cache_creation

解决方案:
  cache_edits → 不改消息 → 缓存命中 → 模型看到的内容少了
  → 同时实现"缓存命中"和"上下文压缩"
```

### 决策流程

```
microcompactMessages()
  │
  ├─ 缓存冷了（≥60分钟）？
  │   └─ 时间驱动：直接改消息内容（反正缓存已冷，改了也不浪费）
  │
  ├─ 缓存仍热 + 支持 cache_edits？
  │   └─ 缓存驱动：用 cache_edits 删除（保持缓存命中）
  │
  └─ 缓存仍热 + 不支持 cache_edits？
      └─ 不做 microcompact（交给 autocompact 处理）
```

---

## 关键文件清单

| 文件 | 路径 | 职责 |
|------|------|------|
| `claude.ts` | `src/services/api/claude.ts` | API 客户端，构建请求参数，添加 `cache_control` 断点，`addCacheBreakpoints()` 消息级缓存标记 |
| `api.ts` | `src/utils/api.ts` | `splitSysPromptPrefix()`（system prompt 分段）、`toolToAPISchema()`（tool 缓存标记） |
| `modelCost.ts` | `src/utils/modelCost.ts` | 缓存计费：`tokensToUSDCost()` 分别计算 cache_creation (1.25x)、cache_read (0.1x)、input (1x) 的费用 |
| `promptCacheBreakDetection.ts` | `src/services/api/promptCacheBreakDetection.ts` | 缓存中断检测（两阶段监控：请求前 hash + 响应后 token 下降） |
| `microCompact.ts` | `src/services/compact/microCompact.ts` | 时间驱动 microcompact（改消息内容） |
| `cachedMicrocompact.ts` | `src/services/compact/cachedMicrocompact.ts` | 缓存驱动 microcompact（`cache_edits` 投影） |
| `compact.ts` | `src/services/compact/compact.ts` | Full compact，含缓存前缀共享策略（`tengu_compact_cache_prefix`） |
| `timeBasedMCConfig.ts` | `src/services/compact/timeBasedMCConfig.ts` | 时间驱动 microcompact 配置（GrowthBook `tengu_slate_heron`） |
| `prompts.ts` | `src/constants/prompts.ts` | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 常量定义，静态/动态 section 划分 |
