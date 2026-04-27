# 关键决策

## 架构与模块

1. 后端采用 Spring Boot 多模块 Maven；当前以 `lovespace-user` 为可启动服务
2. 公共能力在 `lovespace-common`（如 `ApiResponse`）
3. **`lovespace-ai` 模块**：承载 LLM 抽象（`LLMProvider`）、通义千问（`QwenProvider`，阿里云 DashScope Java SDK）、OpenAI（Spring AI `OpenAiChatModel` + `ChatClient`）、`LlmRouter`、`AiChatService` 与 `POST /api/v1/ai/chat`；**`DashScopeEmbeddingModel`** 实现 Spring AI **`EmbeddingModel`**（DashScope `TextEmbedding`，默认 `text-embedding-v2`，与聊天共用 `spring.ai.dashscope.api-key`），在 **`lovespace.ai.embedding.provider=dashscope`**（默认）且无其它 `EmbeddingModel` Bean 时由 **`DashScopeEmbeddingConfiguration`** 注册；配置 `lovespace.ai.embedding.*`；`application.yml` **排除** `OpenAiEmbeddingAutoConfiguration`，不依赖 OpenAI 官方嵌入自动配置。**RAG/Milvus**（`spring-ai-starter-vector-store-milvus`、Redis 多轮等）与 LLM **同在 `lovespace-ai`**，由 **`lovespace-user`** 依赖 **`lovespace-ai`** 打入 fat jar；**Milvus**（`spring-ai-starter-vector-store-milvus`）集合名由 **`spring.ai.vectorstore.milvus.collection-name`** 配置（默认与常量 **`love_knowledge_base`** 对齐）；业务侧原生客户端封装 Bean 名为 **`lovespaceMilvusClient`**（`com.meng.lovespace.ai.milvus.MilvusClient`），**避免**与 Spring AI 自动配置中 **`milvusClient`**（`MilvusServiceClient`）重名；**`MilvusSchemaService`** 在 **`lovespace.milvus.ensure-love-knowledge-schema`**（默认 true）时于启动按 Spring AI `MilvusVectorStore` 同款 schema 建集合并建索引、`loadCollection`；**不再**使用 `lovespace.ai.rag.enabled` 与 **`RagMilvusAutoConfigurationExclusionEnvironmentPostProcessor`**。可选第二集合 **`travel_poi_embeddings`**（`lovespace.milvus.ensure-travel-poi-schema` 骨架）。**恋爱 RAG**：**`LoveQAController`** 为 **`@RestController`** 组件扫描注册；**`LoveQAService`** 实现 **`LoveQaChatFacade`**，在 **`LoveQaRagBeansConfiguration`** 中声明 **`@Bean`**，且 **`@DependsOn("milvusSchemaService")`** 以保证集合就绪顺序；**`POST /api/v1/ai/love-qa/ingest|chat`** 依赖 **`VectorStore` + `EmbeddingModel` + Redis**。**情侣旅游规划**：`TravelPlannerService` + `POST /api/v1/ai/travel/plan`，输出 JSON（`TravelPlanJsonSchema`）。父 POM 引入 `spring-ai-bom`（与 Spring Boot 3.4.x 对齐）；官方 `spring-ai-bom` 不含 DashScope，通义侧不引入 `spring-ai-alibaba` 以免与 Boot 版本冲突
4. **实时消息**：在 REST 持久化基础上增加原生 WebSocket（非 STOMP），会话注册表 `ChatSessionRegistry` 按 `coupleId` 订阅后广播

## 版本与构建

1. Java 21（父 POM + Enforcer）
2. Spring Boot 3.4.x、Spring Cloud 2024.0.x（与 Spring AI 1.0 对齐；以父 `pom.xml` 为准）
3. `maven-compiler-plugin`：`<parameters>true</parameters>`，保留方法参数名供 Spring MVC 使用
4. Controller 层对易踩坑参数仍使用显式名称：`@RequestParam("coupleId")`、`@PathVariable("id")` 等，避免 IDE/非 Maven 编译链路缺参数名

## 数据库

