# 实时进度

## 当前总体状态

- **核心功能已落地**：用户认证、个人资料（含头像上传）、情侣绑定、恋爱时间轴（含图片与视频 URL、区间筛选、作者编辑/删除、分页、可见性表单；直传或小文件 multipart，大文件公共分片；点赞与评论互动）、情侣相册（含相册内照片分页；大图分片 + from-url 登记；相册改名、照片元数据编辑）、私密消息（REST + WebSocket + 定时）、共同计划（计划 + 子任务 CRUD、计划消费流水 CRUD、预算按消费汇总、进度同步）均已落地并可联调
- **账号与登录（手机号）**：登录/注册使用 **中国大陆 11 位手机号**（`PhoneNormalizer` + `^1[3-9]\d{9}$`）；**邮箱**不在注册环节收集，在 `**PUT /api/v1/user/profile`** 中可选填或清空；**用户名**展示名可在资料页修改（唯一性校验）。**情侣邀请**：`POST /api/v1/couple/invite` body 为 `**inviteePhone`**（按手机号查找用户，不再使用 `inviteeUserId`）。JWT 含 `phone` claim（`JwtUtil.CLAIM_PHONE`）；`getPhone(claims)` 可回退读旧 token 的 `email` claim 以兼容过渡期。存量库执行 `sql/users_add_phone_account.sql`；新库用 `users.sql`（含 `phone` UNIQUE、`email` 可空）
- **纪念日倒计时**：后端 `/api/v1/memorial-days`（CRUD、`/next` 最近倒计时、`/upcoming` 近期窗口）、表 `memorial_days`、Redis 缓存 `lovespace:memorial:next:{coupleId}` / `upcoming:{coupleId}`、配置 `lovespace.memorial.*`、定时 `MemorialDayCacheRefreshTask`；`MemorialCountdownCalculator` 用 `MonthDay` 做每年循环与剩余毫秒。前端：`Memorial.tsx` + `memorialStore` + `services/memorial.ts`；路由 `/memorial` 已在 `App.tsx` 注册，顶栏有「纪念日」入口。页面为简约浪漫 + 手绘感（玫瑰主色一致）：`MemorialRomanticDecor`（SVG 爱心/星光）、`MemorialPhotoWall`（从相册拉取照片、双环错落环绕布局、中心柔光；无照片时占位相框）；**恋爱倒计时与祝福语区块在照片墙上方**；照片用 Ant Design `**Image` + `preview`**（列表用缩略图、预览优先原图 `imageUrl`）；**新建**主要靠右下角 FloatButton，折叠面板「全部纪念日」仅列表编辑/删除（**页内已移除月历与标题区新建按钮**）；内容区 `**w-full`** 与 `AppLayout` 的 `max-w-6xl` 对齐。字体：`index.html` 引入 Caveat、马善政；`index.css` 含 `memorial-display` / `memorial-accent` / `memorial-polaroid` 与星光动画（`prefers-reduced-motion` 下减弱）
- **AI 能力**（`lovespace-ai` + `lovespace-user`）：Spring AI 1.0（`spring-ai-starter-model-openai`，聊天可选 OpenAI）+ 通义千问（阿里云 DashScope Java SDK）；`LLMProvider`（`QwenProvider` / `OpenAiProvider`）、`LlmRouter` 与 `lovespace.ai.provider` 路由；`POST /api/v1/ai/chat` 通用对话；情感分析（`EmotionAnalysisService` + `GET /api/v1/ai/emotion`）；情书生成（`LoveLetterService` + `POST /api/v1/ai/love-letter`）；**Milvus 向量库**（`spring-ai-starter-vector-store-milvus`，与 LLM 同在 `**lovespace-ai`**）+ 恋爱 RAG（`LoveQAService`，`POST /api/v1/ai/love-qa/ingest`、`/chat`、`/chat/stream` SSE）：**默认打入 user 可执行包**（**无需** `-Plovespace-rag`），需 **Milvus**、`**EmbeddingModel`**、Redis；嵌入默认 `**DashScopeEmbeddingModel**`（`text-embedding-v2` 等），与聊天**共用** `spring.ai.dashscope.api-key`，配置见 `lovespace.ai.embedding.*`（**排除** `OpenAiEmbeddingAutoConfiguration`）；`**MilvusSchemaService`** 在 `**lovespace.milvus.ensure-love-knowledge-schema**`（默认 true）下启动自检建 `**love_knowledge_base**` 集合（与 Spring AI 表结构一致）；`**RagMilvusConfiguration**` 注册 `**lovespaceMilvusClient**` 避免与 Spring AI 的 `**milvusClient**` Bean 重名；**情侣旅游规划**（`TravelPlannerService`，`POST /api/v1/ai/travel/plan`，JSON 输出）。`application.yml` 含 `spring.ai.vectorstore.milvus.*`、`lovespace.ai.rag.*`、`lovespace.milvus.*`。父工程 Spring Boot 3.4.x + `spring-ai-bom` 以配套 Spring AI

   **RAG 优化增强**（2026-05）：
   - **Embedding 缓存**（`CachedEmbeddingModel`）：Redis 缓存 query → embedding 结果，标准化后 SHA-256 为 key；TTL 24h；预期 API 调用成本降低 60%，延迟 150ms → 5ms（命中时）。
   - **Latency 分解埋点**（`RagMetricsCollector`）：记录 EMBEDDING/RETRIEVE/PROMPT_BUILD/LLM_FIRST_BYTE/LLM_TOTAL/PERSIST 各阶段耗时，DEBUG 日志输出。
   - **检索可视化**：SSE 新增 `retrieved` 事件，前端展示 `📚 参考了 X 条知识` 引用卡片。
   - **Prompt 压缩**（`PromptCompressor`）：检索片段去重（Jaccard 0.85）、上下文长度限制 8000 字符、历史精简（保留最近 10 轮）；预期 Token 消耗降低 20-40%。
