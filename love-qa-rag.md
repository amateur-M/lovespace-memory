# 恋爱知识库（Love QA）与 RAG / Milvus

本文汇总「AI 恋爱问答 / RAG / Milvus」的前后端设计、Bean 装配、集合初始化与部署要点（与当前代码一致）。

---

## 1. 目标与分层

- **`lovespace-ai`**：LLM 抽象（通义千问 / OpenAI）、`LoveQaChatFacade` 与 DTO/异常；**`DashScopeEmbeddingConfiguration`** 在 `lovespace.ai.embedding.provider=dashscope`（默认）且无其它 `EmbeddingModel` Bean 时注册 **`DashScopeEmbeddingModel`**，与聊天共用 **`spring.ai.dashscope.api-key`**。同模块还包含 Spring AI **`VectorStore`（Milvus）**、`LoveQAService`（实现 `LoveQaChatFacade`）、Redis 多轮会话、`MilvusSchemaService`、`RagMilvusConfiguration` 等 RAG 实现。
- **`lovespace-user`**：**`LoveQAController`** 为 **`@RestController`**，由 **`com.meng.lovespace.user`** 包扫描注册（不再使用单独的 `LoveQAControllerConfiguration`）；**`LoveQAHistoryController`** 提供 MySQL 历史分页。

---

## 2. 模块与 Maven

| 模块 | 作用 |
|------|------|
| `lovespace-ai` | `LLMProvider`、`LlmRouter`、`LoveQaChatFacade`、嵌入、旅游规划；以及 `spring-ai-starter-vector-store-milvus`、`LoveQAService`、`DocumentIngestPipeline`、`MilvusSchemaService`、`RagMilvusConfiguration` 等 RAG |
| `lovespace-user` | 默认可执行 JAR **依赖** `lovespace-ai`（含 RAG）；**无需** Maven Profile `-Plovespace-rag` |

打包示例：

```bash
mvn -pl lovespace-user -am package -DskipTests
```

---

## 3. 配置要点