1. 排序规则：`utf8mb4_general_ci`
2. **情侣绑定 `status`**：业务扩展 `0=待接受邀请`，文档与 `CoupleBindingStatus` 一致；接受后双方 ID 字典序规范化存储
3. 时间轴 **`images_json`**：存图片与视频 URL 的 JSON 数组字符串（同一字段；前端/展示按 URL 扩展名区分 mp4/webm/mov 与图片）；与头像共用 OSS/本地上传实现，`AvatarStorageService.uploadTimelineImage`（及分片后的 `uploadTimelineFromLocalFile`）使用 `timeline/` 路径前缀
4. **时间轴互动**：表 `love_record_likes`（`record_id`+`user_id` 唯一）、`love_record_comments`；脚本 `love_record_social.sql`；外键至 `love_records.id ON DELETE CASCADE`。`LoveRecordResponse` 含 `likeCount`、`commentCount`、`likedByMe`；`LoveRecordSocialService` 负责批量统计、点赞切换、评论分页与增删。赞/评权限与单条记录可见性、情侣成员校验一致（仅自己可见的记录对方不可赞评）。业务码示例：`40056` 空评论、`40455` 评论不存在、`40355` 仅作者删评
5. **相册**：表 `couple_albums`、`album_photos`；脚本见 `couple_albums.sql`
6. **相册照片列表**：服务端分页；DTO `AlbumPhotoPageResponse`（`total`、`page`、`pageSize`、`photos`）；与恋爱记录分页类似，默认 `pageSize=10`，上限 100
7. **相册大图分片登记**：分片合并得到 URL 后 `POST /api/v1/albums/{albumId}/photos/from-url`；`AlbumImageUrlValidator` 要求路径匹配 `albums/{yyyy-MM-dd}/{userId}/文件名`，防止盗用他人对象键
8. **私密消息**：表 `private_messages`；`message_type`：text|image|voice|letter；定时字段 `is_scheduled`/`scheduled_time`；脚本见 `private_messages.sql`
9. **共同计划**：表 `couple_plans`（`plan_type`：goal|travel|event；`status`/`progress`/预算等）、`plan_tasks`；脚本见 `couple_plans.sql`；`plan_tasks` 外键 `ON DELETE CASCADE` 至 `couple_plans`
10. **计划消费**：表 `plan_expenses`；`expense_type`：lodging（住宿）、transport（交通）、dining（用餐）、other（其他）；脚本见 `plan_expenses.sql`；`plan_id` 外键 `ON DELETE CASCADE` 至 `couple_plans`。接口返回的 `budgetSpent` 与 `expenseSummary.total` 一致，由 `plan_expenses` 汇总；创建/更新计划请求不再携带手工 `budgetSpent`；消费变更后回写 `couple_plans.budget_spent` 缓存列
11. **纪念日**：表 `memorial_days`（`couple_id` + `user_id` 创建人 + `memorial_date` 月日循环）；脚本 `memorial_days.sql`。业务与 `findActiveOrFrozenMembership` 校验 `coupleId` 一致。倒计时用 `java.time.MonthDay` 计算下一次公历出现与剩余毫秒（非当日到次日 00:00；当日到当日 `LocalTime.MAX`）
12. **用户账号**：表 `users` 以 **`phone`（登录手机号，UNIQUE）** 为登录主键字段；**`email` 可空**，在个人资料接口维护。注册不收集邮箱。手机号格式校验使用 `PhoneNormalizer`（中国大陆 `1[3-9]` 起共 11 位）。存量库迁移脚本：`users_add_phone_account.sql`

## 认证与安全

