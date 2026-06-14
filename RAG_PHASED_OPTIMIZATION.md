# RAG 分阶段优化计划

本文档基于 **2026-06 代码现状** 与架构评审结论，按固定顺序拆解恋爱问答 RAG 的后续优化：**摄入 → 召回/过滤/重排 → 缓存/隔离/限流/降级 → 观测/评测/发布**。实施时须遵守此顺序，避免在脏数据或未隔离的检索上叠加复杂能力。

**关联文档**：

| 文档 | 关系 |
|------|------|
| [love-qa-rag.md](./love-qa-rag.md) | 当前实现、API、Bean、前后端路径 |
| [RAG_OPTIMIZATION.md](./RAG_OPTIMIZATION.md) | **已落地**（2026-05）：Embedding 缓存、Prompt 压缩、Latency 埋点、检索可视化 |
| [decisions.md](./decisions.md) | 架构决策；重大表结构/业务规则变更须同步 |
| [progress.md](./progress.md) | 任务进度与对话摘要 |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | Milvus、DashScope、nginx SSE 代理 |

**核心代码路径**：

| 用途 | 路径 |
|------|------|
| RAG 主流程 | `lovespace-ai/.../rag/LoveQAService.java` |
| 分片入库 | `lovespace-ai/.../rag/DocumentIngestPipeline.java` |
| Prompt 压缩 | `lovespace-ai/.../rag/compress/PromptCompressor.java` |
| Embedding 缓存 | `lovespace-ai/.../embedding/cache/CachedEmbeddingModel.java` |
| 指标 | `lovespace-ai/.../rag/metrics/RagMetricsCollector.java` |
| HTTP/SSE | `lovespace-user/.../controller/LoveQAController.java` |
| 历史/MySQL | `lovespace-user/.../service/impl/LoveQaConversationServiceImpl.java` |
| 前端 | `lovespace-frontend/src/pages/AILoveQA.tsx`、`services/loveQa.ts` |
| 配置 | `lovespace-user/src/main/resources/application.yml` → `lovespace.ai.rag.*` |

---

## 0. 现状摘要与阶段依赖

### 0.1 已具备能力（可复用，勿重复建设）

- Milvus 向量库 + DashScope `text-embedding-v2`（1536 维，COSINE）
- Token 级分片 + 入库 SHA-256 片内去重（`DocumentIngestPipeline`）
- 三种入库：JSON 文本 / UTF-8 文件 / URL 抓取
- 向量检索 topK=4 + 可选 `coupleId` metadata filter
- Prompt 压缩（Jaccard 去重、8000 字符截断、历史最多 10 轮）
- Redis 多轮会话 + MySQL 历史持久化 + Redis 缺失时 MySQL 回填
- SSE 流式 + `retrieved` 事件 + 前端引用卡片
- Embedding 单条 query Redis 缓存（24h TTL）

### 0.2 主要短板（驱动本计划）

| 领域 | 短板 |
|------|------|
| **摄入** | 无 MySQL 文档台账；同步阻塞；URL 清洗简陋；无文档级/sourceUrl 去重；无删除/更新；`coupleId` 可选且无归属校验 |
| **召回** | 纯向量；无相似度阈值；无 rerank；去重算法简化；chat 未传 `coupleId` 时全库检索 |
| **韧性** | 无限流；无 Milvus/Embedding 故障降级；Embedding 批量路径不走缓存；`markEmbeddingDone` 未接入 |
| **观测** | 仅 DEBUG 日志；无 golden eval；无灰度开关；历史消息无 `retrievedChunks` 快照 |

### 0.3 阶段依赖关系

```
阶段 1 知识摄入（台账 + metadata + 质量）
        ↓
阶段 2 召回 / 过滤 / 重排（在干净、可隔离的数据上）
        ↓
阶段 3 缓存 / 隔离 / 限流 / 降级（在生产流量与故障下稳态）
        ↓
阶段 4 观测 / 评测 / 发布（可量化迭代与安全上线）
```

**原则**：阶段 2 的 filter/partition 依赖阶段 1 的 metadata 规范；阶段 3 的隔离策略依赖阶段 1+2；阶段 4 的 eval 依赖阶段 1 的文档可追溯性。

---

## 阶段 1：先把知识摄入链路做对