- 接口前缀统一 `/api/v1`；Vite 开发代理 `/api`、`/local-files`、`**/ws`（WebSocket，`ws: true`）** → 后端 8081
- 父 POM 已启用 `maven-compiler-plugin parameters=true`，且 Controller 中关键 `@RequestParam`/`@PathVariable` 已写显式名称，避免 Spring 绑定报错
- 前端已统一暖色主题（玫瑰主色 + 浅粉背景，Inter 字体），相册照片列表为等尺寸网格（aspect-square + object-cover）
- **相册上传**：Controller 层不再校验单文件大小；`spring.servlet.multipart` 放宽至 10GB（实际仍受 Tomcat、反向代理、磁盘限制）。大图（前端默认大于 4MB）走 `/api/v1/media/uploads` 分片，完成后 `POST /api/v1/albums/{id}/photos/from-url` 落库
- **本机静态资源**：`/local-files/`** 由 `LocalFileRangeController` 提供，支持 Accept-Ranges 与 206 Partial Content（利于视频拖拽进度）；已移除仅 ResourceHandlerRegistry 的 file 映射
- **公共分片上传**：`ChunkedMediaUploadService` + `/api/v1/media/uploads`（init / PUT .../chunks/{i} / status / complete / DELETE）；暂存 `_pending/media/{uuid}/`，定时任务清理超时会话；兼容旧路径 `_pending/timeline/` 的未完成会话
- **私密消息**：Spring Security 放行 `/ws/`** 握手；JWT 在 JwtAuthenticationFilter 中对 WebSocket 路径跳过（避免无 Authorization 头导致握手失败）；浏览器端通过 `?token=` 传 JWT，握手后在 ChatWebSocketHandler 内校验
- **聊天页前端**：关闭轮询，以 WebSocket 实时增量为主；历史首屏仍用 GET /api/v1/messages。WS 已连接时即时/定时仍走 WS JSON（send / scheduled）；**WS 未就绪时回退 REST**（`POST /api/v1/messages/send`、`POST /scheduled`），返回体 `upsertMessage` 去重。WebSocket `useEffect` 依赖 **`partner`**（与 effect 内 `!partner` 守卫一致）
- **可选分布式 Session（Redis）**：`spring-session-data-redis`；`lovespace.session.distributed.enabled` 默认 false；启用时需 `spring.session.store-type=redis` 与可用 Redis。登录写入 Spring Security 上下文至 Session（AuthController + HttpSessionSecurityContextRepository），登出 invalidate Session。JwtAuthenticationFilter：Bearer 过期/无效时若已启用分布式 Session则不直接 401，以便回退 Session Cookie；黑名单仍 401。DistributedSessionCorsConfig：启用分布式 Session 时注册带凭证 CORS（直连 API 跨域时需要）。Cookie 名 LOVESPACE_SESSION
- **前端认证**：authStore 增加 authHydrated；AppLayout 在 hydration 完成前不渲染 Outlet（避免刷新时 isAuthed 尚未从 localStorage 恢复即触发 Navigate → /login）。VITE_SESSION_DISTRIBUTED=true 时 http withCredentials；hydrate 无 JWT 时可 getProfile 依赖 Session
- **前端全局样式**：@ant-design/cssinjs StyleProvider hashPriority="high"（与 Tailwind 共存）；index.css 将 prefers-reduced-motion 限定在 #root *，避免破坏挂到 body 的 Ant Design 浮层；ConfigProvider getPopupContainer={() => document.body}；主题 zIndexPopupBase: 1050
- **页面体验**：情侣首页 Hero、快捷入口、解除关系弱化；AI 情书去掉与按钮重复的 Alert；AppLayout flex 布局 + Footer mt-auto 使页脚贴视口底（内容短时）
- **情侣邀请与「消息」**：GET/POST /api/v1/couple/pending-invites、GET .../pending-invites/count；被邀请方在 /inbox 列表一键接受，无需手输绑定码；AppLayout 下拉消息列表 + inboxStore 角标；CoupleHome 以「查看消息」为主流程
- **容器部署**：根目录 docker-compose.yml（**milvus-etcd**、**milvus**、MySQL / Redis、与 Milvus **共用**业务 **MinIO**、backend / frontend）、双 Dockerfile、env.example、前端 nginx.conf 反代 /api、/local-files、/ws；后端环境变量含 `**MILVUS_HOST`**、`**SPRING_AI_DASHSCOPE_API_KEY**`、`**SPRING_AI_VECTORSTORE_MILVUS_INITIALIZE_SCHEMA**`、`**LOVESPACE_MILVUS_ENSURE_LOVE_KNOWLEDGE_SCHEMA**` 等；说明见 DEPLOYMENT.md
- **Maven 可执行包**：父 POM pluginManagement 为 spring-boot-maven-plugin 指定 ${spring-boot.version}；lovespace-user 显式 repackage 与 mainClass，避免 fat jar 无 Main-Class
- **后台管理系统（2026-06）**：`users.role`（0 USER / 1 ADMIN）+ JWT `role` claim + `/api/v1/admin/**`（`Admin*Controller` / `Admin*Service`）；前端同应用 `/admin/*`（`AdminLayout`、`AdminRoute`、各管理页）；首个管理员见 `lovespace.admin.bootstrap-phone` 或 `sql/users_add_role.sql`；原 `/users` 越权接口已删除
- **MinIO 访问 URL**：lovespace.minio.public-base-url 若配置，MinioAvatarStorageService 返回 {publicBaseUrl}/{bucket}/{objectKey}；未配置时用 http://{endpoint主机}/{bucket}/{objectKey}，生产应对外填写 LOVESPACE_MINIO_PUBLIC_BASE_URL；浏览器直链对象需桶匿名读策略，否则 403

---

## 后端（`lovespace-backend`）

### 模块


