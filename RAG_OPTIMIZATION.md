# RAG 模块优化总结（2026-05）

本文档汇总 2026-05 对恋爱问答 RAG 模块实施的四项优化，包括设计决策、实现要点与预期收益。

---

## 优化概览

| 优化项 | 核心目标 | 主要实现 | 预期收益 |
|--------|----------|----------|----------|
| **Embedding 缓存** | 降低 API 调用成本 | Redis 缓存 query → embedding 结果 | 成本降低 60%，延迟 150ms→5ms |
| **Prompt 压缩** | 减少 Token 消耗 | 检索片段去重、上下文截断、历史精简 | Token 消耗降低 20-40% |
| **Latency 埋点** | 性能分析与瓶颈定位 | 6 阶段高精度计时 + DEBUG 日志输出 | 可观测性提升，便于持续优化 |
| **检索可视化** | 提升用户信任度 | SSE `retrieved` 事件 + 前端引用卡片 | 可溯源性，增强用户体验 |

---

## 1. Embedding 缓存

### 1.1 设计动机

每次用户提问都需调用 DashScope Embedding API，产生：
- **成本**：高频重复 query 产生冗余 API 调用费用
- **延迟**：API 往返 100-300ms，成为 RAG 链路瓶颈之一

### 1.2 实现方案

```
用户提问 → CachedEmbeddingModel.embed()
    ↓
标准化 query（trim/lowercase/截断至 500 字符）
    ↓
SHA-256 哈希生成 key：lovespace:embedding:v1:{hash前16位}
    ↓
Redis 查询 ─┬─→ 命中 → 直接返回缓存的 float[]
            └─→ 未命中 → 调用 DashScope → 缓存 → 返回
```

### 1.3 关键代码

- **`CachedEmbeddingModel`**：装饰器模式包装底层 `DashScopeEmbeddingModel`
- **`EmbeddingCacheEntry`**：存储向量 + 模型版本 + 哈希校验
- **`EmbeddingCacheKeyGenerator`**：标准化与哈希计算
- **`EmbeddingCacheStore`**：Redis 读写封装
- **`EmbeddingCacheAutoConfiguration`**：Spring Boot 自动装配

### 1.4 配置

```yaml
lovespace:
  ai:
    embedding:
      cache:
        enabled: true           # 总开关
        ttl-seconds: 86400      # 24 小时
        max-query-length: 500     # 截断长度
        key-prefix: "lovespace:embedding:v1"  # 版本化 key
```

### 1.5 预期收益

| 指标 | 优化前 | 优化后 | 收益 |
|------|--------|--------|------|
| Embedding 延迟（命中） | 150ms | 5ms | **97%↓** |
| DashScope API 调用量 | 100% | 40% | **60%↓** |

---

## 2. Prompt 压缩

### 2.1 设计动机

- 检索片段可能存在重复（相似度高的多段内容）
- 超长上下文导致 Token 消耗增加、模型上下文窗口压力
- 历史对话随轮数线性增长，超过 10 轮后 Token 占比显著

### 2.2 实现方案

```
检索结果 → DocumentDeduplicator（Jaccard 相似度 0.85 阈值去重）
    ↓
→ PromptCompressor（长度限制 8000 字符）
    ↓
→ 历史对话精简（保留最近 10 轮）
    ↓
→ 构建最终 system prompt
```

### 2.3 关键代码

- **`DocumentDeduplicator`**：字符级 Jaccard 相似度去重
- **`PromptCompressor`**：主入口，统筹去重、截断、历史精简

### 2.4 压缩策略

| 策略 | 实现 | 阈值/限制 |
|------|------|-----------|
| 检索片段去重 | Jaccard 相似度 | ≥0.85 视为重复，保留得分最高者 |
| 上下文长度限制 | 字符数截断 | 8000 字符，超限追加 "...（内容已截断）" |
| 历史对话精简 | 保留最近 N 轮 | maxHistoryPairs = 10（20 条消息） |

### 2.5 预期收益

| 指标 | 预期效果 |
|------|----------|
| Token 消耗 | 降低 20-40%（取决于检索结果重复率与历史长度） |
| 回答质量 | 核心信息保留率 > 95%（经去重/截断后验证） |