1. JWT（HS256）；**登录为手机号 + 密码**，签发 Claims 含 **`phone`**（`CLAIM_PHONE`）；`JwtUtil.getPhone(Claims)` 在无 `phone` 时可回退读旧版 **`email` claim** 以兼容升级前尚未过期的 token，新签发 token 仅含 `phone`。默认 Spring Security `SessionCreationPolicy.STATELESS`。可选 Redis 分布式 Session（`spring-session-data-redis`）：`lovespace.session.distributed.enabled=true` 且 `spring.session.store-type=redis` 时改为 `IF_REQUIRED`，登录将 `SecurityContext` 写入 Session（与 JWT 响应并存）；登出 `HttpSession.invalidate()` 并继续 JWT 黑名单
2. 登出：Redis 黑名单，TTL 对齐 token 剩余寿命
3. 公开路径：`/api/v1/auth/register|login|logout`、`/health/**`、Swagger、`/local-files/**`、`/ws/**`（WebSocket 握手不带标准 `Authorization` 头时仍需放行，由 Handler 内用 query `token` 校验）
4. `JwtAuthenticationFilter`：对 WebSocket 路径前缀 `/ws/` 不执行 Bearer 解析（避免无头时 401 阻断握手）。启用分布式 Session 时：Bearer 解析失败（过期等）不立即 401，继续链以便使用 Session 已恢复的认证；`jti` 黑名单仍 401（显式登出）
5. **私密消息 WebSocket**：连接 URL 携带 `?token=<JWT>`；连接后先发 `subscribe`（含 `coupleId`）再收发业务 JSON。分布式 Session 不替代 WS 的 query token，浏览器仍依赖登录返回的 JWT 存 localStorage 连接 WS
6. **CORS**：`DistributedSessionCorsConfig` 仅在 `lovespace.session.distributed.enabled=true` 时注册 `CorsConfigurationSource`（`allowCredentials(true)` + localhost/127.0.0.1 originPattern），供前端跨域直连 API 时携带 Session Cookie；同源 Vite 代理通常不需要

## 前端全局样式与登录态

1. **Ant Design 5 + Tailwind**：入口 `StyleProvider hashPriority="high"`（`@ant-design/cssinjs`），减轻工具类/全局样式与 `:where()` 低特异性冲突导致的浮层异常
2. `index.css`：`prefers-reduced-motion` 下过渡动画仅作用于 `#root` 内元素，避免误伤 portal 到 document.body 的浮层动画
3. `authStore`：`authHydrated` 表示是否完成 localStorage/Session 恢复；`AppLayout` 在 `authHydrated` 前不渲染子路由，避免 `Navigate → /login` 与 hydrate 异步竞态
4. **分布式 Session 前端**：环境变量 `VITE_SESSION_DISTRIBUTED=true` 时 axios `withCredentials: true`；修改后需重启 Vite。开发推荐 `VITE_API_BASE_URL` 留空走代理，减少跨域 Cookie 问题

## 纪念日页面（前端）

1. **版心**：与主导航一致，页面根容器使用 `w-full`，由 `AppLayout` 内容区 `mx-auto max-w-6xl` 统一限宽，避免纪念日单独 `max-w-4xl` 变窄
2. **照片墙数据源**：从相册 `listAlbums` + `listAlbumPhotos` 聚合近期照片；环绕式双环错落为前端 `memorialRingSlots` 纯布局，与纪念日 DDL 无直接字段关联
3. **预览**：相册缩略图列表展示，`antd` **`Image`** 开启 `preview`，预览 URL 优先 `imageUrl`（无则回退缩略图）
4. **信息架构**：「下一次心动 / 倒计时 / 祝福语」整块在**照片墙上方**；**页内不设月历**、标题区不设主「新建」按钮；新建以 **FloatButton** 为主，折叠「全部纪念日」仅列表与编辑/删除
5. **装饰与字体**：背景爱心/星光为 SVG（非 emoji）；`index.html` 可引入 Caveat、马善政等；`index.css` 中 `memorial-*` 动画在 `prefers-reduced-motion` 下关闭或减弱

## 业务规则摘录