**目标**：入库内容可追溯、可管理、可复现；分片与 metadata 稳定，为召回与情侣隔离打基础。

**阶段完成定义（DoD）**：

- [x] 有文档台账，支持 list / delete / re-ingest
- [x] `coupleId` 写入与权限校验闭环（Sprint 1：`LoveQaIngestValidator`、默认关闭 GLOBAL、`allow-global-ingest` 配置、前端未绑定禁用入库）
- [x] URL/文件质量可控，文档级去重生效（Sprint 2：Jsoup + SSRF、`LoveQaIngestFileValidator`、`LoveQaDocumentDedupeService`）
- [x] 入库异步化 + 状态可查（Sprint 2：`async-ingest`、`LoveQaDocumentIngestAsyncService`、启动恢复、前端轮询）
- [x] 失败可补偿，不产生「半入库脏向量」（P1-A 已具备；异步路径复用 `LoveQaDocumentProcessor`）

---

### 1.1 当前实现与问题

| 能力 | 现状（代码） | 问题 |
|------|--------------|------|
| 分片 | `DocumentIngestPipeline`：`chunkStrategy=token`（默认），`chunkSize=800`，`chunkOverlap=100`；片级 SHA-256 去重 | 无任务状态、无失败重试 |
| 入库 API | `LoveQAController`：`/ingest`、`/ingest/file`、`/ingest/url` | 同步阻塞；大文档无进度 |
| URL 抓取 | `fetchTextFromUrl`：正则去 script/style/标签 | 正文噪声大，质量不可控 |
| 文档目录 | 仅 Milvus 向量 | **无 MySQL 台账**，无法列表/删/改 |
| metadata | `buildMetadata`：`ownerUserId`、`title`、`sourceUrl`、`category`、`coupleId` | 无 schema 校验；`coupleId` 可选 |
| 去重 | `deduplicateByContentHash`（chunk 级） | 无 **document 级 / sourceUrl 级** 去重 |
| 权限 | JWT 登录即可 | 未校验「只能为本情侣入库」 |
| Embedding | `vectorStore.add(docs)` 批量 | `CachedEmbeddingModel.call()` 批量路径直接 delegate，**不入缓存** |

**相关配置**（`application.yml`）：

```yaml
lovespace.ai.rag:
  chunk-size: 800
  chunk-overlap: 100
  chunk-strategy: token          # token | character
  deduplicate-on-ingest: true
```

---

### 1.2 任务拆解

#### P1-A：文档台账 + 向量双写（最高优先级）

**动机**：没有台账则无法 list/delete、无法 eval「库内有什么」、无法做 ingest 失败补偿。

**建议设计**：

**表 `love_qa_documents`（MySQL）**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `document_id` | VARCHAR(36) PK | UUID |
| `couple_id` | VARCHAR(64) NULL | 情侣私有；NULL 表示 GLOBAL（若保留） |
| `owner_user_id` | VARCHAR(64) | 入库用户 |
| `title` | VARCHAR(256) | 标题 |
| `source_url` | VARCHAR(1024) NULL | 来源 URL |
| `category` | VARCHAR(64) NULL | 分类 |
| `scope` | VARCHAR(16) | `COUPLE` \| `GLOBAL` |
| `content_hash` | VARCHAR(64) | 全文 SHA-256（文档级去重用） |
| `status` | VARCHAR(16) | `PENDING` \| `PROCESSING` \| `SUCCESS` \| `FAILED` |
| `chunk_count` | INT | 成功写入 Milvus 的 chunk 数 |
| `error_message` | TEXT NULL | 失败原因 |
| `created_at` / `updated_at` | DATETIME | 审计 |

**表 `love_qa_document_chunks`（可选）**：

| 字段 | 说明 |
|------|------|
| `document_id` + `chunk_index` | 映射 Milvus doc id / vector id |
| `milvus_doc_id` | 删除与补偿用 |

**流程**：

```
校验 → 写台账(PENDING) → 分片 → Embedding → Milvus add → 更新台账(SUCCESS)
                              ↓ 失败
                         台账(FAILED) + 按 documentId 补偿删除已写入向量
```

