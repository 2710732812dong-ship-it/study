# Transformer 推理数据流：Prompt → Input → KV Cache → Output

本文档从数据流角度，完整记录用户提示词（Prompt）到模型输出（Output）之间每个环节的计算过程、数据形态和数量关系。

---

## 目录

- [1. 端到端数据流总览](#1-端到端数据流总览)
- [2. Tokenization：文本 → Token ID](#2-tokenization文本--token-id)
- [3. Embedding：Token ID → 向量](#3-embeddingtoken-id--向量)
- [4. Transformer 层：逐层计算 K/V](#4-transformer-层逐层计算-kv)
- [5. KV Cache：存储结构与容量](#5-kv-cache存储结构与容量)
- [6. 自回归生成：逐 token 输出](#6-自回归生成逐-token-输出)
- [7. 多轮对话中的 KV Cache 演变](#7-多轮对话中的-kv-cache-演变)
- [8. Prompt Cache 与 KV Cache 的关系](#8-prompt-cache-与-kv-cache-的关系)
- [9. cache_edits 投影删除的计算影响](#9-cache_edits-投影删除的计算影响)
- [10. 完整数字示例](#10-完整数字示例)

---

## 1. 端到端数据流总览

```
用户输入                   服务端处理
┌──────────────┐          ┌────────────────────────────────────────────────┐
│              │          │                                                │
│  Prompt      │  HTTP    │  1. Tokenization:  文本 → Token IDs            │
│  (用户提示词) │ ──────→ │  2. Embedding:      IDs → 向量序列             │
│              │          │  3. Transformer:    逐层计算 → 产生 K/V       │
│              │          │  4. KV Cache:       存储 K/V 矩阵             │
│              │          │  5. 自回归生成:     逐 token 产出输出          │
│              │  ←────── │                                                │
│              │  流式响应 │  Output: token by token                        │
└──────────────┘          └────────────────────────────────────────────────┘
```

**五个核心概念**：

| 概念 | 是什么 | 数据形态 | 大小估算 |
|------|--------|---------|---------|
| **Prompt** | 用户发送的完整文本 | 字符串 | 按字符计 |
| **Input Tokens** | Prompt 经分词后的 token 序列 | 整数数组 [id₁, id₂, ...] | 约 1 token ≈ 4 字符（英文） |
| **KV Cache** | 所有 token 在每层的 K/V 向量 | 浮点矩阵 | 约 80 KB/token |
| **Model Output** | 模型生成的 token 序列 | 整数 → 解码为文本 | 逐 token 流式输出 |
| **Usage** | API 返回的计费统计 | cache_read / cache_creation / input / output | 按 token 计费 |

---

## 2. Tokenization：文本 → Token ID

### 过程

Tokenizer（BPE 算法）将文本拆分为子词单元，每个子词映射为一个整数 ID：

```
输入: "1+1="

BPE 分词:
  "1"  → id=16
  "+"  → id=412
  "1"  → id=16
  "="  → id=289

输出: [16, 412, 16, 289]    ← 4 个 token
```

### 分词规则

- 常见词通常 1 个 token（如 "hello" → 1 token）
- 中文通常 1-3 个字 = 1 token
- 代码中的缩进、空格各占 token
- 特殊标记（如 `<|end|>`）占 1 token
- 合并规则取决于训练语料中子词的共现频率

### Token 数 ≠ 字符数

```
"1+1="         → 4 token（可能 3-5，取决于 tokenizer）
"hello world"  → 2 token
"你好世界"      → 约 2-4 token
"  if (x > 0)" → 约 6-8 token（含缩进和空格）
```

### 数量关系

```
Input Token 数 = len(tokenizer.encode(text))

Prompt 大小（token）= system_prompt tokens + tools tokens + messages tokens
```

---

## 3. Embedding：Token ID → 向量

### 过程

每个 Token ID 查询 Embedding 矩阵，得到一个固定维度的向量：

```
Embedding 矩阵 (vocab_size × d_model):
         dim_0   dim_1   ...  dim_4095
id=0:    [0.012, -0.034, ...  0.056]
id=1:    [0.078,  0.091, ... -0.023]
  ...
id=16:   [0.234, -0.567, ...  0.123]    ← "1" 的 embedding
id=289:  [-0.345, 0.678, ...  0.234]    ← "=" 的 embedding
  ...
id=50000:[0.456, -0.789, ... -0.012]

查表过程:
  id=16  → [0.234, -0.567, ... 0.123]
  id=412 → [0.891,  0.012, ... -0.345]
  id=16  → [0.234, -0.567, ... 0.123]  ← 同一 ID，同一向量
  id=289 → [-0.345, 0.678, ... 0.234]
```

### 关键特性

- **确定性**：相同 token ID 永远映射到相同向量
- **上下文无关**：Embedding 只看 token 本身，不看上下文
- **不需要缓存**：因为确定性 + 上下文无关，重新查表即可

### 输出

```
4 个 token → 4 个向量，每个 d_model 维（如 4096 或 8192）

输入到 Transformer: [vec_0, vec_1, vec_2, vec_3]    ← (4 × d_model) 矩阵
```

---

## 4. Transformer 层：逐层计算 K/V

### 单层处理流程

每个 Transformer 层做两件事：**Self-Attention** + **Feed-Forward Network**。

```
输入向量序列: [h₀, h₁, h₂, h₃]    ← 4 个 token 的隐藏向量

Step 1 - Self-Attention:
  每个 hᵢ 通过三个投影矩阵产生 Q, K, V:
    Qᵢ = hᵢ × W_Q     ← 查询向量（仅当前 token 需要，不缓存）
    Kᵢ = hᵢ × W_K     ← 键向量（存入 KV Cache）
    Vᵢ = hᵢ × W_V     ← 值向量（存入 KV Cache）

  Attention 计算:
    scores = Q × Kᵀ           ← 每个 token 的 Q 与所有 K 做点积
    weights = softmax(scores)  ← 归一化为概率分布
    output = weights × V       ← 按权重提取 V，加权求和

Step 2 - Feed-Forward Network:
  output → FFN → 新的隐藏向量 h'ᵢ    ← 逐 token 独立处理

输出: [h'₀, h'₁, h'₂, h'₃]    ← 传给下一层
```

### 多层串叠

```
Embedding:  [e₀, e₁, e₂, e₃]
              ↓
Layer 0:    Self-Attention(产生 K₀,V₀) + FFN → [h₀⁰, h₁⁰, h₂⁰, h₃⁰]
              ↓
Layer 1:    Self-Attention(产生 K₁,V₁) + FFN → [h₀¹, h₁¹, h₂¹, h₃¹]
              ↓
Layer 2:    Self-Attention(产生 K₂,V₂) + FFN → [h₀², h₁², h₂², h₃²]
              ↓
  ...
              ↓
Layer N-1:  Self-Attention(产生 Kₙ₋₁,Vₙ₋₁) + FFN → [h₀ⁿ, h₁ⁿ, h₂ⁿ, h₃ⁿ]
              ↓
          最终隐藏向量 → 预测下一个 token
```

### 层的语义含义

| 层级 | 捕捉的信息 | 示例 |
|------|-----------|------|
| 浅层（0-10） | 基础特征：词性、局部搭配 | "猫"→名词，"坐"→动词 |
| 中层（10-40） | 词间关系：主谓、修饰、指代 | "猫"是"坐"的主语 |
| 深层（40-80） | 句法结构、语义意图 | 整句描述一个静态场景 |

### Q/K/V 的维度

Q、K、V 的维度通常小于 d_model，因为使用多头注意力（Multi-Head Attention）：

```
d_model = 8192（总隐藏维度）
num_heads = 64（注意力头数）
head_dim = 128（每个头的维度）= d_model / num_heads

每个 token 在每层的 K 维度 = head_dim = 128
每个 token 在每层的 V 维度 = head_dim = 128
```

---

## 5. KV Cache：存储结构与容量

### 存储内容

```
KV Cache = 每个 token × 每层的 K 向量和 V 向量

对于 N 个 token、L 层、d 维:
  K 矩阵: L 个 (N × d) 矩阵
  V 矩阵: L 个 (N × d) 矩阵
  总计: 2 × L × N × d 个 float
```

### 具体示例：2 token × 3 层 × 4 维

```
Layer 0:
  K (2×4):                V (2×4):
       d0    d1   d2  d3       d0    d1    d2    d3
  tok0: [0.23, -0.71, 0.55, 0.12]   tok0: [0.91, 0.22, -0.33, 0.65]
  tok1: [1.05, 0.33, -0.48, 0.92]  tok1: [-0.17, 0.88, 0.44, -0.56]

Layer 1:  (每层值不同)
  K (2×4):                V (2×4):
       d0    d1   d2  d3       d0    d1    d2    d3
  tok0: [0.55, 0.12, -0.89, 0.44]  tok0: [-0.55, 0.44, 0.67, 0.33]
  tok1: [0.31, -0.67, 1.02, 0.78]  tok1: [0.89, -0.11, 0.23, -0.55]

Layer 2:
  K (2×4):                V (2×4):
       d0    d1   d2  d3       d0    d1    d2    d3
  tok0: [0.11, -0.95, 0.73, 0.58]  tok0: [0.78, 0.15, -0.44, 0.67]
  tok1: [-0.36, 0.29, 0.88, -0.41] tok1: [0.23, -0.55, 0.91, -0.33]
```

### 容量计算

```
假设: 层数 L=80, 维度 d=128, float32 (4 bytes)

1 个 token:
  80 layers × 2 (K+V) × 128 dim × 4 bytes = 81,920 bytes ≈ 80 KB

1,000 个 token:
  81,920 × 1,000 = ~80 MB

100,000 个 token:
  81,920 × 100,000 = ~8 GB
```

### 为什么 KV Cache 是稀缺资源

```
GPU 显存分配:
  模型参数:  ~40 GB (175B 模型)
  KV Cache:  ~8 GB (100k token 上下文)
  其他开销:  ~4 GB

→ 100k token 的 KV Cache 占用约 8 GB 显存
→ 服务端需要为每个活跃会话维护一份
→ 因此有 TTL 过期机制和 token 计费
```

---

## 6. 自回归生成：逐 token 输出

### 生成过程

模型**一次只输出一个 token**，每生成一个 token，其 K/V 追加到缓存中：

```
已有 KV Cache: [tok_0, tok_1, tok_2, tok_3]  ← 4 行

Step 1 - 预测 tok_4:
  tok_4 的隐藏向量 → 产生 Q₄, K₄, V₄
  Q₄ × [K₀, K₁, K₂, K₃, K₄]ᵀ → attention scores → softmax
  scores × [V₀, V₁, V₂, V₃, V₄] → 输出向量
  输出向量 → logits → 采样 → tok_4 = "2"    ← 输出第一个 token

  K₄, V₄ 追加到缓存: [tok_0, tok_1, tok_2, tok_3, tok_4]  ← 5 行

Step 2 - 预测 tok_5:
  tok_5 产生 Q₅, K₅, V₅
  Q₅ × [K₀..K₅]ᵀ → scores → softmax
  scores × [V₀..V₅] → 输出
  采样 → tok_5 = "\n"

  K₅, V₅ 追加到缓存: [tok_0..tok_5]  ← 6 行

  ... 重复直到遇到 <EOS> 或达到 max_tokens
```

### 计算量对比

```
无 KV Cache（每步重算所有 token 的 K/V）:
  第 1 步: 计算 4 个 token × 80 层 的 K/V
  第 2 步: 计算 5 个 token × 80 层 的 K/V
  第 3 步: 计算 6 个 token × 80 层 的 K/V
  总计: (4+5+6+...+(4+T)) × 80 层  ← O(T²) 复杂度

有 KV Cache（只算新 token 的 K/V，旧 token 直接查缓存）:
  每步: 计算 1 个 token × 80 层 的 K/V + 查缓存
  总计: T × 80 层  ← O(T) 复杂度
```

### 输出 token 不进入 Prompt Cache

```
生成 tok_4 时:
  tok_4 的 K/V 被追加到本轮的 KV Cache 中（GPU 显存内）
  → 用于生成 tok_5, tok_6, ...

本轮请求结束后:
  KV Cache 存入 Prompt Cache 的部分 = 输入 token 的 K/V
  输出 token 的 K/V 不单独缓存

下一轮请求:
  输入 = 旧上下文 + 上轮输出 + 新消息
  缓存匹配: 命中旧上下文部分（cache_read）
  上轮输出 + 新消息: cache_creation
```

---

## 7. 多轮对话中的 KV Cache 演变

### 第一轮：用户问 "1+1="

```
客户端发送:
  system(5k) + tools(20k) + messages: [{"role":"user","content":"1+1="}]

服务端:
  Tokenization: "1+1=" → [16, 412, 16, 289] (4 token)
  Embedding: 4 个向量
  Transformer: 4 个 token × 80 层 → 产生 K/V (4行×80层×2)
  自回归生成: "2" → 1 个输出 token

KV Cache 存储 (Prompt Cache):
  system(5k) + tools(20k) + "1+1=" → cache_creation: 25k token

API 返回 usage:
  cache_creation_input_tokens:  25,004  (5k + 20k + 4)
  input_tokens:                 25,004
  output_tokens:                1
```

### 第二轮：用户问 "+1="

```
客户端发送:
  system(5k) + tools(20k) + messages: ["1+1=", "2", "+1="]

服务端:
  缓存匹配: 前 25,004 token 命中 → cache_read: 25,004
  新增: "2" + "+1=" → cache_creation: 3 token
  自回归生成: "3" → 1 个输出 token

KV Cache 存储 (新增部分):
  "2" + "+1=" → cache_creation: 3 token
  总缓存: 25,007 token

API 返回 usage:
  cache_read_input_tokens:      25,004  ← 便宜 (1/12.5 价格)
  cache_creation_input_tokens:  3
  input_tokens:                 25,007
  output_tokens:                1
```

### 缓存逐轮增长

```
轮次   cache_read    cache_creation   总缓存    说明
  1         0            25,004       25,004   首轮全部 creation
  2    25,004                3        25,007   几乎全部 hit
  3    25,007               10        25,017   追加新消息
  4    25,017                5        25,022   ...
  ...
 20    25,080              15        25,095   缓存持续增长
```

### 缓存过期场景

```
轮次 20 结束 → 用户离开 6 分钟 → TTL(5min) 过期

轮次 21:
  cache_read: 0        ← 缓存全部失效
  cache_creation: 25,095  ← 全部重算（贵）
```

---

## 8. Prompt Cache 与 KV Cache 的关系

### 两者不是同一个东西

| | KV Cache | Prompt Cache |
|---|---|---|
| **本质** | Transformer 计算产生的 K/V 矩阵 | 服务端对 KV Cache 的持久化存储 |
| **生命周期** | 单次请求内存在 GPU 显存 | 跨请求持久化在服务端存储 |
| **触发** | 每次推理自动产生 | 客户端用 `cache_control` 标记才缓存 |
| **匹配** | 无（每次重新计算） | 按前缀哈希匹配 |
| **TTL** | 无（请求结束即释放） | 5min / 1h |

### 关系

```
没有 Prompt Cache:
  每次请求 → 全部 token 从头计算 → 产生 KV Cache → 推理 → 释放 KV Cache
  → 重复计算成本高

有 Prompt Cache:
  首次请求 → 计算 → KV Cache → 存入 Prompt Cache
  后续请求 → 前缀匹配 → 直接加载 Prompt Cache 中的 KV → 只算新增 token
  → 重复部分零计算成本
```

### 计费映射

```
cache_creation_input_tokens = 写入 Prompt Cache 的 token 数（首次，贵）
cache_read_input_tokens     = 从 Prompt Cache 读取的 token 数（命中，便宜）
input_tokens                = 模型实际参与计算的 token 数
output_tokens               = 模型输出的 token 数

关系:
  input_tokens = cache_read + cache_creation + 非缓存输入
  cache_deleted = 从缓存读取但被 cache_edits 投影删除的 token
  模型实际计算量 = input_tokens - cache_deleted
```

---

## 9. cache_edits 投影删除的计算影响

### 不修改 KV Cache 数据

```
5 个 token 的 KV Cache:

Layer 0 的 K:           Layer 0 的 V:
tok_0: [0.23, ...]     tok_0: [0.91, ...]    ← 被标记删除
tok_1: [1.05, ...]     tok_1: [-0.17, ...]   ← 正常
tok_2: [-0.48, ...]    tok_2: [0.44, ...]    ← 被标记删除
tok_3: [0.67, ...]     tok_3: [0.88, ...]    ← 正常
tok_4: [0.31, ...]     tok_4: [0.23, ...]    ← 正常

缓存数据: 5 行完整保留，没有删除任何一行
```

### Attention 计算时的跳过

```
无投影删除:
  Q × 所有 K → [0.82, 0.45, 0.15, 0.67, 0.33]
  softmax → [0.35, 0.19, 0.06, 0.28, 0.12]
  加权 V = 0.35×V₀ + 0.19×V₁ + 0.06×V₂ + 0.28×V₃ + 0.12×V₄

有投影删除 (tok_0, tok_2):
  Q × K → tok_0 跳过, tok_2 跳过
  有效分数: [_, 0.45, _, 0.67, 0.33]
  softmax → [_, 0.31, _, 0.44, 0.25]
  加权 V = 0.31×V₁ + 0.44×V₃ + 0.25×V₄
```

### 间接信息残留

投影删除是近似删除。被删除 token 的信息可能通过其他 token 间接保留：

```
tok_0 → tok_1 → tok_2 → tok_3 → tok_4

tok_3 被删除:
  tok_4 的 K/V 在生成时"看过" tok_3
  → tok_4 的向量中间接包含 tok_3 的信息
  → 投影删除阻止新 token 直接关注 tok_3
  → 但 tok_3 对 tok_4 的影响已"烙印"在 tok_4 的 K/V 中

效果:
  直接信息: 被切断（新 token 无法直接关注 tok_3）
  间接信息: 可能残留（通过 tok_4 等中间 token 传递）
  实际影响: 直接关注权重远大于间接影响，投影删除仍有效
```

### 对计费的影响

```
有 cache_edits 的请求:
  cache_read:       100k   ← 全部从缓存读取（缓存数据完整）
  cache_creation:     5k   ← 新增消息
  cache_deleted:     20k   ← 投影删除的部分
  input_tokens:      85k   ← = 100k - 20k + 5k（模型实际计算量）
```

---

## 10. 完整数字示例

### 场景：5 轮对话，含 cache_edits 投影删除

**配置**: system_prompt 5k, tools 20k, 模型 80 层 × 128 维

```
轮次 1: 用户问 "1+1="
─────────────────────────────────────────
  输入 tokens: 5k(system) + 20k(tools) + 4("1+1=") = 25,004
  缓存: cache_creation = 25,004
  KV Cache 存储: 25,004 × 80 × 2 × 128 × 4 bytes ≈ 2 GB
  输出: "2" (1 token)
  费用: 25,004 × $3.75/M = $0.0938 (cache_creation 价格)

轮次 2: 用户问 "+1="
─────────────────────────────────────────
  输入 tokens: 25,004(缓存) + 3(新) = 25,007
  缓存: cache_read = 25,004, cache_creation = 3
  KV Cache 存储: 25,007 × 80 × 2 × 128 × 4 bytes ≈ 2 GB
  输出: "3" (1 token)
  费用: 25,004 × $0.30/M + 3 × $3.75/M = $0.0075 + $0.0000 = $0.0076

轮次 3: 用户问 "+1=" (触发 cache_edits 删除 "1+1=" 的 4 token)
─────────────────────────────────────────
  输入 tokens: 25,007(缓存) + 3(新) = 25,010
  缓存: cache_read = 25,007, cache_creation = 3
  cache_edits: 删除 4 token
  cache_deleted: 4
  模型实际计算: 25,010 - 4 = 25,006
  KV Cache 存储: 25,010 行（完整），但 Attention 只用 25,006 行
  输出: "4" (1 token)
  费用: 25,007 × $0.30/M + 3 × $3.75/M ≈ $0.0076

轮次 4: 用户离开 6 分钟后回来
─────────────────────────────────────────
  缓存 TTL(5min) 过期 → 全部失效
  输入 tokens: 25,010 全部重算
  缓存: cache_creation = 25,010
  KV Cache 存储: 重新计算全部 ≈ 2 GB
  输出: "5" (1 token)
  费用: 25,010 × $3.75/M = $0.0938 (又变贵了)

轮次 5: 用户继续问
─────────────────────────────────────────
  输入 tokens: 25,010(缓存) + 10(新) = 25,020
  缓存: cache_read = 25,010, cache_creation = 10
  费用: 25,010 × $0.30/M + 10 × $3.75/M ≈ $0.0079
```

### 费用对比

```
轮次   cache_read   cache_creation   费用      省了多少
  1         0          25,004       $0.0938     —
  2    25,004               3       $0.0076    92%
  3    25,007               3       $0.0076    92%
  4         0          25,010       $0.0938    0% (TTL 过期)
  5    25,010              10       $0.0079    92%
```

### KV Cache 容量随轮次变化

```
轮次   token 数   KV Cache 大小    说明
  1    25,004     ~2.0 GB         首轮全量
  2    25,007     ~2.0 GB         微增
  3    25,010     ~2.0 GB         微增（cache_edits 不减存储）
  4    25,010     ~2.0 GB         重算（TTL 过期后重建）
  5    25,020     ~2.0 GB         持续增长
  ...
 50    25,200     ~2.0 GB         microcompact 开始回收
 100    50,000     ~4.0 GB         增长显著
 200   167,000    ~13.4 GB        触发 AutoCompact 阈值
       ↓ Full Compact
       25,000     ~2.0 GB         压缩回来
```

---

## 与其他文档的关系

| 文档 | 覆盖范围 | 与本文档的区别 |
|------|---------|--------------|
| [prompt-cache.md](./prompt-cache.md) | Prompt Cache 的 API 机制、cache_control、cache_edits、scope、缓存中断检测 | 侧重 API 层面的缓存策略，本文档侧重计算过程和数据形态 |
| [context-compaction.md](./context-compaction.md) | 四层压缩机制的触发逻辑和执行流程 | 侧重压缩策略的选择和执行，本文档侧重压缩对 KV Cache 的影响 |
| **本文档** | Prompt → Token → Embedding → KV Cache → Output 的完整计算链路 | 侧重数据在每一步的形态、大小和转换关系 |