1. **恋爱天数**：`ChronoUnit.DAYS.between(startDate, today) + 1`（含首日），业务日期 Asia/Shanghai
2. **时间轴可见性**：`1` 仅作者，`2` 情侣双方；列表/回忆按当前用户过滤
3. **时间轴列表日期筛选**：`startDate`/`endDate` 可选、闭区间；`startDate > endDate` 返回业务错误码 40054
4. **相册**：创建/列表/删相册、上传/列表/删照片、收藏均需交往或冻结情侣成员校验；删照片仅上传者；列出照片为分页查询（page/pageSize，默认 10）。改相册名：`PUT /api/v1/albums/{id}`，情侣成员均可。改照片元数据：`PUT /api/v1/albums/{albumId}/photos/{photoId}`，请求体 `PhotoUpdateRequest` 四字段整单替换（description、locationJson、takenDate、tagsJson），情侣任一方均可更新；`takenDate` 不可晚于业务日（40063）；空相册名 40065
5. **私密消息**：发送/列表/已读/撤回/定时创建均需有效情侣成员；撤回仅发送方，且发送后 2 分钟内（超时 40078）
6. **私密消息 WebSocket**：send/scheduled/read/retract 与 REST 共用 `MessageServiceImpl`，保证先落库再广播
7. **共同计划**：创建/列表/更新/删除计划及子任务 CRUD 均需交往或冻结情侣成员（`findActiveOrFrozenMembership`）；任务更新使用 `PlanTaskReplaceRequest`（整表替换语义：title、assigneeId、dueDate、completed）；负责人须为情侣双方之一或空
8. **计划消费**：列表/创建/更新/删除同计划下消费记录需情侣成员校验；创建使用 `PlanExpenseCreateRequest`；更新使用 `PlanExpenseReplaceRequest`（整单替换，与任务替换思路一致）；未找到消费业务码 40482
9. **情侣邀请**：`POST /couple/invite` body 为 **`inviteePhone`**（中国大陆手机号，与 `users.phone` 一致）；解析失败返回「用户不存在」类业务码；不可向自己的手机号发邀请

## 前后端联调

1. API 前缀 `/api/v1`
2. 前端 Vite `server.proxy`：`/api`、`/local-files`、**`/ws`（`ws: true`）** → 127.0.0.1:8081；`VITE_API_BASE_URL` 为空时页面同源，`/ws/chat` 经 Vite 转发至后端（与 `nginx.conf` 的 `/ws/` 一致）。生产环境仍依赖反向代理 **Upgrade / Connection**（见 `DEPLOYMENT.md`）
3. 前端情侣主键：使用 `GET /couple/info` 返回的 `bindingId` 作为时间轴、相册、私密消息、共同计划、AI 情感/情书的 `coupleId`
4. 前端静态资源：相对路径 `/local-files/**` 可通过 `VITE_API_BASE_URL` 与 `resolveMediaUrl` 拼完整 URL
5. **AI 长耗时接口**（情感分析、情书）：前端 HTTP timeout 建议 ≥120s；需配置 `spring.ai.dashscope.api-key`（及可选 `spring.ai.openai.api-key`）；路由 `lovespace.ai.provider`：qwen | openai

## AI 业务规则摘录

1. **通用对话**：`POST /api/v1/ai/chat`，由 `AiChatService` 按 `lovespace.ai.provider` 选择 `QwenProvider` 或 `OpenAiProvider`
2. **情感分析**：`GET /api/v1/ai/emotion`，`EmotionAnalysisService`；数据来自 `LoveRecordService.listVisibleRecordsInRange`（与时间轴一致可见性）；统计心情分布与日趋势 + 通义 JSON 解析 `overallMood`/`moodScore`/`insights`；默认区间、最长期间与业务码见 `EmotionAnalysisService` / `TimelineBusinessException`（`EmotionAnalysisController` 与时间轴共用 `UserGlobalExceptionHandler`）
3. **情书生成**：`POST /api/v1/ai/love-letter`，`LoveLetterService`；发送方/接收方/恋爱天数由服务端从当前用户与 `CoupleInfoResponse` 解析；`memories` 可选，缺省时用近一年可见记录摘要；风格 romantic/humorous/sincere，篇幅 short/medium/long；异常 `LoveLetterBusinessException`（`UserGlobalExceptionHandler`）
4. **恋爱知识库 RAG**：`POST /api/v1/ai/love-qa/ingest`（文本入库）、`POST /api/v1/ai/love-qa/chat`（**非流式**：检索 + LLM + **Redis 多轮记忆** + **MySQL 每轮持久化** `love_qa_conversations` / `love_qa_messages`，脚本 `love_qa.sql`）、**`POST /api/v1/ai/love-qa/chat/stream`（SSE 流式，语义与 `/chat` 一致，流结束后同样 `appendChatRound`）**；`GET .../love-qa/conversations`、`GET .../love-qa/conversations/{id}/messages` 供历史查看；`LoveQAController` 在 `lovespace-user`（需 JWT），仅在存在 `LoveQaChatFacade` Bean（RAG 启用且 `VectorStore` 等就绪）时注册；`LoveQAHistoryController` 不依赖 Milvus；会话键 `lovespace:love-qa:conv:{id}`，TTL 与最大轮数见 `lovespace.ai.rag.*`；**多轮上下文**由 **`LLMProvider.chatWithSystemAndHistory(Streaming)`** 以 **system（RAG 指令+检索片段）+ 多条 user/assistant + 本轮 user** 调用模型（避免把历史仅塞进超长 system）；**Redis 无会话但 MySQL 有历史**时 **`restoreRedisSessionIfMissing`** 在 `/chat` 与 `/chat/stream` 调用 Facade 前回填；向量维度与嵌入模型见 **`lovespace.ai.embedding.*`** 与 **`spring.ai.vectorstore.milvus.embedding-dimension`**（须一致）；部署需 **`SPRING_AI_DASHSCOPE_API_KEY`**（嵌入与通义聊天共用，RAG 场景下）；经 **nginx** 等代理 SSE 时注意 **`proxy_buffering off`** 等以免增量被缓冲
5. **情侣旅游规划**：`POST /api/v1/ai/travel/plan`，`TravelPlannerService` 输出结构化 JSON；可选 `lovespace.ai.travel.poi-vector-search-enabled` 与 POI 向量骨架；高德 `AmapPlacesClient` 当前为占位实现