**新增 API（建议）**：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/ai/love-qa/documents` | 分页列表（按 coupleId / owner） |
| GET | `/api/v1/ai/love-qa/documents/{id}` | 详情 + chunk 数 |
| DELETE | `/api/v1/ai/love-qa/documents/{id}` | 删 Milvus + 台账 |
| POST | `/api/v1/ai/love-qa/documents/{id}/reingest` | 强制重入库 |

**后端分层**：`LoveQaDocumentController` → `LoveQaDocumentService` → `DocumentIngestPipeline` + Milvus 删除封装。

**前端**：`AILoveQA.tsx` 增加「知识库管理」Tab 或侧栏入口，展示列表与删除。

**验收**：

- 入库后可查 documentId、chunkCount
- 删除后同 question 不再召回该文档片段
- 失败状态可重试

---

#### P1-B：metadata 与 coupleId 策略

**动机**：检索 filter `coupleId == 'xxx'` 仅在 chat 传 coupleId 时生效；入库若不强制 coupleId，隔离无从谈起。

**建议规则**：

| 类型 | `scope` | `coupleId` | 写入权限 |
|------|---------|------------|----------|
| 情侣私有知识 | `COUPLE` | 必填，且等于 `GET /couple/info` 的 `bindingId` | 已绑定情侣的成员 |
| 平台公共（可选） | `GLOBAL` | NULL | 仅管理员或关闭此路径 |

**Milvus metadata 强制字段**（入库时写入每个 chunk）：

- `documentId`、`chunkIndex`、`totalChunks`
- `coupleId`（COUPLE  scope）或 `scope=GLOBAL`
- `ownerUserId`、`title`、`sourceUrl`、`category`、`ingestedAt`

**服务端校验**（`LoveQAController.buildMetadata` 或独立 `IngestValidator`）：

- 已登录用户请求带 `coupleId` 时，必须与当前用户情侣 binding 一致
- 未绑定情侣：拒绝 COUPLE 入库，或仅允许只读 chat

**前端**：

- `AILoveQA.tsx` 已传 `coupleId`（来自 coupleStore）；未绑定时禁用入库并提示

**验收**：

- 无法用他人 coupleId 入库
- Milvus filter 表达式可稳定使用 `coupleId` / `scope`

---

#### P1-C：URL / 文件摄入质量

**动机**：当前 URL 入库质量差会直接污染召回与 LLM 上下文。

**URL**：

- 引入 Jsoup 或 readability 提取正文
- 限制：响应大小、Content-Type、超时、禁止内网 IP（SSRF 防护）
- 可选：同 `sourceUrl` 已存在 → 409 + 提示更新

**文件**：

- 白名单扩展名：`.txt`、`.md`
- 单文件大小上限（如 5MB，与 Tomcat/代理一致）
- 编码检测（UTF-8；非 UTF-8 拒绝或转码）

**文档级去重**：

- 入库前查台账：`content_hash` 或 `source_url` + `couple_id` 唯一
- 策略：`REJECT` | `UPDATE`（删旧向量再入新）可配置

**验收**：

- 典型公众号/博客 URL 正文可读性明显提升
- 同一 URL 重复提交行为符合配置

---

#### P1-D：异步入库与任务状态

**动机**：大文档同步调用易触发 60s/120s 超时；Embedding 批量耗时不可控。

**建议**：

- `POST /ingest` 立即返回 `{ jobId / documentId, status: PENDING }`
- 后台 `@Async` 或 Redis 队列消费：分片 → embed → Milvus
- `GET /ingest/jobs/{id}` 或 document status 轮询

**前端**：

- 提交后展示「处理中」，完成后 toast + 刷新文档列表

**验收**：

- 10MB 级 markdown 不阻塞 HTTP
- 服务重启后 PENDING/PROCESSING 任务可恢复或标记 FAILED（视实现深度）

---

#### P1-E：分片策略与入库可观测

**建议**：

- `chunkStrategy` / `chunkSize` / `chunkOverlap` 保持 `RagAiProperties` 可配置
- 入库日志：`documentId`、`strategy`、`chunkCount`、`elapsedMs`
- 远期：按 `category` 差异化分片参数（配置 map）

**验收**：

- 同一文档 reingest 的 chunkCount 在相同配置下稳定

---

#### P1-F：Embedding 入库路径（为阶段 3 铺垫，逻辑归属摄入）

**建议**：

- `CachedEmbeddingModel.call(EmbeddingRequest)`：逐条 cache-aside 或分 batch 写缓存
- 入库失败时按 `documentId` filter 删除 Milvus 已写入部分（补偿事务）

**验收**：

- 重复 ingest 相同 chunk 文本时 Embedding API 调用减少（DEBUG/指标可证）

---

### 1.3 阶段 1 风险

| 风险 | 说明 |
|------|------|
| 无台账做 Partition/批量删 | Milvus 与业务状态难以一致 |
| 过早优化分片算法 | 应先保证台账与 metadata，再调 chunk 参数 |
| GLOBAL 知识范围不清 | 产品需先定是否允许「公共恋爱百科」 |

---

### 1.4 阶段 1 建议迭代（Sprint 1–2）

| Sprint | 交付 |
|--------|------|
| **Sprint 1** | MySQL 台账表 + list/delete API + coupleId 服务端校验 |
| **Sprint 2** | 异步 ingest + URL/文件质量 + 文档级去重 + 前端知识库列表 |

---

## 阶段 2：再把召回、过滤、重排做稳

**目标**：检索结果相关、干净、可解释；减少 Prompt 噪声与幻觉输入。

**阶段 DoD**：

- [x] 过滤规则文档化且集成测试覆盖情侣隔离（P2-A：`RagRetrievalFilterBuilder` + `LoveQaChatRetrievalValidator` + 单元测试）
- [x] candidate → rerank → final topK 流水线可配置（P2-B：`RagRetrievalPipeline` + L1 rerank）
- [x] 低分片段被阈值拦截（P2-A：`similarity-threshold` + `RagSimilarityFilter`）
- [x] 检索结果与前端引用、Prompt 来源列表一致（含历史 `retrieved_chunks_json` 快照）
- [ ] 具备 offline baseline（至少 50–100 条 golden）

---

### 2.1 当前实现与问题

| 能力 | 现状 | 问题 |
|------|------|------|
| 召回 | `LoveQAService.prepareChat` → `vectorStore.similaritySearch`，`retrieve-top-k: 4` | 纯向量；关键词弱 |
| 过滤 | `coupleId == '{id}'` 仅当 chat body 含 coupleId | 未绑定时全库；无 scope/category |
| 分数 | `RetrievedChunk.fromDocument`：`distance` → similarity | **无最低阈值**，低质量仍进 Prompt |
| 去重 | `DocumentDeduplicator`（Prompt 阶段） | 简化 Jaccard；未按 documentId 去重 |
| 重排 | 无 | 代码注释预留 Hybrid/Reranker |
| Query | 用户原问句直接 embed | 无 rewrite / 多 query |

**System Prompt**（已较好，保持）：

- 强制 `【1】【2】` 引用
- 不足时拒答模板
- `PromptCompressor.buildSystemPrompt` 追加【来源列表】

---

### 2.2 任务拆解

#### P2-A：检索过滤策略固化

**状态**：✅ 已落地（2026-06-12）

**建议 filter 表达式**（Spring AI Expression Language）：

```
# 已绑定情侣
(coupleId == '{bindingId}' || scope == 'GLOBAL')