| 模块                 | 说明                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lovespace-common` | 统一响应 ApiResponse 等                                                                                                                                                                                                                                                                                                                                                                                                    |
| `**lovespace-ai`** | LLM 抽象（LLMProvider）、LlmRouter、QwenProvider（DashScope）、OpenAiProvider（Spring AI OpenAiChatModel）、`**DashScopeEmbeddingModel**`（`EmbeddingModel`）、AiChatService、旅游规划（`TravelPlannerService`、`TravelPlannerController`）、AiChatController（POST /api/v1/ai/chat）；以及 Milvus `VectorStore`、`LoveQAService`、`DocumentIngestPipeline`、`**MilvusSchemaService**`、`**RagMilvusConfiguration**`（`**lovespaceMilvusClient**`）等 RAG |
| `lovespace-user`   | 可运行用户服务（端口默认 8081）；依赖 **lovespace-ai**；`@ComponentScan` 含 **com.meng.lovespace.user** 与 **com.meng.lovespace.ai**；`**LoveQAController`** 为 `@RestController` 扫描注册                                                                                                                                                                                                                                                     |


### 数据表脚本（`lovespace-user/src/main/resources/sql/`）

- users.sql — 用户表（`phone` 登录账号 UNIQUE、`email` 可空）；存量迁移见 users_add_phone_account.sql
- couple_binding.sql — 情侣绑定（含 status：0 待接受、1 交往、2 冻结、3 解除）
- love_records.sql — 时间轴记录（含 images_json）
- love_records_add_images_json.sql — 已有库仅缺图片列时执行
- love_record_social.sql — love_record_likes、love_record_comments（恋爱记录点赞/评论，删记录级联）
- couple_albums.sql — couple_albums、album_photos
- private_messages.sql — private_messages（私密消息：类型、定时、已读、撤回等）
- couple_plans.sql — couple_plans、plan_tasks（共同计划与子任务；plan_tasks.plan_id 外键级联删除）
- plan_expenses.sql — plan_expenses（计划消费：expense_type 为 lodging|transport|dining|other；plan_id 外键级联删除）
- memorial_days.sql — memorial_days（纪念日：couple_id、user_id、name、memorial_date 等）
- love_qa.sql — love_qa_conversations、love_qa_messages（恋爱问答会话与消息持久化）
- love_qa_documents.sql — love_qa_documents（恋爱知识库文档台账：状态、chunkCount、content 快照）

### 主要 API 分组

- **认证/资料**：`POST /auth/register`（phone、username、password）、`POST /auth/login`（phone、password）；`/api/v1/user/profile` GET/PUT（PUT 可含 username、email；email 空串清空；头像 POST `/profile/avatar`）
- **情侣**：/api/v1/couple/*（`POST /invite` body 为 **inviteePhone**；accept、info、start-date、separate；GET /couple/pending-invites、GET .../pending-invites/count 待处理邀请）
- **时间轴**：/api/v1/timeline/*
  - POST /upload — 时间轴图片或视频整文件 multipart → URL（lovespace.timeline-upload 分档大小与扩展名；与 images_json 数组字段共用）
  - GET/POST /records，GET/PUT/DELETE /records/{id} — 列表/详情/新建返回的 LoveRecordResponse 含 likeCount、commentCount、likedByMe（LoveRecordSocialService 批量统计）
  - GET /records?coupleId&page&pageSize&startDate&endDate — 可选日期区间
  - GET /memories — 回忆推送（同上含互动统计）
  - POST /records/{id}/like — 切换点赞 → LoveRecordLikeStateResponse
  - GET /records/{id}/comments — 分页评论（时间正序，pageSize≤50）
  - POST /records/{id}/comments — body LoveRecordCommentCreateRequest（≤500 字）
  - DELETE /records/{id}/comments/{commentId} — 仅评论作者
- **公共媒体分片（时间轴 + 相册）**：/api/v1/media/uploads/*（需登录）
  - POST /init — body：target=TIMELINE|ALBUM，ALBUM 时必填 albumId，及 fileName/fileSize/可选 contentType
  - PUT /{uploadId}/chunks/{chunkIndex} — 原始二进制，Content-Length 须与该片长度一致
  - GET /{uploadId}/status、POST /{uploadId}/complete → 返回最终 URL；DELETE /{uploadId} 取消
  - 配置：lovespace.timeline-upload.chunk-size-bytes、pending-session-hours；相册分片仅允许头像同款图片扩展名，单文件上限与时间轴图片上限一致
- **相册**：/api/v1/albums/*
  - PUT /albums/{id} — AlbumUpdateRequest 改相册名称（情侣成员）
  - GET /albums/{id}/photos?page&pageSize — 分页列出照片（默认 page=1、pageSize=10，最大 100）；响应体 AlbumPhotoPageResponse
  - PUT /albums/{albumId}/photos/{photoId} — PhotoUpdateRequest 整单更新描述、locationJson、takenDate、tagsJson（情侣任一方，四字段均可为 null 清空）
  - POST /albums/{albumId}/photos/from-url — JSON PhotoRegisterFromUrlRequest：分片合并得到的 imageUrl + 可选元数据，AlbumImageUrlValidator 校验路径属当前用户 albums/日期/userId/
  - 其余：创建相册、列表、删相册、POST .../photos multipart 上传、删照片、收藏等
- **私密消息**：/api/v1/messages/*
  - POST /send、GET（列表）、PUT /{id}/read、POST /{id}/retract、POST /scheduled（定时创建）
  - 业务：MessageServiceImpl + MessageBusinessException + MessageControllerExceptionHandler
  - 撤回：created_at 起 2 分钟内可撤回，否则 40078
- **WebSocket**：GET ws://.../ws/chat?token=...（非 /api 前缀），文本 JSON：subscribe / send / scheduled / read / retract；落库后 ChatSessionRegistry.broadcastToCouple 推送 {type:"privateMessage",data:...}
- **共同计划**：/api/v1/plans（POST 创建、GET?coupleId= 列表、PUT /{id} 更新、DELETE /{id} 删除）；/api/v1/plans/{id}/tasks（POST 创建任务）；/api/v1/plans/{id}/tasks/{taskId}（PUT + PlanTaskReplaceRequest、DELETE）；/api/v1/plans/{id}/expenses（GET 列表、POST PlanExpenseCreateRequest）；/api/v1/plans/{id}/expenses/{expenseId}（PUT PlanExpenseReplaceRequest、DELETE）。PlanResponse 含 budgetSpent（消费汇总）与 expenseSummary（按类型拆分）；创建/更新计划不再接受手工 budgetSpent。业务：PlanServiceImpl + PlanExpenseMapper + PlanBusinessException + PlanControllerExceptionHandler。
- **纪念日**：/api/v1/memorial-days（POST / GET?coupleId= / GET /{id} / PUT /{id} / DELETE /{id}）；GET /memorial-days/next?coupleId=（最近一个 + millisecondsUntilNext）；GET /memorial-days/upcoming?coupleId=（窗口内列表）；情侣成员校验同 findActiveOrFrozenMembership；MemorialDayRedisCache + MemorialDayServiceImpl；异常 MemorialDayBusinessException（示例码 40490 / 40390 / 40091）+ MemorialDayControllerExceptionHandler。
- **AI（/api/v1/ai，需登录）**：
  - POST /ai/chat — body AiChatRequest（message），返回 AiChatResponseData（reply）；路由 lovespace.ai.provider：qwen | openai。
  - GET /ai/emotion?coupleId&startDate&endDate — EmotionAnalysisController；EmotionAnalysisReport：overallMood、moodScore、emotionDistribution、trendData、insights；统计来自 LoveRecordService.listVisibleRecordsInRange + 通义 JSON 解析；区间默认近 30 天、最长 366 天；TimelineControllerExceptionHandler 已包含 EmotionAnalysisController。
  - POST /ai/love-letter — body LoveLetterGenerateRequest（coupleId、style、length、memories 可选）；LoveLetterService 填充发送方/接收方/恋爱天数，回忆可选手写或由近 1 年记录摘要；LoveLetterControllerExceptionHandler。
  - POST /ai/love-qa/ingest|ingest/file|ingest/url — 写 MySQL 台账 + Milvus；返回 `LoveQaIngestResponseData`（documentId、status、chunkCount）；`LoveQaDocumentService` + `LoveQAService.ingestDocumentWithTracking`。
  - GET /ai/love-qa/documents — 文档台账分页（可选 coupleId）；GET /documents/{id} 详情；DELETE /documents/{id} 删向量+台账；POST /documents/{id}/reingest 重入库。
  - POST /ai/love-qa/chat — LoveQaChatRequest（message、可选 conversationId、coupleId）；**非流式** `ApiResponse`；Redis 多轮（`lovespace:love-qa:conv:*`）；每轮成功后 **MySQL** `appendChatRound`；检索 Milvus 后 RAG；前端 `http` timeout ≥120s 仍可用。
  - POST /ai/love-qa/chat/stream — **SSE**（`text/event-stream`）；事件 **meta**（conversationId）/ **delta**（增量 `t`）/ **done**（reply + conversationId）/ **error**；`LoveQaChatFacade.chatStream` + 虚拟线程；流结束后 **MySQL appendChatRound**；主站前端 `**postLoveQaChatStream`**（`fetch` + ReadableStream）实时拼 assistant。
  - **会话**：`/chat` 与 `/chat/stream` 在带 `conversationId` 时先 `**restoreRedisSessionIfMissing`**（Redis 无则从 MySQL 恢复轮次）。LLM 走 `**chatWithSystemAndHistory(Streaming)**`（多消息 + system 上下文）。
  - GET /ai/love-qa/conversations?page&pageSize — `LoveQAHistoryController`：当前用户会话列表（摘要标题、更新时间）。
  - GET /ai/love-qa/conversations/{conversationId}/messages — 该会话下全部消息（按 id 升序）；业务码 40492 不存在、40392 非本人会话。
  - POST /ai/travel/plan — TravelPlanRequest（destination、days、budgetMin/Max、preferences、transportMode）；返回 `TravelPlanJsonSchema`；`AiFeaturesExceptionHandler` 处理 LLM 不可用 / JSON 非法。
- **时间轴扩展**：LoveRecordService.listVisibleRecordsInRange — 情侣成员下、日期区间内可见记录全量列表（不分页），供情感分析等使用。
- **实现注意**：listPlans 内对任务列表排序前使用 new ArrayList<>(getOrDefault(..., List.of()))，避免对不可变空列表 sort 抛异常。

### 定时任务

- ScheduledMessageDispatchTask：每 15s 扫描到期定时消息，置 is_scheduled=0 并广播
- MediaChunkCleanupTask：约每小时清理 _pending/media 与旧 _pending/timeline 中超时的分片会话目录
- MemorialDayCacheRefreshTask：每日定时对 memorial_days 中出现过的 couple_id 调用 warmCache 预热 Redis（写操作仍会 evictCouple）

### 领域要点

- CoupleBindingService.findActiveOrFrozenMembership：校验 coupleId（相册、时间轴、私密消息、共同计划、纪念日均复用）
- 时间轴列表可见性：visibility=2 或（visibility=1 且作者为当前用户）；改删仅作者；赞与评同样受单条可见性约束（LoveRecordSocialService.requireViewableRecord）
- 心情：LoveMood（happy/sad/excited/calm/loved/missed）
- 存储：头像、时间轴图/视频、相册图共用 AvatarStorageService；路径前缀 timeline/、albums/；支持 uploadTimelineFromLocalFile、uploadAlbumFromLocalFile（分片合并落盘后再发布）
- 启动类：LoveSpaceUserApplication 含 @EnableScheduling

---

## 前端（`lovespace-frontend`）

### 路由（App.tsx）

- /login、/register（AuthPageShell 暖色渐变背景）
- / 首页、/profile 个人资料、/couple 情侣首页、/inbox 消息（待处理情侣邀请）、/timeline 时间轴、/album 相册、/chat 私密消息、/plan 共同计划、**/memorial 纪念日**、/emotion 情感洞察、**/love-qa 恋爱问答**、/love-letter AI 情书
- * → 404

### 状态与服务

- authStore（含 authHydrated、hydrate 异步恢复）+ services/auth.ts（`User` 含 `phone`；`login`/`register` 传手机号；`updateProfile` 含 username、email）、http.ts（可选 withCredentials、VITE_SESSION_DISTRIBUTED）
- coupleStore + services/couple.ts（`inviteCouple(inviteePhone)`；bindingId 作 coupleId）；listPendingInvites / getPendingInviteCount
- inboxStore：pendingCount、refreshPendingCount（顶栏角标）
- services/timeline.ts — 列表（含区间）、创建、更新、删除、uploadTimelineMedia（整文件）；toggleTimelineLike、listTimelineComments、postTimelineComment、deleteTimelineComment；可见性常量 VISIBILITY_*；LoveRecord 含 likeCount/commentCount/likedByMe
- services/mediaChunkUpload.ts — uploadMediaResumable（target: TIMELINE | ALBUM + 可选 albumId）、uploadTimelineMediaAuto（阈值默认 4MB）、MEDIA_CHUNK_THRESHOLD_BYTES；请求 /api/v1/media/uploads
- services/album.ts — 相册与照片 CRUD、updateAlbum、updateAlbumPhoto、uploadAlbumPhoto、uploadAlbumPhotoAuto、registerAlbumPhotoFromUrl、收藏；listAlbumPhotos
- utils/timelineMedia.ts — isTimelineVideoUrl、校验与 20MB 图 / 100MB 视频等与后端对齐的常量
- services/plan.ts — 计划与任务 API；消费流水 listPlanExpenses / createPlanExpense / updatePlanExpense / deletePlanExpense；normalizePlan / getPlanTasks 兜底 tasks；expenseSummary 与 PLAN_EXPENSE_TYPE_OPTIONS 等常量
- services/emotion.ts — getEmotionReport（GET /api/v1/ai/emotion，超时 120s；可选 **AbortSignal** 供页面取消 StrictMode / 快速切换日期时的重复请求）
- services/loveLetter.ts — generateLoveLetter（POST /api/v1/ai/love-letter，超时 120s）
- services/memorial.ts — 纪念日 CRUD、getNextMemorial、listUpcomingMemorials（与后端 /api/v1/memorial-days 对齐）
- **services/loveQa.ts** — 恋爱问答 RAG：`postLoveQaChatStream`（SSE）、`postLoveQaIngest*`、会话历史；`RetrievedChunk` 类型；超时 ≥120s
- memorialStore（Zustand）— 列表、nextPayload、countdownAnchor（轮询与本地秒级倒计时对齐）、invalidate

### UI 与主题

- src/theme/antdTheme.ts + App.tsx ConfigProvider：locale=zhCN，暖色 token
- index.css：ls-surface、ls-page-intro、ls-link；Inter 字体
- 顶栏：AppLayout 含「私密消息」「共同计划」「**纪念日**」「情感洞察」「**恋爱问答**」「AI情书」等入口

### 页面与组件摘要

- **时间轴**：Timeline.tsx（工具栏日期筛选、Pagination 每页 10 条、无顶部大标题 Card；赞/评变更后 loadPage 刷新当前页）；TimelineForm（谁可见默认双方；mediaChunkUpload 大图/视频分片 + 断点续传）；TimelineItem（图片 + video 展示；作者右上角 ⋯；底部 TimelineRecordSocial：心形赞、评论展开、分页加载、发送/删除自己的评论；列表刷新时依赖 record.commentCount 重拉评论避免列表与条数不一致）
- **共同计划**：Plan.tsx — 主从布局（左侧 PlanListItem 仅摘要：标题、类型、短日期、微型进度；右侧 PlanDetailHero 完整详情、统计卡、预算与消费、进度条）；PlanExpensePanel（消费明细表、记一笔 Modal）；PlanForm（无手工「已花费」）；TaskFormModal、TaskItem、ProgressBar、BudgetTracker；任务变更后 updatePlan({ progress }) 与任务完成比例对齐（computeTaskProgressPercent）。已移除旧版 PlanCard（由侧栏行 + 详情头图替代）
- **情侣**：CoupleHome（邀请弹窗输入对方 **手机号**；Hero、「一起去看看」快捷链、底部弱化解除关系）、CoupleCard、DaysCounter
- **相册**：Album.tsx（uploadAlbumPhotoAuto；saveAlbumName；进入相册后 Pagination（PHOTO_PAGE_SIZE，当前前端常量多为 12；后端单页上限仍为 100）；上传成功后回到第 1 页并刷新）；AlbumCard（仅封面进入；双击名称内联编辑、回车保存、Esc 取消）；PhotoGrid、PhotoViewer（灯箱内编辑信息 Drawer，元数据保存后 patchPhotoMeta）、UploadButton
- **私密消息**：Chat.tsx；MessageBubble、MessageInput、ScheduledPicker（一体化输入区、会话时间格式、滚动加载更早消息）；见上「WS + HTTP 回退」
- **工具**：utils/mediaUrl.ts（`resolveMediaUrl` — 时间轴/相册/纪念日媒体及 **Avatar 头像** 相对路径 `/local-files/**` 均需经此拼接 `VITE_API_BASE_URL`）；utils/mood.ts（`MOOD_OPTIONS`、`moodLabel`，供 TimelineForm / 情感洞察）；utils/timelineMedia.ts、utils/planProgress.ts
- **情感洞察**：EmotionAnalysis.tsx + EmotionAnalysisCharts.tsx（ECharts 环形图 + 折线面积图）；日期区间、仪表盘分数、AI 建议、分布列表；加载报告用 **AbortController** 避免开发环境 StrictMode 下重复请求与 loading 竞态；**EmotionAnalysisCharts** 内 **单次 effect** 同时初始化两图并统一 resize/dispose，减轻双 effect 带来的重复初始化与闪动
- **AI 情书**：AILoveLetter.tsx — 风格/篇幅/可选回忆、生成结果与复制；表单下仅保留简短耗时提示（已去与按钮重复的 Alert）
- **纪念日**：`Memorial.tsx` + `MemorialDayFormModal.tsx` + `components/memorial/MemorialRomanticDecor.tsx` + `MemorialPhotoWall.tsx` — 标题区（手绘标题 + 波浪线文案）；**倒计时与祝福语在照片墙之上**；照片墙从相册聚合、环绕式双环错落、**点击照片 Ant Design Image 预览放大**（mask 文案「查看」）；FloatButton 新建；折叠「全部纪念日」列表（编辑/删除）；无页内月历与标题下「新建纪念日」主按钮；倒计时每秒刷新（`tick` 触发重算 + `remainingMsFromAnchor` 用 `Date.now()`，无多余 `useMemo`）
- **恋爱问答**：`AILoveQA.tsx` + `services/loveQa.ts` — 侧栏会话列表 + 主区千问式布局；SSE 流式问答；assistant 消息展示 **检索来源卡片**（摘要 + `[n]` 卡片含 source/textPreview；正文 `【n】`/`[n]` 可点击滚动高亮对应卡片，`id=source-{messageKey}-{idx}`）；知识库入库 Modal（文本/文件/URL 三 Tab）

### 验证

- `npm run build` 与 `npx eslint .` 均已通过（2026-06-02 前端代码审查修复后）

---

## 对话任务摘要（按时间线合并）

### 历史基线（任务 1-6）

JavaDoc 与后端多文件注释规范；情侣绑定全链路；情侣首页；恋爱时间轴；图片与 images_json、POST /timeline/upload；Spring 参数名 -parameters + 显式 @RequestParam/@PathVariable；时间轴日期区间后端 startDate/endDate，前端 RangePicker；情侣相册前后端（表、实体、API、相册页、缩略图网格、multipart 配置）；暖色主题与 CODING_RULES.md。

### 私密消息与聊天（任务 7-12）

private_messages 表、PrivateMessage、MessageMapper/Service/Controller、定时任务、消息业务码与 MessageControllerExceptionHandler；前端 Chat.tsx + 气泡/输入/定时选择；路由 /chat 与导航；撤回 2 分钟限制（前后端）；WebSocket：spring-boot-starter-websocket、WebSocketConfig、ChatWebSocketHandler、ChatSessionRegistry；Security 放行 /ws/**；JWT Filter 跳过 /ws/；握手后 subscribe + 业务消息；落库后广播；前端聊天轮询关闭；**Vite 代理 `/ws` → 8081**（`VITE_API_BASE_URL` 为空时同源 WS 可达后端）；**WS 不可用时 HTTP 发送兜底**；`partner?.id` 稳定 WS 订阅 effect；聊天 UI：会话下边线、时间文案（当天/昨天/星期）、自适应高度、底部最新消息、输入区与发送按钮一体化。

### 时间轴页增强（任务 13-15）

去掉顶部「恋爱时间轴」大 Card，保留记录日期 RangePicker 工具栏；编辑/删除（仅作者）、Pagination 默认每页 10 条；新建/编辑表单增加谁可见（默认双方可见），列表展示「仅自己可见」提示。

### 相册照片分页（任务 16-17）

后端 AlbumPhotoPageResponse、AlbumService.pagePhotos、GET /albums/{id}/photos 支持 page/pageSize（MyBatis-Plus selectPage）；前端 Album.tsx + services/album.ts：AlbumPhotoPage 类型；网格下方 Ant Design Pagination，默认每页 10 张；灯箱仅切换当前页内照片。

### 共同计划（任务 18-23）

表 couple_plans、plan_tasks；实体 Plan、PlanTask；PlanMapper、PlanTaskMapper；PlanService/Controller（计划 CRUD、任务创建；任务更新为 PUT + PlanTaskReplaceRequest、DELETE 任务；情侣成员校验同相册）；前端 /plan、services/plan.ts、Plan.tsx 与上述组件；列表 + 详情、预算与进度展示、任务进度回写；修复：前端对 tasks null/非数组归一化；后端 listPlans 排序前复制为可变 ArrayList；任务与计划编辑/删除收入右上角 ⋯ 菜单，与 TimelineItem 交互一致；时间轴删除菜单项补删除图标；表 plan_expenses、实体 PlanExpense、PlanExpenseMapper（含 sumAmountByPlanId）；PlanExpenseSummaryResponse、PlanExpenseResponse、PlanExpenseCreateRequest、PlanExpenseReplaceRequest；PlanResponse 增加 expenseSummary；budgetSpent 以消费汇总为准并回写 couple_plans.budget_spent；PlanController 增加 /plans/{id}/expenses CRUD；前端 services/plan.ts 消费 API 与类型；PlanExpensePanel；BudgetTracker 可选展示四类 breakdown；PlanForm 去掉手工已花费字段；侧栏 + 详情分层，避免左右重复堆叠同一套 PlanCard；新增 PlanListItem、PlanDetailHero；删除 PlanCard.tsx；顶栏文案与主按钮圆角、侧栏 sticky、空状态占位优化。

### 时间轴视频与上传能力增强（任务 24-27）

时间轴视频：lovespace.timeline-upload 分档（图/视频扩展名与大小）；TimelineController 直传；images_json 仍为 URL 字符串数组；前端 TimelineForm/TimelineItem 与 timelineMedia.ts；TimelineUploadProperties 增加 chunk-size-bytes、pending-session-hours；初版分片 API 在 /api/v1/timeline/uploads（已迁移）；Range：删除 WebStaticResourceConfig 的静态映射；LocalFileRangeController 处理 GET /local-files/{*filepath}，返回 Accept-Ranges 与单段 206；公共分片：ChunkedMediaUploadService、MediaChunkUploadController（/api/v1/media/uploads）、MediaChunkBusinessException + MediaChunkExceptionHandler；upload 包（MediaChunkTarget、MediaUploadSessionMeta、AlbumImageUrlValidator）；AlbumService.registerPhotoFromUploadedUrl + PhotoRegisterFromUrlRequest；前端 mediaChunkUpload.ts；原 /api/v1/timeline/uploads 已废弃，外部客户端需改 /api/v1/media/uploads。

### 恋爱记录点赞与评论（任务 28-30）

表 love_record_likes、love_record_comments（love_record_social.sql）；实体与 Mapper；LoveRecordSocialService/Impl；LoveRecordResponse 扩展 likeCount、commentCount、likedByMe；LoveRecordServiceImpl 列表/详情/新建/回忆走 enrichWithSocial；TimelineController：POST .../like、GET/POST .../comments、DELETE .../comments/{commentId}；前端 TimelineRecordSocial.tsx、services/timeline.ts 互动 API；TimelineItem 嵌入互动区；UI 参考 ui-ux-pro-max 暖色与图标规范（Ant Design 图标，非 emoji）；TimelineRecordSocial：useEffect 在 panelOpen 且 record.commentCount（及 record.id/loadComments）变化时重新 loadComments(1)，修复仅条数更新而评论列表不刷新。

### 相册改名与照片信息编辑（任务 31-32）

AlbumUpdateRequest、PUT /api/v1/albums/{id}；PhotoUpdateRequest、PUT /api/v1/albums/{albumId}/photos/{photoId}（AlbumServiceImpl.updateAlbum/updatePhoto）；情侣成员改相册名；情侣任一方改照片元数据；takenDate 不可晚于当日；前端 updateAlbum、updateAlbumPhoto；PhotoViewer 侧栏编辑 Drawer（zIndex 高于灯箱）；初期曾用重命名 Modal + 内页按钮，后改为仅列表双击名称内联编辑（AlbumCard onSaveName），移除相册内标题旁「编辑名称」与卡片角标按钮。

### AI 服务框架与情感分析（任务 33-36）

新增模块 lovespace-ai：依赖 spring-ai-bom 1.0.0、spring-ai-starter-model-openai；通义千问使用 com.alibaba:dashscope-sdk-java（官方 spring-ai-bom 不含 DashScope）。QwenProvider（chat / chatWithSystem）、OpenAiProvider（ChatClient + OpenAiChatModel）、LovespaceAiProperties（lovespace.ai.provider）。父 POM：Spring Boot 3.4.5、Spring Cloud 2024.0.0、lovespace-user 依赖 lovespace-ai；application.yml：spring.ai.dashscope.api-key、spring.ai.model、spring.ai.openai.*、lovespace.ai.provider；POST /api/v1/ai/chat：AiChatController + AiChatService；LoveRecordService.listVisibleRecordsInRange；EmotionAnalysisService + EmotionAnalysisController GET /api/v1/ai/emotion；DTO EmotionAnalysisReport、EmotionTrendPoint；TimelineControllerExceptionHandler 增加 EmotionAnalysisController；前端 /emotion、services/emotion.ts、EmotionAnalysis.tsx、EmotionAnalysisCharts.tsx；顶栏「情感洞察」。**后续**：情感页加载用 AbortSignal、图表单次 effect，见上文「情感洞察」要点。

### 情书生成（任务 37-38）

LoveLetterService/Impl、LoveLetterController POST /api/v1/ai/love-letter；DTO LoveLetterGenerateRequest、LoveLetterResponseData；LoveLetterBusinessException + LoveLetterControllerExceptionHandler；前端 /love-letter、AILoveLetter.tsx、services/loveLetter.ts；顶栏「AI情书」。

### 前端 Ant Design 浮层与全局样式（任务 39-41）

main.tsx：StyleProvider（@ant-design/cssinjs）hashPriority="high"，缓解 Tailwind 与 Ant Design 5 :where 低特异性导致 Select/DatePicker/Dropdown 浮层异常；index.css：@media (prefers-reduced-motion: reduce) 仅作用于 #root *，避免全局 * + !important 破坏 body 上 Ant Design 浮层的 rc-motion；App.tsx：ConfigProvider getPopupContainer={() => document.body}；antdTheme：token.zIndexPopupBase: 1050（高于灯箱 z-[1000] 等）。

### 可选分布式 Session 与 JWT/Session 协同（任务 42-45）

依赖：spring-session-data-redis；配置 LovespaceSessionProperties（lovespace.session.distributed.enabled，默认 false）、spring.session.*（默认 store-type: none；启用时 redis + namespace/timeout）；SecurityConfig：启用时 SessionCreationPolicy.IF_REQUIRED；AuthController：分布式 Session 开启时登录后 securityContextRepository.saveContext；登出 session.invalidate()（仍可对 JWT 黑名单）；JwtAuthenticationFilter：Bearer 解析失败时若 lovespace.session.distributed.enabled=true 则继续过滤器链（保留 Session 认证）；黑名单仍 401；DistributedSessionCorsConfig：@ConditionalOnProperty，启用分布式 Session 时 CORS 带凭证（localhost/127.0.0.1 等 pattern）。

### 情侣首页、AI 情书、页脚布局（任务 46-48）

CoupleHome.tsx：未绑定/已绑定 Hero、已绑定快捷入口（时间轴/相册/消息/计划）、解除关系改为底部虚线区链接 + Popconfirm 说明；AILoveLetter.tsx：移除与主按钮重复的 Alert，仅保留一行耗时说明；Empty 文案调整；AppLayout：Layout flex min-h-screen flex-col，Content flex-1，Footer mt-auto shrink-0 贴底。

### 刷新页面误跳转登录修复（任务 49）

authStore：新增 authHydrated（初始 false）；hydrate() 结束或 login/logout 时置 true；AppLayout 在 !authHydrated 时显示加载态，不挂载 Outlet，避免各页 Navigate to=/login 早于 localStorage 恢复执行。

### 情侣邀请流程与消息页（任务 50-51）

后端：CouplePendingInviteResponse；CoupleBindingService listPendingInvitesForInvitee / countPendingInvitesForInvitee（PENDING + user_id2=被邀请方）；CoupleController GET /api/v1/couple/pending-invites、GET .../pending-invites/count；前端：/inbox Inbox.tsx；inboxStore；AppLayout 下拉消息列表 + Badge；CoupleHome 弱化手输绑定码，引导查看消息；Chat.tsx 在 VITE_API_BASE_URL 为空时用 window.location.host 推导 WebSocket 基址（同源 + Nginx /ws/）。

### Docker 与 Maven 可执行包（任务 52-54）

根目录 docker-compose.yml、lovespace-backend/Dockerfile（app.jar 或 target/*.jar）、lovespace-frontend/Dockerfile（dist/ + nginx.conf）、env.example；data/mysql|redis|minio|uploads 持久化；父 POM spring-boot-maven-plugin 补 version；lovespace-user repackage + LoveSpaceUserApplication mainClass，保证 java -jar 可用；MinIO：public-base-url 与 buildUrl 含 /{bucket}/ 段；生产配置 LOVESPACE_MINIO_PUBLIC_BASE_URL；桶策略匿名读与 403 说明见 DEPLOYMENT.md。

### 纪念日倒计时（任务 55-58）

后端：表 memorial_days、实体 MemorialDay、MemorialDayMapper（含 selectDistinctCoupleIds）、MemorialDayService/Impl、MemorialDayRedisCache、LovespaceMemorialProperties（lovespace.memorial）、MemorialDayController（/api/v1/memorial-days）、MemorialDayControllerExceptionHandler；LoveSpaceUserApplication 启用 @EnableScheduling（与既有定时任务一致）；前端：services/memorial.ts、memorialStore、Memorial.tsx、MemorialDayFormModal.tsx；轮询 /next 与列表、本地 countdownAnchor 实现秒级展示；工具类：MemorialCountdownCalculator 使用 MonthDay 显式 import，JavaDoc 与参数名小清理。

### 纪念日页 UI 重设计（任务 59，接续 55-58）

按 ui-ux-pro-max 原则（装饰用 SVG、非 emoji 图标、主题色与 `antdTheme` 一致）重设纪念日页：**简约浪漫 + 手绘感**（Caveat / 马善政字体，`index.css` memorial 工具类）；**布局**为顶区标题、**恋爱倒计时与祝福语**、**相册照片墙**（`listAlbums` + `listAlbumPhotos`）、底栏折叠「全部纪念日」列表；**爱心与星光**背景组件 `MemorialRomanticDecor`。**迭代**：页面 `**w-full`** 与顶栏内容区 `**max-w-6xl**` 一致；移除标题区「新建纪念日」与页内 **月历**；照片墙改为 **双环环绕、错落有致**；照片 **Image 预览**（缩略展示 / 原图预览）；倒计时整块 **上移至照片墙上方**；文案微调如照片墙副标题「回忆轻轻围成一圈」、预览 mask「查看」。新建仍以 **FloatButton** 为主。

### 登录注册与情侣邀请重构（任务 60）

**目标**：手机号作为登录账号；邮箱在个人信息页维护；用户名可修改；情侣绑定按手机号邀请。

**后端**：`User` 增加 `phone`；`RegisterRequest`/`LoginRequest` 改为手机号；`UserResponse` 含 `phone`；`JwtUtil` 签发 `CLAIM_PHONE`，`JwtUserPrincipal` 第三字段为 `phone`；`UpdateProfileRequest` 支持 `username`、`email`（资料内校验唯一与邮箱格式）；`CoupleInviteRequest` 改为 `inviteePhone`，`CoupleBindingServiceImpl.invite` 经 `UserService.findByNormalizedPhone` 解析被邀请方；`UserController`/`POST /users` 种子接口同步 phone；`sql/users.sql` 与 `users_add_phone_account.sql` 迁移。

**前端**：`Login`/`Register` 表单为手机号；`Profile` 展示只读手机号、可编辑用户名与邮箱（`updateProfile`）；`HomePage` 展示手机号与邮箱；`couple.ts` 与 `CoupleHome` 邀请走 `inviteePhone`。

**个人资料页小调**：头像上传后 `Avatar` 以 `user.avatarUrl` 为准，展示时经 **`resolveMediaUrl`**；`avatarUrl` 与 store 同步逻辑见当前 `Profile.tsx`。

### 恋爱问答 RAG、Milvus 与通义嵌入（接续）

**部署**：`docker-compose.yml` 含 **milvus-etcd**、**milvus**，对象存储与业务 **共用同一 MinIO**；`backend` 注入 `**MILVUS_HOST`**、`**SPRING_AI_DASHSCOPE_API_KEY**`、`**SPRING_AI_VECTORSTORE_MILVUS_***`、`**LOVESPACE_MILVUS_ENSURE_LOVE_KNOWLEDGE_SCHEMA**` 等；`env.example`、`memory/DEPLOYMENT.md` 已同步。

**嵌入**：`lovespace-ai` 中 `**DashScopeEmbeddingModel`** + `**DashScopeEmbeddingConfiguration**`（`lovespace.ai.embedding.provider=dashscope` 时注册），与 `**QwenProvider**` 共用 `**spring.ai.dashscope.api-key**`；`lovespace.ai.embedding.*` 与 Milvus `**embedding-dimension**` 对齐；**排除** `OpenAiEmbeddingAutoConfiguration`。

### 恋爱问答 RAG 与 Docker（2026-04 更新摘要）

- **移除** `lovespace.ai.rag.enabled`、`**RagMilvusAutoConfigurationExclusionEnvironmentPostProcessor`**、Maven Profile `**lovespace-rag**`；RAG 已并入 `**lovespace-ai**`，`**lovespace-user**` 仅依赖 `**lovespace-ai**`。
- **Bean**：去掉 RAG 链路上 `**@ConditionalOnBean`**；`**LoveQAController**` 恢复 `**@RestController**` 扫描；`**loveQAService**` 使用 `**@DependsOn("milvusSchemaService")**`。
- **Milvus**：`**lovespaceMilvusClient`** 与 Spring AI `**milvusClient**` 解名冲突；`**MilvusSchemaService**` 启动按 Spring AI 结构 **ensure** `**love_knowledge_base`**（`**lovespace.milvus.ensure-love-knowledge-schema**`，默认 true）。
- **文档**：`memory/love-qa-rag.md`、`DEPLOYMENT.md`、`decisions.md`、`PROJECT_STRUCTURE.md`、`INDEX.md`、`blockers.md`、`progress.md` 与根目录 `**docker-compose.yml`** / `**env.example**` 已按上述约定更新。

### 恋爱问答：多轮记忆修复、流式 SSE 与页面（2026-04）

- **问题与修复**：原先多轮历史挤在单条 **system** 中，通义侧易被忽略；改为 `**LLMProvider.chatWithSystemAndHistory`** 使用 **system + 多轮 user/assistant + 本轮 user**。历史会话 **Redis 过期**后续聊：在 `**LoveQAController`** 调 Facade 前 `**LoveQaConversationService.restoreRedisSessionIfMissing**` 从 **MySQL** 回填 Redis。
- **流式**：新增 `**POST /api/v1/ai/love-qa/chat/stream`**（`LoveQaStreamCallback`、`SseEmitter` 120s、虚拟线程）；保留 `**/chat**`。前端 `**AILoveQA**` 主路径 `**postLoveQaChatStream**` 解析 SSE 实时更新 assistant。
- **页面**：`/love-qa` **侧栏 + 主区**（玫瑰色与全站一致）；聊天卡片 **固定视口高度**、消息区 **内部滚动**、输入区贴底；`**useLayoutEffect`** 控制切换会话滚底。

### 私密消息 WebSocket 与情感洞察页（2026-04 会话）

- **现象**：`VITE_API_BASE_URL` 为空时，开发环境 WebSocket 连到 Vite 端口但原先未代理 `/ws`，提示「WebSocket 未连接」；情感洞察页打开时内容/请求像「跑两遍」。
- **修复（代码）**：`vite.config.ts` 增加 `**/ws` → 8081** 且 `**ws: true`**。`Chat.tsx`：WS 未就绪时 **HTTP** `POST /api/v1/messages/send` 与 `/scheduled`；WebSocket 依赖 `**partner?.id`**。`EmotionAnalysis.tsx`：`getEmotionReport` 传 **AbortSignal**，effect 清理时 **abort**，避免 StrictMode 重复请求与 loading 竞态；`EmotionAnalysisCharts.tsx`：**合并**饼图与折线图初始化为 **单个 `useEffect`**。
- **文档**：`memory/decisions.md`（Vite 代理含 `/ws`）、`blockers.md`（第 10 条更新）、本文件「当前总体状态」与「页面与组件摘要」已同步。

### RAG 优化实施（任务 61，2026-05）

**目标**：提升恋爱问答 RAG 模块的性能、可观测性与用户体验，降低运营成本。

**后端**：
- **Embedding 缓存**：`CachedEmbeddingModel` 装饰器包装 `DashScopeEmbeddingModel`，Redis 缓存 embedding 结果；`EmbeddingCacheEntry` 含向量、模型版本、哈希校验；`EmbeddingCacheStore` 负责读写；`EmbeddingCacheAutoConfiguration` 自动装配；配置 `lovespace.ai.embedding.cache.*`。
- **Latency 埋点**：`RagPhase` 枚举定义 6 个阶段；`RagTimer` 高精度计时（nanoTime）；`RagMetricsReport` 指标报告；`RagMetricsCollector` 收集器；集成到 `LoveQAService.chat()` 与 `chatStream()`。
- **检索可视化**：`LoveQaStreamCallback` 新增 `onRetrieved()` 回调；`RetrievedChunk` DTO；`LoveQAController` SSE 发送 `retrieved` 事件。
- **Prompt 压缩**：`DocumentDeduplicator` 相似度去重（Jaccard 0.85）；`PromptCompressor` 主入口（去重、长度限制 8000 字符、历史精简）；集成到 `LoveQAService.prepareChat()`。

**前端**：
- `services/loveQa.ts`：新增 `RetrievedChunk` 类型，`LoveQaChatStreamHandlers` 新增 `onRetrieved` 回调，SSE 解析处理 `retrieved` 事件。
- `AILoveQA.tsx`：消息类型扩展 `retrievedChunks`，`onSend` 中处理 `onRetrieved`；UI 展示引用摘要与来源卡片（**2026-06 补全**：可点击引用、卡片高亮、修复 build 阻断）。

**新增文件**（ lovespace-backend/lovespace-ai ）：
- `embedding/cache/CachedEmbeddingModel.java`
- `embedding/cache/EmbeddingCacheEntry.java`
- `embedding/cache/EmbeddingCacheKeyGenerator.java`
- `embedding/cache/EmbeddingCacheStore.java`
- `embedding/cache/EmbeddingCacheAutoConfiguration.java`
- `rag/metrics/RagPhase.java`
- `rag/metrics/RagTimer.java`
- `rag/metrics/RagMetricsReport.java`
- `rag/metrics/RagMetricsCollector.java`
- `rag/compress/DocumentDeduplicator.java`
- `rag/compress/PromptCompressor.java`
- `dto/RetrievedChunk.java`

**修改文件**：
- `LovespaceAiProperties.java`：新增 `Embedding.Cache` 配置类
- `LoveQAService.java`：集成压缩与埋点
- `LoveQaRagBeansConfiguration.java`：注入新 Bean
- `LoveQAController.java`：SSE 发送 `retrieved` 事件
- `LoveQaStreamCallback.java`：新增 `onRetrieved()`
- `application.yml`：新增 `lovespace.ai.embedding.cache.*` 配置
- `lovespace-ai/pom.xml`：添加 `commons-codec` 依赖
- `lovespace-frontend/src/services/loveQa.ts`：处理 `retrieved` 事件
- `lovespace-frontend/src/pages/AILoveQA.tsx`：展示引用卡片

**文档更新**：`memory/love-qa-rag.md`、`memory/decisions.md`、`memory/progress.md` 已同步。

### 前端代码审查与修复（任务 62，2026-06-02）

**背景**：全量检查 `lovespace-frontend` 发现 `npm run build` 失败（`AILoveQA.tsx` 未使用变量）、RAG 检索可视化半成品、头像未走 `resolveMediaUrl`、ESLint 6 errors + 大量 Prettier warning。

**修复**：

1. **AILoveQA 检索可视化补全**
   - assistant 消息上方：摘要「📚 参考了 X 条知识」+ 逐条来源卡片（`[n]`、source、textPreview）
   - 正文 `【n】`/`[n]` 可点击，滚动至 `id="source-{messageKey}-{idx}"` 并短暂高亮
   - 修复 TS/ESLint：`ingestMode` 类型守卫（去掉 `as any`）、引用正则

2. **头像 URL**
   - `AppLayout`、`Profile`、`CoupleCard`、`Chat`、`Inbox` 的 `Avatar` 统一 `resolveMediaUrl(avatarUrl)`

3. **结构与依赖清理**
   - 新增 `utils/mood.ts`（`MOOD_OPTIONS`、`moodLabel` 从 `MoodTag.tsx` 拆出，消除 Fast Refresh ESLint error）
   - 删除死代码 `stores/useAppStore.ts`、演示组件 `EChartsDemo.tsx`（首页已移除引用）
   - `package.json` 显式依赖 `dayjs`

4. **Lint 与 hooks**
   - `eslint . --fix` 修复 Prettier 格式
   - `Chat.tsx`：WebSocket effect 依赖改为 `partner`
   - `Memorial.tsx`：倒计时去掉多余 `useMemo`，`void tick` + 每帧 `remainingMsFromAnchor`

**验证**：`npx eslint .` → 0 errors / 0 warnings；`npm run build` → 通过。

**文档**：`memory/progress.md`（本段）、`memory/love-qa-rag.md`、`memory/PROJECT_STRUCTURE.md`、`memory/CODING_RULES.md`、`memory/RAG_OPTIMIZATION.md` 已同步。

### RAG 分阶段优化（2026-06-11 起）

- **计划文档**：memory/RAG_PHASED_OPTIMIZATION.md
- **当前**：阶段 2 召回 / **P2-C 混合检索已完成** → 下次阶段 3 韧性或 P2-D Query 增强
- **P2-C 已完成（混合检索）**：
  - MySQL FULLTEXT（ngram：`title` + `category` + `content`）+ `LoveQaKeywordRetriever` / `LoveQaDocumentKeywordMapper`
  - `RagReciprocalRankFusion`：向量 rank + BM25 rank RRF 融合（`hybrid-rrf-k`）
  - `RagRetrievalPipeline`：向量 candidate-k ∥ BM25 topK → RRF → 阈值 → dedupe → L1 rerank
  - 配置：`hybrid-retrieval-enabled`、`hybrid-bm25-top-k`；存量库执行 `love_qa_documents_add_fulltext.sql`
- **Sprint 4 已完成（P2-B + P2-E）**：
  - `RagRetrievalPipeline`：candidate-k → 阈值 → `RagDocumentIdDeduplicator` → `RagL1Reranker` → final topK
  - L1 rerank：向量分 + category 匹配加分 + ingestedAt 新近度加分（`rerank-*` 配置）
  - 历史引用快照：`love_qa_messages.retrieved_chunks_json` + `appendChatRound` 持久化 + API/前端还原卡片
  - 单元测试：`RagDocumentIdDeduplicatorTest`、`RagL1RerankerTest`
- **Sprint 3 已完成（P2-A）**：
  - `RagRetrievalFilterBuilder`：已绑定 `(coupleId == binding || scope == GLOBAL)`；未绑定且 `allow-global-ingest` 时 `scope == GLOBAL`；`allow-global-fallback: false` 时仅 coupleId 私有
  - `LoveQaChatRetrievalValidator`：chat 侧 coupleId 校验与 filter 解析（与入库 `LoveQaIngestValidator` 对齐）
  - `RagSimilarityFilter` + 配置 `retrieve-candidate-k` / `similarity-threshold` / `retrieve-top-k`：candidate 初召回 → 阈值过滤 → final topK
  - 0 命中拒答：不调用 LLM，直接返回标准拒答模板；SSE 不发 `retrieved`
  - 单元测试：`RagRetrievalFilterBuilderTest`（情侣 A/B 隔离表达式）、`RagSimilarityFilterTest`（阈值）
  - 前端：未绑定情侣禁用提问输入 + Alert 提示
- **Sprint 2 已完成**：P1-C/D + 前端知识库 Tab（见上文）
- **阻塞**：存量库需 `love_qa_documents.sql`、`love_qa_messages_add_retrieved_chunks_json.sql`；**混合检索**另需 `love_qa_documents_add_fulltext.sql`（ngram FULLTEXT）
- **下次**：阶段 3 韧性（限流 / 降级 / Embedding 缓存补全）或 P2-D Query rewrite

### 情侣行程规划 Agent（2026-06-16 起）

- **计划文档**：memory/TRAVEL_PLANNER_AGENT.md（P0–P3 任务表、DTO/API 契约、验收用例、附录）
- **当前**：**P0 未开始** — 后端已有 `TravelPlannerService` + `POST /api/v1/ai/travel/plan`（单次 LLM）；高德 `AmapPlacesClient` 为 NoOp；POI 向量检索返回空；无天气客户端；无前端 `/travel-planner`
- **推荐下一步**：P0-06/07 扩展 DTO/Schema → P1-01~08 天气 + 高德客户端（无 Key 可 Mock）
- **待拍板**：天气数据源（和风 vs 高德）、首期城市列表（文档默认杭州/成都/厦门/西安/大理）
- **阻塞**：高德 Web 服务 Key、和风天气 Key（无 Key 可用 Mock 跑通 P1 流程）