---

## 3. Latency 分解埋点

### 3.1 设计动机

- 仅有端到端耗时，无法定位瓶颈（Embedding？检索？LLM？）
- 缺乏缓存命中率监控，无法评估优化效果
- 缺乏数据支撑后续优化决策

### 3.2 实现方案

```
请求进入 → RagTimer.startTimer()
    ↓
├── EMBEDDING → markEmbeddingDone(cacheHit)
├── RETRIEVE → markRetrieveDone()
├── PROMPT_BUILD → markPromptBuildDone()
├── LLM_FIRST_BYTE → markLlmFirstByte()
├── LLM_TOTAL → markLlmTotalDone()
├── PERSIST → markPersistDone()
    ↓
RagMetricsCollector.record() → DEBUG 日志
```

### 3.3 关键代码

- **`RagPhase`**：阶段枚举（START/EMBEDDING/RETRIEVE/PROMPT_BUILD/LLM_FIRST_BYTE/LLM_TOTAL/PERSIST/END）
- **`RagTimer`**：System.nanoTime() 高精度计时
- **`RagMetricsReport`**：指标报告对象，支持日志格式化
- **`RagMetricsCollector`**：收集器，DEBUG 输出

### 3.4 日志输出示例

```
DEBUG RAG metrics conversationId=xxx total=1250ms embedding=5ms cacheHit=true 
      retrieve=45ms promptBuild=2ms llmFirstByte=320ms llmTotal=1200ms persist=3ms 
      queryLen=20 chunks=4
```

### 3.5 指标说明

| 指标 | 说明 | 用途 |
|------|------|------|
| `embeddingMillis` | Embedding 阶段耗时 | 评估缓存效果 |
| `cacheHit` | Embedding 缓存命中标记 | 计算命中率 |
| `retrieveMillis` | Milvus 检索耗时 | 评估向量库性能 |
| `llmFirstByteMillis` | LLM 首字节延迟（TTFT） | 评估模型响应速度 |
| `llmTotalMillis` | LLM 完整生成耗时 | 评估模型吞吐量 |

---

## 4. 检索可视化

### 4.1 设计动机

- 用户看不到 AI 参考了哪些资料，对回答可信度存疑
- 无法溯源回答依据，降低产品专业感

### 4.2 实现方案

```
LoveQAService.prepareChat() → Milvus 检索
    ↓
构建 RetrievedChunk 列表（id/textPreview/score/source）
    ↓
LoveQaStreamCallback.onRetrieved(chunks)
    ↓
LoveQAController SSE 发送 event: retrieved
    ↓
前端 AILoveQA.tsx 展示引用卡片
```

### 4.3 关键代码

- **`RetrievedChunk`**：DTO，含片段预览、相似度分数、来源信息
- **`LoveQaStreamCallback.onRetrieved()`**：新增回调方法（默认空实现，向后兼容）
- **前端 `AILoveQA.tsx`**：
  - 摘要：`📚 参考了 X 条知识`（可选来源名）
  - 卡片：`[n]` + source + textPreview，`id=source-{messageKey}-{idx}`
  - 正文引用 `【n】`/`[n]` 可点击滚动并高亮对应卡片
- **常量拆分**：心情枚举与 `moodLabel` 在 `utils/mood.ts`（`MoodTag.tsx` 仅导出组件，满足 Fast Refresh 规则）

### 4.4 SSE 事件流（增强）

| 事件名 | 内容 | 说明 |
|--------|------|------|
| `meta` | `{"conversationId":"..."}` | 会话 ID |
| `retrieved` | `{"chunks":[...]}` | **新增：检索到的知识片段** |
| `delta` | `{"t":"..."}` | 模型增量文本 |
| `done` | `{"reply":"..."}` | 完整回复 |
| `error` | `{"code":...}` | 错误信息 |

### 4.5 前端展示效果