# 仅 GLOBAL（未绑定情侣且产品允许）
scope == 'GLOBAL'
```

**Chat 侧规则**：

- 已绑定：body **必须**带 `coupleId`（后端可默认从 session 解析，但 filter 必须生效）
- 未绑定：拒绝 COUPLE 知识检索，或仅 GLOBAL + 明确 UI 提示

**配置项**（扩展 `RagAiProperties`）：

```yaml
lovespace.ai.rag:
  retrieve-top-k: 4              # 最终进 LLM
  retrieve-candidate-k: 16       # 初召回（新增）
  similarity-threshold: 0.55     # 低于则丢弃（新增）
  allow-global-fallback: true    # COUPLE 检索不足是否补 GLOBAL
```

**0 命中行为**：

- 不调用 LLM 编造；返回拒答模板或 SSE 提示「知识库无相关内容」

**验收**：

- 集成测试：情侣 A 文档不被情侣 B 召回
- 阈值以下 chunk 不出现在 `retrieved` SSE 与 Prompt

---

#### P2-B：两阶段召回（Recall → Rerank）

**状态**：✅ 已落地（2026-06-12）

**流水线**：

```
query embed
  → Milvus similaritySearch(topK = candidate-k)
  → 阈值过滤
  → 去重（documentId 保留最高分 chunk；或 n-gram Jaccard）
  → Rerank（见下）
  → 取 top-k 进 Prompt + retrieved SSE
