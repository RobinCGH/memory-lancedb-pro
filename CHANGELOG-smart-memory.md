# memory-lancedb-pro 智能记忆增强 — 修改说明

> **日期**: 2026-03-03  
> **版本**: v1.1.0 (Smart Memory Enhancement)  
> **目标**: 融入 epro-memory 和 memx-memory 的最优实践，提升记忆写入质量与生命周期管理

---

## 一、变更摘要

本次增强围绕三个核心主题展开：

| 主题         | 来源项目                  | 核心改进                                      |
| ------------ | ------------------------- | --------------------------------------------- |
| 智能记忆提取 | epro-memory               | LLM 6 类别提取替代正则触发，L0/L1/L2 分层存储 |
| 记忆衰减模型 | memx-memory               | Weibull 拉伸指数衰减，三层晋升/降级           |
| 智能去重管线 | epro-memory + memx-memory | 向量预过滤 + LLM 决策 (CREATE/MERGE/SKIP)     |

---

## 二、新增文件 (6 个)

### 1. `src/memory-categories.ts`

**6 类别分类系统**，移植自 epro-memory / OpenViking：

- **UserMemory**: `profile`（身份）、`preferences`（偏好）、`entities`（实体）、`events`（事件）
- **AgentMemory**: `cases`（问题-方案对）、`patterns`（可复用流程）

定义了类别感知的合并策略：

- `profile` → 始终合并（跳过去重）
- `preferences` / `entities` / `patterns` → 支持合并
- `events` / `cases` → 仅 CREATE 或 SKIP（独立记录，不合并）

同时定义了 `MemoryTier` 类型（`core` / `working` / `peripheral`）和 `CandidateMemory`、`DedupResult`、`ExtractionStats` 等类型。

---

### 2. `src/llm-client.ts`

**LLM 客户端**，复用现有 OpenAI SDK 依赖：

- `createLlmClient(config)` → 工厂函数
- `completeJson<T>(prompt)` → 发送提示并解析 JSON 响应
- 内置 JSON 容错解析：支持 markdown 代码块包裹（` ```json ``` `）和平衡大括号提取
- 低温度 (0.1) 保证输出稳定性
- 30 秒超时保护

---

### 3. `src/extraction-prompts.ts`

**3 个 LLM 提示模板**，移植自 epro-memory/prompts.ts（OpenViking 源）：

| 函数                                        | 用途                                                |
| ------------------------------------------- | --------------------------------------------------- |
| `buildExtractionPrompt(text, user)`         | 从对话中提取 6 类别 L0/L1/L2 记忆，含 few-shot 示例 |
| `buildDedupPrompt(candidate, existing)`     | CREATE / MERGE / SKIP 去重决策                      |
| `buildMergePrompt(existing, new, category)` | 将新旧记忆合并为三层结构                            |

提取提示包含：

- 记忆价值判断标准（个性化 / 长期性 / 具体性）
- 核心决策逻辑表（问题 → 类别映射）
- 常见混淆澄清（"计划做 X" → events 而非 entities）
- 三层结构定义（L0 索引 / L1 结构化摘要 / L2 完整叙述）
- 6 个 few-shot 示例（每类别一个）

---

### 4. `src/smart-extractor.ts`

**智能提取管线**，核心类 `SmartExtractor`，流水线：

```
对话文本 → LLM 提取 → CandidateMemory[] → 向量去重 → LLM 决策 → 持久化
```

关键方法：

- `extractAndPersist(conversationText, sessionKey)` — 主入口，返回统计
- `extractCandidates(text)` — LLM 提取候选记忆
- `deduplicate(candidate, vector)` — 两阶段去重：
  1. **向量预过滤**: embedding 相似度 ≥ 0.7 的 top 5
  2. **LLM 决策**: 送入 top 3 相似记忆，由 LLM 判断 CREATE/MERGE/SKIP
