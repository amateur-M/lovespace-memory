# 恋爱知识库（Love QA）与可选 RAG 说明

本文汇总「AI 恋爱问答 / RAG / Milvus」相关的前后端设计、实现要点、如何开启 RAG，以及部署与配置注意点。

---

## 1. 目标与分层

- **lovespace-ai**：LLM 抽象（通义千问 / OpenAI）、通用 DTO、`LoveQaChatFacade` 接口与恋爱问答相关异常；**不**默认携带 Milvus 与 Redis（RAG 专用）依赖。
- **lovespace-ai-rag**（可选模块）：Milvus 向量库、Spring AI `VectorStore`、Redis 多轮会话、`LoveQAService` 对 `LoveQaChatFacade` 的实现，以及在未启用 RAG 时排除 Milvus 自动配置的 `EnvironmentPostProcessor`。
- **lovespace-user**：`LoveQAController`（入库 / 聊天，仅在存在 `LoveQaChatFacade` Bean 时注册）、`LoveQAHistoryController` 与 MySQL 持久化（会话与消息列表）、安全与 OpenAPI。

---

## 2. 后端设计要点

### 2.1 模块与依赖

| 模块 | 作用 |
|------|------|
| `lovespace-ai` | `LLMProvider`、`LlmRouter`、`LovespaceAiProperties`（provider + travel）、`LoveQaChatFacade`、`LoveQaChatParams`/`LoveQaChatResult`、`LoveQaConversation*Exception` |
| `lovespace-ai-rag` | `spring-ai-starter-vector-store-milvus`、`spring-boot-starter-data-redis`、RAG 与 Milvus 骨架实现 |
| `lovespace-user` | Web、JWT、MyBatis、`LoveQAController` 等；**默认 POM 不依赖** `lovespace-ai-rag` |

`lovespace-user` 通过 Maven **Profile `lovespace-rag`** 按需引入 `lovespace-ai-rag`（见 `lovespace-user/pom.xml`）。默认打包产物**不包含** Milvus 相关 JAR，满足「无 Milvus 环境不拉向量库依赖」的诉求。

### 2.2 配置开关

- **`lovespace.ai.rag.enabled`**（默认 `false`）：业务上是否启用 RAG；为 `true` 时不应再排除 Milvus 自动配置（见下条）。
- **`lovespace.ai.rag.*`**：分片、`retrieveTopK`、Redis 会话 TTL、`max-history-pairs` 等（由 `lovespace-ai-rag` 内 `RagAiProperties` 绑定）。
- **`lovespace.milvus.*`**：扩展集合名、`ensure-travel-poi-schema` 等（旅游 POI 骨架）。
- **`spring.ai.vectorstore.milvus.*`**：Milvus 客户端、集合名、`embedding-dimension` 等（与 Spring AI 文档一致）。

`lovespace-ai-rag` 中的 **`RagMilvusAutoConfigurationExclusionEnvironmentPostProcessor`**：当 **`lovespace.ai.rag.enabled` 不为 true** 时，向 `spring.autoconfigure.exclude` 追加  
`org.springframework.ai.autoconfigure.vectorstore.milvus.MilvusVectorStoreAutoConfiguration`，避免未部署 Milvus 时仍初始化向量客户端。

### 2.3 Bean 装配关系（概念）

- `LoveQAController`：`@ConditionalOnBean(LoveQaChatFacade.class)`，仅当 RAG 模块在 classpath 且满足条件时出现实现类时注册。
- `LoveQAService`：`implements LoveQaChatFacade`，`@ConditionalOnProperty(lovespace.ai.rag.enabled=true)` 且 `@ConditionalOnBean(VectorStore.class)`。
- 多轮短期记忆：Redis（键前缀 `lovespace:love-qa:conv:`）；MySQL 侧由 `LoveQaConversationService` 做持久化与历史列表（与 RAG 是否启用独立，历史接口仍可用）。

### 2.4 API 约定（用户服务）

- `POST /api/v1/ai/love-qa/ingest`：知识库入库（需 RAG 开启且认证通过）。
- `POST /api/v1/ai/love-qa/chat`：多轮问答（长耗时，前端 `http` 建议 timeout ≥ 120s，与项目规范一致）。
- `GET /api/v1/ai/love-qa/conversations`、  
  `GET /api/v1/ai/love-qa/conversations/{id}/messages`：历史分页与消息（MySQL）。

---

## 3. 前端设计要点

- **路由**：`/love-qa`，入口在主导航（如 `AppLayout`）。
- **页面**：`lovespace-frontend/src/pages/AILoveQA.tsx`，调用 `services/loveQa.ts`。
- **请求路径**：与后端 `/api/v1/ai/love-qa/*` 对齐；聊天、入库使用 POST，历史使用 GET。
- **体验**：未启用后端 RAG 时，聊天/入库接口可能 404 或无对应 Bean；产品侧可后续加「能力探测」或文案提示（当前以服务端是否注册 Controller 为准）。