```

**Rerank 选项**（按成本递增）：

| 方案 | 说明 |
|------|------|
| **L1 规则** | Milvus score + category 匹配加权 + 新近度 |
| **L2 交叉编码** | DashScope / BGE reranker API，对 candidate 重排 |
| **L3 自研** | 小模型 local rerank（远期） |

**建议**：先 **L1 + documentId 去重**，有 eval 数据后再上 L2。

**验收**：

- 重复片段率下降（对比阶段 1 结束时的 baseline）
- 人工标注 50 条，「有用片段率」或 NDCG@4 可对比

---

#### P2-C：混合检索（中期）

**状态**：✅ 已落地（2026-06-12）

**动机**：恋爱 FAQ、专有名词（「冷战」「道歉信」）纯向量易漏召。

**建议**：

- MySQL 全文索引（`title` + 摘要）或 Elasticsearch BM25
- 向量 + BM25 **RRF（Reciprocal Rank Fusion）** 融合 → 再 rerank

**依赖**：阶段 1 台账中的 `title`、摘要字段。

**验收**：

- 含明确关键词的问题，召回率高于纯向量 baseline（eval 集可证）

---

#### P2-D：Query 增强（可选，过滤/重排稳定后）

| 手段 | 适用 | 成本 |
|------|------|------|
| 多轮 query rewrite | 指代消解（「他」「那次」） | 一次短 LLM 调用 |
| HyDE | 语义扩召 | 高 |
| 多 query 并行检索 | 提升 recall | Embedding × N |

**建议**：先做 **rewrite**（仅多轮且 history 非空时），HyDE 列入 P4 eval 对比。

---

#### P2-E：Prompt / 前端 / 历史一致

**状态**：✅ 已落地（2026-06-12，随 Sprint 4 一并完成）

**问题**：MySQL 历史仅存 assistant 文本，重开对话无 `retrievedChunks`。

**建议**：

- `love_qa_messages` 增加 `retrieved_chunks_json`（可选列）
- 流式 `done` 前后持久化本轮 chunks 快照
- `getLoveQaMessages` 返回后前端还原引用卡片

**验收**：

- 切换历史会话后仍可见「参考了 N 条知识」（与当时一致）

---

### 2.3 阶段 2 风险

| 风险 | 说明 |
|------|------|
| 过早上 hybrid/rerank | 掩盖脏数据；应阶段 1 完成后再开 |
| 阈值过高 | 拒答率飙升；需 eval 调参 |
| filter 与入库 metadata 不一致 | 导致「库里有但检不到」 |

---

### 2.4 阶段 2 建议迭代（Sprint 3–4）

| Sprint | 交付 |
|--------|------|
| **Sprint 3** | 强制 filter + similarity 阈值 + candidate-k + 0 命中拒答 |
| **Sprint 4** | documentId 去重 + rerank L1/L2 + 历史 retrieved 快照 |

---

## 阶段 3：再补缓存、隔离、限流和降级

**目标**：生产环境抗流量、抗故障、成本可控。

**阶段 DoD**：

- [ ] Embedding 缓存命中率可观测（重复 query 场景）
- [ ] ingest/chat 限流生效
- [ ] Milvus / LLM / Redis 故障有明确降级行为
- [ ] couple 隔离从入库到检索全链路一致

---

### 3.1 当前实现与问题

| 能力 | 现状 | 问题 |
|------|------|------|
| Embedding 缓存 | `CachedEmbeddingModel.embed()` 单条 | `call()` 批量 bypass；`RagTimer.markEmbeddingDone` 未调用 |
| 会话缓存 | Redis `lovespace:love-qa:conv:*`，TTL 7 天 | 合理 |
| 隔离 | 检索 filter 部分；入库未强制 | 见阶段 1/2 |
| 限流 | 无 | ingest/chat 可被刷 |
| 降级 | 异常 → SSE `error` | 无 Milvus 不可用时的分级 fallback |
| Provider 切换 | `LlmRouter` qwen/openai | 未与 RAG 降级策略联动 |

---

### 3.2 任务拆解

#### P3-A：完善 Embedding 缓存

**改动点**：

- `CachedEmbeddingModel.call(EmbeddingRequest)`：逐条 get/set Redis
- 在检索路径记录 `cacheHit` 并调用 `timer.markEmbeddingDone(cacheHit)`
- 缓存 key 已含 model 校验（`EmbeddingCacheEntry.validate`），保持

**可选（P4）**：semantic cache（query 向量近邻），需 eval 证明收益

**验收**：

- DEBUG/metrics 可见 `cacheHit=true/false`
- 重复 question Embedding 延迟显著下降

---

#### P3-B：隔离策略端到端

| 环节 | 动作 |
|------|------|
| 入库 | P1-B 校验 |
| 检索 | P2-A filter |
| Milvus | 可选 Partition：`couple_{id}` + `global` |
| 解除情侣 | `/couple/separate` 后策略：归档 / 软删 / 保留只读（产品定案后实现） |

**验收**：

- 安全测试：跨 couple 读写均拒绝

---

#### P3-C：限流与配额

**Redis 滑动窗口示例**：

| 维度 | 建议限额 |
|------|----------|
| ingest / 用户 / 日 | 20 次 |
| ingest / couple / 日 | 50 次 |
| chat/stream / 用户 / 分钟 | 10 次 |
| 并发 SSE / 用户 | 1–2 |

**业务码**：`42991`（RAG 限流），与 `ApiBusinessException` 体系一致。

**前端**：收到 429 友好提示。

---

#### P3-D：降级矩阵

| 故障 | 策略 |
|------|------|
| Milvus 不可用 | `degrade-mode=STRICT`：拒答 + 说明；`RELAXED`：无 RAG 的纯 LLM（须 UI 标明） |
| Embedding 超时 | 重试 1 次 → 失败则拒答 |
| LLM 流中断 | 返回 partial + 可重试 |
| Redis 不可用 | 单轮可用；无多轮记忆 |
| DashScope 429 | 排队或 `LlmRouter` 切 openai |

**实现建议**：

- `RagHealthGate`：启动/定时检查 Milvus、Redis
- `LoveQAService.prepareChat` 入口判断健康状态
- 配置：`lovespace.ai.rag.degrade-mode: STRICT | RELAXED`

---

#### P3-E：成本与参数外置

- `PromptCompressor` 的 8000 字符、10 轮历史 → 迁入 `RagAiProperties`
- ingest 与 chat Embedding 分离监控配额
- 大文档 ingest 可夜间批处理（可选）

---

### 3.3 阶段 3 建议迭代（Sprint 5）

| Sprint | 交付 |
|--------|------|
| **Sprint 5** | 缓存补全 + 限流 + 降级矩阵 + health gate |

---

## 阶段 4：最后建设观测、评测和发布体系

**目标**：可持续迭代——知道好不好、慢在哪、能否安全上线。

**阶段 DoD**：

- [ ] Grafana（或 Actuator）看板覆盖 ingest/retrieve/llm/cache
- [ ] Golden eval 自动化，有 baseline 对比
- [ ] 重大 RAG 参数可灰度 + 回滚
- [ ] 线上问题可凭 traceId 定位到阶段

---

### 4.1 当前实现与问题

| 能力 | 现状 | 问题 |
|------|------|------|
| 指标 | `RagMetricsCollector` → DEBUG | 无 Prometheus；EMBEDDING 阶段常为空 |
| 追踪 | `conversationId` 日志 | 无 ingest→retrieve→LLM 统一 traceId |
| 评测 | `RAG_OPTIMIZATION.md` 验证清单多未勾 | 无 golden set |
| 发布 | 无 feature flag | 无法灰度 |
| 前端 |  live 流式有引用 | 历史无 chunks |

---

### 4.2 任务拆解

#### P4-A：可观测性（Micrometer + 结构化日志）

**建议指标**：

```
rag.ingest.duration / success / chunk_count
rag.retrieve.duration / candidate_count / final_count
rag.embedding.duration + tag cache=hit|miss
rag.llm.ttft / total
rag.chat.error{reason=milvus|embedding|llm|timeout}
```

**日志字段**：`traceId`、`conversationId`、`coupleId`、`documentId`

**落地**：Spring Boot Actuator + Micrometer Prometheus；或先增强 DEBUG 日志格式统一。

---

#### P4-B：离线评测集 + CI 门禁

**目录建议**：`lovespace-backend/lovespace-ai/src/test/resources/eval/love-qa-golden.jsonl`

**单条样本**：

```json
{
  "id": "q001",
  "question": "和对象吵架了怎么办？",
  "coupleId": "test-couple-1",
  "expectedSources": ["冲突解决"],
  "mustCite": true,
  "mustRefuse": false
}
```

**检查项**：

- 回复含 `【n】` 引用（`mustCite`）
- 检索 chunks 含期望 source/category
- 拒答类 `mustRefuse=true` 时不编造

**CI**：PR smoke 10 条；nightly 全量 100+ 条。

---

#### P4-C：在线质量采样

- 随机 1% 对话异步写入 `love_qa_eval_samples`
- 管理端标注：有用 / 无用 / 幻觉
- 反馈驱动补充 ingest（闭环回阶段 1）

---

#### P4-D：发布与配置管理

**Feature flags**（`lovespace.ai.rag` 或独立配置）：

```yaml
lovespace.ai.rag:
  features:
    hybrid-retrieval: false
    rerank-enabled: false
    strict-couple-filter: true