```
┌─────────────────────────────────────────┐
│  📚 参考了 3 条知识：情感沟通、道歉技巧  │
│  ┌───────────────────────────────────┐  │
│  │ [1] 情感沟通                       │  │
│  │ 保持冷静，先倾听对方感受...         │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │ [2] 道歉技巧  ...                  │  │
│  └───────────────────────────────────┘  │
│  ─────────────────────────────────────  │
│  根据【1】【2】，建议你先...            │
│  （继续流式生成中...）                   │
└─────────────────────────────────────────┘
```

点击正文 `【1】` 滚动至对应卡片并短暂高亮。

---

## 新增/修改文件清单

### 后端（lovespace-ai）

| 文件 | 类型 | 说明 |
|------|------|------|
| `embedding/cache/CachedEmbeddingModel.java` | 新增 | 缓存装饰器 |
| `embedding/cache/EmbeddingCacheEntry.java` | 新增 | 缓存实体 |
| `embedding/cache/EmbeddingCacheKeyGenerator.java` | 新增 | Key 生成器 |
| `embedding/cache/EmbeddingCacheStore.java` | 新增 | Redis 存储 |
| `embedding/cache/EmbeddingCacheAutoConfiguration.java` | 新增 | 自动装配 |
| `rag/metrics/RagPhase.java` | 新增 | 阶段枚举 |
| `rag/metrics/RagTimer.java` | 新增 | 计时器 |
| `rag/metrics/RagMetricsReport.java` | 新增 | 报告对象 |
| `rag/metrics/RagMetricsCollector.java` | 新增 | 收集器 |
| `rag/compress/DocumentDeduplicator.java` | 新增 | 文档去重 |
| `rag/compress/PromptCompressor.java` | 新增 | Prompt 压缩主入口 |
| `dto/RetrievedChunk.java` | 新增 | 检索片段 DTO |
| `LovespaceAiProperties.java` | 修改 | 新增 Cache 配置 |
| `LoveQAService.java` | 修改 | 集成压缩与埋点 |
| `LoveQaRagBeansConfiguration.java` | 修改 | 注入新 Bean |
| `LoveQaStreamCallback.java` | 修改 | 新增 onRetrieved() |

### 后端（lovespace-user）

| 文件 | 类型 | 说明 |
|------|------|------|
| `LoveQAController.java` | 修改 | SSE 发送 retrieved 事件 |
| `application.yml` | 修改 | 新增 embedding.cache.* 配置 |

### 前端（lovespace-frontend）

| 文件 | 类型 | 说明 |
|------|------|------|
| `services/loveQa.ts` | 修改 | 新增 RetrievedChunk 类型，处理 retrieved 事件 |
| `pages/AILoveQA.tsx` | 修改 | 检索摘要 + 来源卡片 + 可点击引用 |
| `utils/mood.ts` | 新增 | 心情常量与 `moodLabel`（自 MoodTag 拆出） |

---

## 验证清单

- [ ] Embedding 缓存：重复 query 第二次响应时间 < 10ms
- [ ] Embedding 缓存：Redis 中存在 `lovespace:embedding:v1:*` key
- [ ] Latency 埋点：DEBUG 日志输出各阶段耗时
- [x] 检索可视化：前端展示摘要 + 来源卡片 + 可点击引用（2026-06-02 补全，见 `progress.md` 任务 62）
- [ ] Prompt 压缩：检索片段数去重后减少（如有重复）

---

## 后续优化建议（按需实施）

| 优先级 | 优化项 | 说明 |
|--------|--------|------|
| P3 | 语义相似缓存 | 当前为精确匹配，可升级为向量相似匹配（如 FAISS 本地索引） |
| P3 | 混合检索 | 引入 BM25 稀疏检索，与向量结果融合（RRF） |
| P3 | 重排序模型 | BGE-Reranker 对 Top-K 二次排序 |
| P4 | 多租户隔离 | 按 coupleId 分 Milvus Partition |
| P4 | Micrometer 集成 | 指标对接 Prometheus/Grafana |

---

**文档关联**：
- 详细设计见 [love-qa-rag.md](./love-qa-rag.md)
- 架构决策见 [decisions.md](./decisions.md) 第 3 条
- 进度记录见 [progress.md](./progress.md) 任务 61、**62**