## 异常与响应

1. **user 控制器统一处理**：`UserGlobalExceptionHandler`（`basePackages = com.meng.lovespace.user.controller`）：所有继承 `lovespace-common` 的 `ApiBusinessException` 的领域异常 → `ApiResponse.error(code, message)` + `warn` 日志；`@RequestBody` 校验失败 → `MethodArgumentNotValidException` → HTTP 400 + code `40001`；方法级 `@Validated` 约束 → `ConstraintViolationException` → HTTP 400 + code `40001`
2. **领域异常类型**（均继承 `ApiBusinessException`）：`CoupleBindingBusinessException`、`TimelineBusinessException`、`AlbumBusinessException`、`MessageBusinessException`、`PlanBusinessException`、`MediaChunkBusinessException`、`LoveLetterBusinessException`、`MemorialDayBusinessException`、`LoveQaBusinessException`；纪念日示例码 40490 未找到、40390 非成员、40091 名称为空
3. 未绑定情侣查询 info：HTTP 200 + code=40442（前端常量 `COUPLE_NOT_BOUND_CODE`）
4. **lovespace-ai 控制器**：`AiFeaturesExceptionHandler`（限定 `TravelPlannerController`、`AiChatController`）；`IllegalStateException` → HTTP 503 + code `50301`；JSON 解析失败 → HTTP 502 + code `50201`
5. **恋爱 RAG 会话**（仍在 `UserGlobalExceptionHandler`）：会话不存在或过期 → HTTP 404 + code `40491`；越权或 coupleId 不一致 → HTTP 403 + code `40391`

## 纪念日 API 与缓存

1. 路径前缀 `/api/v1/memorial-days`：CRUD；`GET .../next?coupleId=` 返回最近一个纪念日及 `millisecondsUntilNext`；`GET .../upcoming?coupleId=` 返回配置窗口内（默认 7 天）列表；可选 `useCache` 查询参数
2. **Redis**（`StringRedisTemplate` + JSON）：`lovespace:memorial:next:{coupleId}`、`lovespace:memorial:upcoming:{coupleId}`；TTL 与窗口见 `lovespace.memorial`（`cache-ttl-seconds`、`upcoming-window-days`、`zone-id`）。写操作 `evictCouple`；定时 `MemorialDayCacheRefreshTask` 预热 DISTINCT `couple_id`
3. 前端若恢复 `/memorial`：`FloatButton` 新建、`Calendar` 以展示为主（`headerRender` 含「今天」）、有纪念日日期圆形高亮、`memorialStore` 轮询 + 本地锚点秒级倒计时

## 本地静态访问（`/local-files/**`）

1. 使用 `LocalFileRangeController` 提供 `GET /local-files/{*filepath}`，显式支持 `Accept-Ranges: bytes` 与单段 `Range` 的 `206 Partial Content`（视频进度条友好）；不再仅用 `ResourceHandlerRegistry` 映射 `file:`（避免 Range 行为不一致）
2. 分片暂存目录：`_pending/media/{uuid}/`；定时清理超时会话；兼容读取旧路径 `_pending/timeline/{uuid}/` 以完成升级前未结束的会话