```

**流程**：

- 参数变更写入 `decisions.md`
- 生产先小流量 couple 白名单验证

---

#### P4-E：前端可观测与体验

- 入库成功：展示 `documentId`、`chunkCount`
- 历史会话：加载 `retrievedChunks`（依赖 P2-E）
- 开发环境可选 debug panel（score、latency）

---

### 4.3 阶段 4 建议迭代（Sprint 6+）

| Sprint | 交付 |
|--------|------|
| **Sprint 6** | Micrometer 指标 + traceId + golden eval 脚本 |
| **Sprint 7** | Feature flag + 在线采样 + Grafana 看板 |

---

## 5. 总路线图

```
Sprint 1  ─ 台账 + coupleId 校验 + list/delete
Sprint 2  ─ 异步 ingest + URL/文件质量 + 文档去重 + 前端列表
Sprint 3  ─ filter 固化 + 阈值 + candidate-k + 0 命中拒答
Sprint 4  ─ 去重升级 + rerank + 历史 chunks 快照
Sprint 5  ─ 缓存补全 + 限流 + 降级 + health gate
Sprint 6  ─ 指标 + golden eval + traceId
Sprint 7  ─ 灰度 + 在线采样 + 看板
```

---

## 6. 资源受限时的「最小路径」

每阶段只做一条主线仍可显著降风险：

| 阶段 | 最小交付 |
|------|----------|
| **1** | MySQL 台账 + coupleId 服务端校验 |
| **2** | 相似度阈值 + candidate 扩大 + 同 documentId 去重 |
| **3** | chat/ingest 限流 + Milvus 不可用 STRICT 拒答 |
| **4** | 补 Embedding 埋点 + 20 条 golden eval 脚本 |

---

## 7. 与已落地优化的关系

[RAG_OPTIMIZATION.md](./RAG_OPTIMIZATION.md) 中四项（2026-05）**保留复用**，本计划在其之上补齐：

| 已落地 | 本计划增强 |
|--------|------------|
| Embedding 缓存（单条） | P3-A 批量路径 + 命中率指标 |
| Prompt 压缩 | P2-B 检索前去重升级；参数外置 P3-E |
| Latency 埋点 | P4-A Micrometer；P3-A 补 EMBEDDING 阶段 |
| 检索可视化 | P2-E 历史快照；P4-E 入库反馈 |

---

## 8. 文档维护约定

- 每完成一个 Sprint，更新 [progress.md](./progress.md) 任务摘要
- 表结构、filter 规则、降级策略变更同步 [decisions.md](./decisions.md)
- API 变更同步 [love-qa-rag.md](./love-qa-rag.md) 与 [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)
- 本文件版本：在文首记录 `最后更新：YYYY-MM-DD` 与已完成 Sprint 勾选

---

## 9. 会话接续 checklist

新开 Cursor 窗口继续 RAG 优化时，按本节操作即可恢复上下文，**不必**重读整个 `memory/` 或全仓代码。

### 9.1 开场：文档加载顺序

| 顺序 | 文档 | 用途 |
|------|------|------|
| ① | [INDEX.md](./INDEX.md) | 确认路由表 |
| ② | **本文件** `RAG_PHASED_OPTIMIZATION.md` | 阶段计划、Sprint、DoD、任务拆解 |
| ③ | [progress.md](./progress.md) | 「对话任务摘要」：上次进度与阻塞 |
| ④ | [love-qa-rag.md](./love-qa-rag.md) | 当前实现、API、Bean（**改代码时**再读） |
| ⑤ | [RAG_OPTIMIZATION.md](./RAG_OPTIMIZATION.md) | 仅当改动 2026-05 已落地项（缓存/压缩/埋点/可视化） |
| ⑥ | [decisions.md](./decisions.md) | 改表、filter、隔离策略时 |
| ⑦ | [DEPLOYMENT.md](./DEPLOYMENT.md) | 联调 Milvus / DashScope / nginx SSE 时 |

**原则**：计划看本文件，现状看 `love-qa-rag.md`，进度看 `progress.md`。

### 9.2 开场：可复制提示词

将下面模板粘贴为新会话**第一条消息**，并替换 `[...]`：

```text
你是 LoveSpace 全栈开发助手，继续 RAG 优化工作。