---

## 4. 如何开启并使用 RAG

### 4.1 构建：引入 RAG 模块

在项目根或 `lovespace-backend` 下执行（示例）：

```bash
mvn -pl lovespace-user -am package -Plovespace-rag
```

日常不启用 RAG 时，可省略 `-Plovespace-rag`，产物不含 `lovespace-ai-rag`。

### 4.2 配置：打开业务开关

在 `application.yml`（或环境变量覆盖）中设置：

```yaml
lovespace:
  ai:
    rag:
      enabled: true
```

并配置：

- **Milvus**：`spring.ai.vectorstore.milvus.client.host` / `port` 等，以及集合、`embedding-dimension` 与索引参数。
- **嵌入模型**：Milvus 依赖 **`EmbeddingModel`**。默认由 **`spring.ai.dashscope.api-key`** 驱动通义 **text-embedding**（`lovespace.ai.embedding.model`）；若改用 OpenAI 嵌入则配置 `spring.ai.openai.api-key` 并见下文「注意点」。
- **Redis**：多轮会话需要可用的 `StringRedisTemplate`（与现有 `spring.data.redis` 一致）。
- **LLM**：`lovespace.ai.provider` 与 `spring.ai.dashscope` / `spring.ai.openai` 等与现有 AI 模块一致。

### 4.3 运行与验证

- 启动 Milvus、Redis、MySQL；设置上述配置后启动 `lovespace-user`。
- Swagger：`/swagger-ui.html` 中应出现恋爱知识库相关接口（在 Controller 已注册的前提下）。
- 先调用 **ingest** 写入片段，再 **chat** 验证检索与多轮。

---

## 5. 注意点

### 5.1 嵌入模型与 RAG（默认通义 DashScope）

当前默认使用 **通义 DashScope 文本向量**（`lovespace.ai.embedding.*`，实现为 `DashScopeEmbeddingModel`），与聊天共用 **`spring.ai.dashscope.api-key`**；`spring.autoconfigure.exclude` 中包含 **`OpenAiEmbeddingAutoConfiguration`**，避免拉 OpenAI 嵌入。

若改用 **OpenAI 嵌入**：去掉对 `OpenAiEmbeddingAutoConfiguration` 的排除，配置 `spring.ai.openai.api-key`，并设置 `lovespace.ai.embedding.provider=openai`（且勿再注册 DashScope 的 `EmbeddingModel` Bean）。

若既不启用 OpenAI 嵌入自动配置、也未提供可用的 **`EmbeddingModel` Bean**，Milvus 向量存储无法在启动阶段正确装配。

### 5.2 `lovespace.ai.rag.enabled` 与 EPP

- `enabled=false`（默认）时，即使已加入 `lovespace-ai-rag` JAR，也会通过 EPP 排除 Milvus 自动配置，**不会**创建 `VectorStore`，`LoveQaChatFacade` 实现不会出现，`LoveQAController` 不注册。
- `enabled=true` 时必须保证 Milvus 与嵌入链路可用，否则可能在启动或首次向量操作时失败。

### 5.3 默认不打包 RAG

未使用 `-Plovespace-rag` 时，Fat JAR 中**没有** Milvus 相关依赖，适合无向量库的运行环境；需要 RAG 的部署流水线应固定带上该 Profile 或等价依赖声明。

### 5.4 安全与敏感信息

- 不在日志中打印完整 API Key、Token；长连接与超时策略与 `lovespace-coding-standards` 及 `http.ts` 约定一致。

### 5.5 数据双轨

- **Redis**：短期多轮上下文（TTL 由 `lovespace.ai.rag.conversation-ttl-seconds` 控制）。
- **MySQL**：会话与消息持久化、历史列表；与 RAG 开关解耦，便于未开 RAG 时仍查看历史（若库表已迁移）。

---

## 6. 相关路径速查

| 用途 | 路径 |
|------|------|
| User POM Profile | `lovespace-backend/lovespace-user/pom.xml`（`lovespace-rag`） |
| 应用配置 | `lovespace-backend/lovespace-user/src/main/resources/application.yml` |
| RAG 实现与 EPP | `lovespace-backend/lovespace-ai-rag/` |
| Facade 与 DTO | `lovespace-backend/lovespace-ai/`（`api/`、`dto/`、`exception/`） |
| 前端页面 | `lovespace-frontend/src/pages/AILoveQA.tsx` |
| 前端 API | `lovespace-frontend/src/services/loveQa.ts` |

---

文档随实现迭代可更新；重大架构变更请同步 `memory/decisions.md`（若项目约定要求）。