## 公共分片上传（`/api/v1/media/uploads`）

1. `ChunkedMediaUploadService`：`init` 中 `target`=TIMELINE|ALBUM；ALBUM 须带 `albumId` 且校验相册与情侣成员；分片 PUT 须带正确 `Content-Length`
2. 合并后：TIMELINE → `uploadTimelineFromLocalFile`；ALBUM → `uploadAlbumFromLocalFile`（与直传同一对象键规则）
3. 配置项在 `lovespace.timeline-upload`：`chunk-size-bytes`、`pending-session-hours`，以及图/视频扩展名与 `image-max-size-bytes` / `video-max-size-bytes`；相册分片仅允许 `lovespace.avatar.allowed-extensions` 中的图片，大小上限与时间轴图片上限一致
4. **破坏性变更**：原 `/api/v1/timeline/uploads` 已移除，统一为 `/api/v1/media/uploads`

## 上传与 Multipart

1. **头像**：仍受 `lovespace.avatar.max-size-bytes` 与扩展名配置约束
2. **时间轴直传 `POST /timeline/upload`**：校验见 `TimelineUploadProperties`（与头像配置独立）；支持图 + 视频扩展名
3. **相册 multipart `POST .../photos`**：不在 Controller 校验文件大小；仅校验非空与扩展名（与头像扩展名白名单一致）
4. **Spring Multipart**：`spring.servlet.multipart.max-file-size` / `max-request-size` 设为 10GB；生产若经 Nginx 等，需同步 `client_max_body_size`；大文件分片 PUT 亦需注意代理对 `Content-Length` 的保留

## 前端视觉

1. **主题**：暖色（玫瑰主色、页面背景 `#fff1f2`）；字体 Inter；Ant Design token 集中在 `src/theme/antdTheme.ts`
2. **相册网格**：缩略图统一 1:1 裁切展示
3. **相册列表改名**：不使用卡片角标按钮与相册内页「编辑名称」；仅封面点击进入相册；双击相册名称内联编辑，回车保存、Esc 取消（名称 title 提示「双击编辑名称」）。灯箱 `PhotoViewer` 提供编辑信息 Drawer（描述、拍摄日、地点、标签），zIndex 高于灯箱
4. **时间轴评论列表**：列表刷新导致 `record.commentCount` 变化且评论区展开时，须重拉评论（`TimelineRecordSocial` 依赖 `record.commentCount` 等，避免条数变而列表陈旧）
5. **列表卡片操作**：时间轴记录 `TimelineItem`、共同计划 `TaskItem` 与侧栏 `PlanListItem` 采用右上角 ⋯（MoreOutlined）+ Dropdown；编辑/删除与时间轴一致；删除前 `Modal.confirm`（任务/计划文案区分）。详情区 `PlanDetailHero`：主按钮「编辑」+ 圆形菜单仅「删除计划」，避免与侧栏重复同一套大卡片

## 部署与对象存储（摘要）

1. **容器编排**：根目录 `docker-compose.yml` 管理 MySQL、Redis、MinIO、backend、frontend；`env.example` 复制为 `.env`；Compose 变量默认值语法 `${VAR:-默认值}`（`:-`）
2. **可执行 jar**：仅 `lovespace-user` 模块产出 Spring Boot fat jar；父 POM `spring-boot-maven-plugin` 须带版本；详见 `DEPLOYMENT.md`
3. **MinIO 返回 URL**：`lovespace.minio.public-base-url` 若设置，`MinioAvatarStorageService.buildUrl` 为 `{publicBaseUrl}/{bucket}/{objectKey}`；未设置时用 `http://{endpoint主机}/{bucket}/{objectKey}`。生产应对浏览器暴露 `LOVESPACE_MINIO_PUBLIC_BASE_URL`；匿名读对象需桶策略，否则浏览器 403

## 工程规范

1. 代码风格、注释与日志要求见 `CODING_RULES.md`，后续开发默认遵守
2. 部署、MinIO、Nginx 反代与故障排查的完整说明见 `DEPLOYMENT.md`