请先读：
1. memory/INDEX.md（路由）
2. memory/RAG_PHASED_OPTIMIZATION.md（分阶段计划）
3. memory/progress.md（最近任务摘要）

当前进度：
- 阶段：[1 摄入 / 2 召回 / 3 韧性 / 4 观测]
- Sprint：[如 Sprint 1]
- 任务：[如 P1-A 文档台账]
- 上次状态：[未开始 / 进行中 / 已完成 / blocked 及原因]

本次目标：[具体要完成的一件事]

约束：
- 遵守 .cursor/rules/lovespace-coding-standards.mdc
- 按 RAG_PHASED_OPTIMIZATION 阶段顺序，不跳阶段
- 重大变更同步 memory/decisions.md 与 progress.md
- 未明确要求不要 git commit

读完上述文档后，先简要复述当前阶段 DoD 与本次任务，再开始实现。
```

### 9.3 按任务类型 @ 代码文件

| 本次要做的事 | 建议 @ 或让 Agent 打开 |
|--------------|-------------------------|
| 改入库 | `LoveQAController.java`、`DocumentIngestPipeline.java` |
| 改检索 / Prompt | `LoveQAService.java`、`PromptCompressor.java` |
| 改缓存 / 指标 | `CachedEmbeddingModel.java`、`RagMetricsCollector.java` |
| 改前端问答页 | `lovespace-frontend/src/pages/AILoveQA.tsx`、`services/loveQa.ts` |
| 改配置 | `lovespace-user/src/main/resources/application.yml` |
| 新建 SQL | `lovespace-user/src/main/resources/sql/` + `memory/blockers.md` |

### 9.4 收工：写回 memory（供下次接续）

**A. 更新 [progress.md](./progress.md)** — 在「对话任务摘要」追加或修订：

```markdown
### RAG 分阶段优化（YYYY-MM-DD 起）
- **计划文档**：memory/RAG_PHASED_OPTIMIZATION.md
- **当前**：阶段 N / Sprint M / 任务 Px-y
- **已完成**：…
- **进行中**：…
- **阻塞**：…
- **下次**：…
```

**B. 更新本文件** — 勾选对应阶段 DoD / Sprint 项；更新下方「最后更新」日期。

**C. 若变更表结构、filter、降级策略** — 同步 [decisions.md](./decisions.md)；API 变更同步 [love-qa-rag.md](./love-qa-rag.md)。

### 9.5 默认起点（尚未开工时）

若 `progress.md` 尚无 RAG 分阶段条目，默认从：

- **阶段 1 → Sprint 1 → P1-A**：MySQL 文档台账 + `coupleId` 服务端校验 + list/delete API

### 9.6 不必重复做的事

- 不必让 Agent 一次性加载全部 `memory/` 文档
- 不必重讲 Milvus + DashScope + SSE 基础（见 `love-qa-rag.md`）
- 不必重审 2026-05 四项已落地优化，除非修改对应实现文件

---

**最后更新：2026-06-12**（P2-C 混合检索：MySQL FULLTEXT + RRF 融合）