- **`lovespace.ai.rag.*`**：`RagAiProperties` — 分片、`retrieve-top-k`、Redis 会话 TTL、`max-history-pairs` 等（**已无**业务总开关 `enabled`）。
- **`spring.ai.vectorstore.milvus.*`**：客户端、**`collection-name`**、**`database-name`**、**`embedding-dimension`**、**`initialize-schema`**、索引类型/度量等（与 [Spring AI Milvus 文档](https://docs.spring.io/spring-ai/reference/api/vectordbs/milvus.html) 一致）。
- **`lovespace.milvus.*`**：`MilvusProperties` — **`ensure-love-knowledge-schema`**（默认 `true`）：启动时若集合不存在，由 **`MilvusSchemaService`** 按 Spring AI `MilvusVectorStore` 同款字段建表并建索引、`loadCollection`；**`ensure-travel-poi-schema`** 仍为旅游 POI 骨架开关。
- **`lovespace.ai.embedding.cache.*`**：Embedding 缓存配置 — `enabled`（默认 `true`）、`ttl-seconds`（默认 86400，24 小时）、`max-query-length`（默认 500）、`key-prefix`（默认 `lovespace:embedding:v1`）。用于降低 Embedding API 调用成本与延迟。

---

## 4. Bean 装配（摘要）

- **`LoveQAService` / `LoveQAConversationStore` / `MilvusSchemaService`**：在 **`LoveQaRagBeansConfiguration`** 中声明为 `@Bean`（**不再**使用 `@ConditionalOnBean`）；**`loveQAService`** 使用 **`@DependsOn("milvusSchemaService")`**，保证 **`MilvusSchemaService`** 的 `@PostConstruct` 先完成集合就绪逻辑。
- **`RagMilvusConfiguration`**：注册业务封装 **`lovespaceMilvusClient`**（类型 `com.meng.lovespace.ai.milvus.MilvusClient`），**避免**与 Spring AI 自动配置中名为 **`milvusClient`** 的 `MilvusServiceClient` Bean **重名**（勿改回 `milvusClient`，除非开启 `spring.main.allow-bean-definition-overriding`）。
- **`CachedEmbeddingModel`**：`EmbeddingCacheAutoConfiguration` 自动装配，包装底层 `DashScopeEmbeddingModel`，对 embed() 结果进行 Redis 缓存；key 为 query 标准化后的 SHA-256 前缀，value 为 `EmbeddingCacheEntry`（含向量、模型版本、创建时间）。
- **`PromptCompressor`**：`LoveQaRagBeansConfiguration` 注册为 Spring Bean，负责检索片段去重、上下文长度限制、历史对话精简。
- **`RagMetricsCollector`**：RAG 流程指标收集器，记录各阶段 Latency（Embedding/检索/LLM 首字节/总耗时）。
- **已移除**：`lovespace.ai.rag.enabled`、`RagMilvusAutoConfigurationExclusionEnvironmentPostProcessor`（不再按开关排除 Milvus 自动配置）。

---

## 5. API（用户服务）

- **`POST /api/v1/ai/love-qa/ingest`**：知识库入库（需登录；依赖 Milvus + `EmbeddingModel` + Redis）。
- **`POST /api/v1/ai/love-qa/chat`**：多轮问答，**非流式**，`ApiResponse<LoveQaChatResponseData>`（前端 `http` timeout ≥ 120s 仍可用作兼容）。
- **`POST /api/v1/ai/love-qa/chat/stream`**：**流式 SSE**，`Content-Type: text/event-stream`；`LoveQAController` 在**虚拟线程**中调用 **`LoveQaChatFacade.chatStream`**；SSE **事件名**与 JSON **字符串**载荷：
  - **`meta`**：`{"conversationId":"..."}`（首轮即下发，新会话在此拿到 UUID）
  - **`retrieved`**：`{"chunks":[{"id":"...","textPreview":"...","score":0.95,"source":"..."}]}`（检索到的知识片段，用于前端可视化引用来源）
  - **`delta`**：`{"t":"..."}`（模型**增量**文本；通义侧 `GenerationParam.incrementalOutput(true)` + `streamCall`）
  - **`done`**：`{"reply":"...","conversationId":"..."}`（与 Redis 已写入的整轮一致）
  - **`error`**：`{"code":...,"message":"..."}`（如 40491 会话不存在、40391 越权等）
  流正常结束后由 Controller 调用 **`LoveQaConversationService.appendChatRound`** 写 **MySQL**（与 `/chat` 一致）。**超时**：`SseEmitter` 120s。
- **`GET /api/v1/ai/love-qa/conversations`**、**`GET .../conversations/{id}/messages`**：历史（MySQL）。

**会话 Redis 回填**：在 **`/chat`** 与 **`/chat/stream`** 中，若请求带 `conversationId` 且 **`LoveQAConversationStore`** 中无键，则先调用 **`LoveQaConversationService.restoreRedisSessionIfMissing`**，从 **`love_qa_messages`** 恢复最近若干轮（受 **`lovespace.ai.rag.max-history-pairs`** 限制）再调 RAG，避免「历史会话点开后续聊无记忆」。

若 Milvus 或 DashScope Key 未就绪，应用**启动阶段**可能失败（不再通过条件注解静默跳过 RAG Bean）。

**反向代理**：若经 nginx 等代理 SSE，需关闭对响应体的过度缓冲（如 `proxy_buffering off`），否则增量可能攒包后才到浏览器。

---

## 5.1 多轮记忆与 LLM 调用（实现要点）

- **`LoveQAService`**：`prepareChat`（解析/创建会话、Milvus 检索、**Prompt 压缩**、拼 system 与 `priorForLlm`）与 **`persistRound`**（写入 user+assistant、`trimHistory`、`conversationStore.save`）被 **`chat`** 与 **`chatStream`** 共用。
- **`LoveQaChatFacade`**：除 **`chat`** 外增加 **`chatStream(LoveQaChatParams, LoveQaStreamCallback)`**；回调顺序 **`onMeta` → `onRetrieved` → 多次 `onDelta` → `onCompleted`**（`onCompleted` 在 Redis 持久化完成之后触发，供 Controller 发 `done` 与落 MySQL）。
- **`LLMProvider`**：**`chatWithSystemAndHistory`**（整段返回）与 **`chatWithSystemAndHistoryStreaming`**（`Consumer<String> onDelta`）。**`QwenProvider`**：`buildChatMessages` + **`Generation.streamCall`** + **`incrementalOutput(true)`**。**`OpenAiProvider`**：`ChatClient.prompt(Prompt).stream().content()`（`Flux`）。默认实现将整段一次性 `onDelta`。

### 5.1.1 Prompt 压缩

- **检索片段去重**：`DocumentDeduplicator` 基于字符级 Jaccard 相似度去重（阈值 0.85），保留高相似度片段中得分最高的一个。
- **上下文长度限制**：`PromptCompressor` 限制总上下文长度 8000 字符，超限时截断并提示「内容已截断」。
- **历史对话精简**：超长历史（>10 轮）仅保留最近 10 轮，降低 Token 消耗 20-40%。

### 5.1.2 Latency 分解埋点

- **`RagMetricsCollector`** + **`RagTimer`**：高精度计时（System.nanoTime），记录各阶段耗时：
  - `EMBEDDING`：query → embedding vector（含缓存命中标记）
  - `RETRIEVE`：Milvus similaritySearch
  - `PROMPT_BUILD`：Prompt 组装（含压缩）
  - `LLM_FIRST_BYTE`：发送请求到收到第一个 delta
  - `LLM_TOTAL`：发送请求到流结束
  - `PERSIST`：写入 Redis/MySQL
- **日志输出**：DEBUG 级别，`RAG metrics conversationId=xxx total=1250ms embedding=5ms cacheHit=true retrieve=45ms llmFirstByte=320ms llmTotal=1200ms chunks=4`

---

## 6. 前端

- 路由 **`/love-qa`**；页面 **`lovespace-frontend/src/pages/AILoveQA.tsx`**；API **`services/loveQa.ts`**。
- **主路径**：**`postLoveQaChatStream`**（`fetch` + `ReadableStream`，解析 **`meta` / `retrieved` / `delta` / `done` / `error`**；`AbortController` 120s；`VITE_SESSION_DISTRIBUTED` 时 `credentials: 'include'` 与 axios 一致）。**`postLoveQaChat`** 仍保留为非流式兼容。
- **布局（千问式 + 全站玫瑰色）**：大屏 **侧栏**（新建对话、最近会话列表、刷新）+ **主区**；空状态居中问候 + 大圆角输入区；有消息时 **消息区可滚动**、底部输入区 **`shrink-0`**。根容器 **`h-[calc(100dvh-11rem)]`** 等，避免长对话撑开整页；**`useLayoutEffect`** 在**切换会话**时瞬间滚到底、同会话内新消息平滑滚动。
- **检索可视化**：`AILoveQA.tsx` 在 assistant 消息内展示：
  - 摘要行 `📚 参考了 X 条知识`（可选列出前 3 个 source）
  - **来源卡片列表**：每条含 `[n]`、source、textPreview；DOM `id="source-{messageKey}-{idx}"`（同页多消息不冲突）
  - 正文中的 `【n】` / `[n]` 可点击，滚动至对应卡片并 **短暂高亮**（rose 边框/背景）
  - SSE `onRetrieved` 写入 `UiMessage.retrievedChunks`；`ingestMode` Tab 切换用类型守卫（无 `any`）

---

## 7. 相关路径速查

| 用途 | 路径 |
|------|------|
| User 依赖与打包 | `lovespace-backend/lovespace-user/pom.xml` |
| 应用与 Milvus 片段 | `lovespace-backend/lovespace-user/src/main/resources/application.yml` |
| RAG 实现 | `lovespace-backend/lovespace-ai/`（`com.meng.lovespace.ai.rag`、`com.meng.lovespace.ai.milvus`） |
| RAG 压缩与指标 | `lovespace-backend/lovespace-ai/src/main/java/com/meng/lovespace/ai/rag/compress/`、`metrics/` |
| Embedding 缓存 | `lovespace-backend/lovespace-ai/src/main/java/com/meng/lovespace/ai/embedding/cache/` |
| Facade / DTO | `lovespace-backend/lovespace-ai/` |
| 启动类扫描 | `lovespace-backend/lovespace-user/src/main/java/com/meng/lovespace/user/LoveSpaceUserApplication.java` |
| 前端 | `lovespace-frontend/src/pages/AILoveQA.tsx`、`lovespace-frontend/src/services/loveQa.ts` |

---

## 8. 嵌入与 OpenAI

默认 **DashScope** 嵌入；`application.yml` 排除 **`OpenAiEmbeddingAutoConfiguration`**。若改用 OpenAI 嵌入，需自行调整排除项与 `lovespace.ai.embedding.provider`，并保证 **`EmbeddingModel`** 与 Milvus 维度一致。

---

文档随实现迭代更新；架构总览见 `memory/decisions.md`，部署见 `memory/DEPLOYMENT.md`。