- `handleProfileMerge()` — Profile 类别始终合并
- `handleMerge(candidate, matchId)` — LLM 驱动的记忆合并

**向后兼容设计**：

- 6 类别映射到现有 5 类别存储类型（`profile→fact`, `preferences→preference`, `entities→entity`, `events→decision`, `cases→fact`, `patterns→other`）
- L0/L1/L2 存储在 `metadata` JSON 字段中（无需修改 LanceDB schema）
- 按类别设置默认重要度（profile: 0.9, patterns: 0.85, cases: 0.8, preferences: 0.8, entities: 0.7, events: 0.6）

---

### 5. `src/decay-engine.ts`

**Weibull 衰减引擎**，移植自 memx-memory/src/memory/decay.ts：

复合衰减分数公式：

```
composite = recencyWeight × recency + frequencyWeight × frequency + intrinsicWeight × intrinsic
```

三个分量：
| 分量 | 公式 | 含义 |
|------|------|------|
| **recency** | `exp(-λ × t^β)` | Weibull 拉伸指数衰减，importance 调制半衰期 |
| **frequency** | `(1 - exp(-n/5)) × (0.5 + 0.5 × recentnessBonus)` | 对数饱和 + 时间加权访问模式 |
| **intrinsic** | `importance × confidence` | 内在价值 |

层级特定衰减形状：

- **Core** (β=0.8): 亚指数衰减 → 衰减缓慢，地板 0.9
- **Working** (β=1.0): 标准指数衰减，地板 0.7
- **Peripheral** (β=1.3): 超指数衰减 → 衰减加速，地板 0.5

关键特性：

- `score(memory)` — 单条记忆衰减评分
- `applySearchBoost(results)` — 搜索结果衰减加权
- `getStaleMemories(memories)` — 识别过期记忆（composite < 0.3）
- 重要性调制：`effectiveHL = halfLife × exp(μ × importance)`，重要记忆半衰期更长

---

### 6. `src/tier-manager.ts`

**三层晋升/降级管理器**，移植自 memx-memory 的生命周期模型：

```
Peripheral ⟷ Working ⟷ Core
```

晋升条件：
| 方向 | 条件 |
|------|------|
| Peripheral → Working | `accessCount ≥ 3` 且 `composite ≥ 0.4` |
| Working → Core | `accessCount ≥ 10` 且 `composite ≥ 0.7` 且 `importance ≥ 0.8` |

降级条件：
| 方向 | 条件 |
|------|------|
| Working → Peripheral | `composite < 0.15` 或 `(age > 60天 且 accessCount < 3)` |
| Core → Working | `composite < 0.15` 且 `accessCount < 3`（极少触发） |

---

## 三、修改文件

### `index.ts` — 插件入口

#### 新增导入

```typescript
import { SmartExtractor } from "./src/smart-extractor.js";
import { createLlmClient } from "./src/llm-client.js";
import { createDecayEngine, DEFAULT_DECAY_CONFIG } from "./src/decay-engine.js";
import { createTierManager, DEFAULT_TIER_CONFIG } from "./src/tier-manager.js";
```

#### 新增配置字段

```typescript
interface PluginConfig {
  // ...existing fields...
  smartExtraction?: boolean; // 是否启用 LLM 智能提取 (默认 true)
  llm?: {
    apiKey?: string; // LLM API Key (默认复用 embedding.apiKey)
    model?: string; // LLM 模型 (默认 gpt-4o-mini)
    baseURL?: string; // LLM API 端点
  };
  extractMinMessages?: number; // 最少消息数 (默认 4)
  extractMaxChars?: number; // 最大字符数 (默认 8000)
}
```

#### `agent_end` 钩子变更

```diff
+ // 优先使用 SmartExtractor (LLM 6类别提取)
+ if (smartExtractor && texts.length >= minMessages) {
+   const stats = await smartExtractor.extractAndPersist(conversationText, sessionKey);
+   return; // 智能提取处理完毕
+ }
  // 降级：原有正则触发逻辑
  const toCapture = texts.filter(text => shouldCapture(text));
```

#### `before_agent_start` 钩子变更

```diff
- `- [category:scope] text (score%)`
+ `- [T][memory_category:scope] L0_abstract (score%)`
```

新增 L0 摘要显示、6 类别标签、层级标记（`[C]`ore/`[W]`orking/`[P]`eripheral）。

---

## 四、配置示例

### 最简配置（复用 embedding 的 API Key）

```json
{
  "embedding": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "text-embedding-3-small"
  },
  "smartExtraction": true
}
```

### 完整配置（独立 LLM 端点）

```json
{
  "embedding": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "text-embedding-3-small"
  },
  "smartExtraction": true,
  "llm": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "gpt-4o-mini",
    "baseURL": "https://api.openai.com/v1"
  },
  "extractMinMessages": 4,
  "extractMaxChars": 8000,
  "autoCapture": true,
  "autoRecall": true
}
```

### 禁用智能提取（回退到原有正则逻辑）

```json
{
  "smartExtraction": false
}
```

---

## 五、向后兼容性

| 方面           | 兼容方式                                      |
| -------------- | --------------------------------------------- |
| LanceDB Schema | 新字段存储在 `metadata` JSON 中，不修改表结构 |
| 记忆类别       | 6 类别自动映射到原有 5 类别                   |
| 检索           | 现有 Vector+BM25 混合检索完全保留             |
| 去重           | 新逻辑仅在 smartExtraction=true 时生效        |
| 配置           | 全部新增配置项均有默认值，零配置可用          |
| 已有数据       | 新记忆包含 L0/L1/L2 元数据，旧记忆正常读取    |

### 旧记忆升级（新增）

支持将旧格式记忆统一升级为新智能记忆格式，使所有记忆共享同一套生命周期管理系统（衰减、晋升、去重）。

**升级方式**：

```bash
# 查看有多少旧记忆需要升级
openclaw memory-pro upgrade --dry-run

# 使用 LLM 生成 L0/L1/L2（推荐）
openclaw memory-pro upgrade

# 不调用 LLM，使用简单规则生成（离线可用）
openclaw memory-pro upgrade --no-llm

# 限制批量处理数量
openclaw memory-pro upgrade --batch-size 5 --limit 50
```

**升级内容**：

| 字段              | 旧格式 | 升级后              |
| ----------------- | ------ | ------------------- |
| `memory_category` | 无     | 反向映射 5→6 类别   |
| `tier`            | 无     | `"working"`         |
| `access_count`    | 无     | `0`                 |
| `confidence`      | 无     | `0.7`               |
| `l0_abstract`     | 无     | LLM 生成 / 首句截取 |
| `l1_overview`     | 无     | LLM 生成 / 单行摘要 |
| `l2_content`      | 无     | 原始 `text` 全文    |

**新增文件**: `src/memory-upgrader.ts`

**启动检测**: 插件启动 5 秒后自动检测旧记忆数量并在日志中提示运行升级命令。

---

## 六、技术来源对照

| 功能模块   | 来源文件                                       | 本项目文件                  |
| ---------- | ---------------------------------------------- | --------------------------- |
| 6 类别分类 | `epro-memory/types.ts`                         | `src/memory-categories.ts`  |
| 提取提示   | `epro-memory/prompts.ts`                       | `src/extraction-prompts.ts` |
| 提取管线   | `epro-memory/extractor.ts` + `deduplicator.ts` | `src/smart-extractor.ts`    |
| LLM 客户端 | `epro-memory/llm.ts`                           | `src/llm-client.ts`         |
| 衰减引擎   | `memx-memory/src/memory/decay.ts`              | `src/decay-engine.ts`       |
| 三层晋升   | `memx-memory` 整体设计                         | `src/tier-manager.ts`       |
