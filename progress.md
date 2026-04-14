# 实时进度

## 当前总体状态

- **核心功能已落地**：用户认证、个人资料（含头像上传）、情侣绑定、恋爱时间轴（含图片与视频 URL、区间筛选、作者编辑/删除、分页、可见性表单；直传或小文件 multipart，大文件公共分片；点赞与评论互动）、情侣相册（含相册内照片分页；大图分片 + from-url 登记；相册改名、照片元数据编辑）、私密消息（REST + WebSocket + 定时）、共同计划（计划 + 子任务 CRUD、计划消费流水 CRUD、预算按消费汇总、进度同步）均已落地并可联调
- **账号与登录（手机号）**：登录/注册使用 **中国大陆 11 位手机号**（`PhoneNormalizer` + `^1[3-9]\d{9}$`）；**邮箱**不在注册环节收集，在 **`PUT /api/v1/user/profile`** 中可选填或清空；**用户名**展示名可在资料页修改（唯一性校验）。**情侣邀请**：`POST /api/v1/couple/invite` body 为 **`inviteePhone`**（按手机号查找用户，不再使用 `inviteeUserId`）。JWT 含 `phone` claim（`JwtUtil.CLAIM_PHONE`）；`getPhone(claims)` 可回退读旧 token 的 `email` claim 以兼容过渡期。存量库执行 `sql/users_add_phone_account.sql`；新库用 `users.sql`（含 `phone` UNIQUE、`email` 可空）
- **纪念日倒计时**：后端 `/api/v1/memorial-days`（CRUD、`/next` 最近倒计时、`/upcoming` 近期窗口）、表 `memorial_days`、Redis 缓存 `lovespace:memorial:next:{coupleId}` / `upcoming:{coupleId}`、配置 `lovespace.memorial.*`、定时 `MemorialDayCacheRefreshTask`；`MemorialCountdownCalculator` 用 `MonthDay` 做每年循环与剩余毫秒。前端：`Memorial.tsx` + `memorialStore` + `services/memorial.ts`；路由 `/memorial` 已在 `App.tsx` 注册，顶栏有「纪念日」入口。页面为简约浪漫 + 手绘感（玫瑰主色一致）：`MemorialRomanticDecor`（SVG 爱心/星光）、`MemorialPhotoWall`（从相册拉取照片、双环错落环绕布局、中心柔光；无照片时占位相框）；**恋爱倒计时与祝福语区块在照片墙上方**；照片用 Ant Design **`Image` + `preview`**（列表用缩略图、预览优先原图 `imageUrl`）；**新建**主要靠右下角 FloatButton，折叠面板「全部纪念日」仅列表编辑/删除（**页内已移除月历与标题区新建按钮**）；内容区 **`w-full`** 与 `AppLayout` 的 `max-w-6xl` 对齐。字体：`index.html` 引入 Caveat、马善政；`index.css` 含 `memorial-display` / `memorial-accent` / `memorial-polaroid` 与星光动画（`prefers-reduced-motion` 下减弱）
- **AI 能力**（`lovespace-ai` + `lovespace-user`）：Spring AI 1.0（`spring-ai-openai`）+ 通义千问（阿里云 DashScope Java SDK）；`LLMProvider`（`QwenProvider` / `OpenAiProvider`）与 `lovespace.ai.provider` 路由；`POST /api/v1/ai/chat` 通用对话；情感分析（`EmotionAnalysisService` + `GET /api/v1/ai/emotion`，统计 + 通义解读）；情书生成（`LoveLetterService` + `POST /api/v1/ai/love-letter`）。父工程 Spring Boot 3.4.x + `spring-ai-bom` 以配套 Spring AI
- 接口前缀统一 `/api/v1`；Vite 开发代理 `/api`、`/local-files` → 后端 8081
- 父 POM 已启用 `maven-compiler-plugin parameters=true`，且 Controller 中关键 `@RequestParam`/`@PathVariable` 已写显式名称，避免 Spring 绑定报错
- 前端已统一暖色主题（玫瑰主色 + 浅粉背景，Inter 字体），相册照片列表为等尺寸网格（aspect-square + object-cover）
- **相册上传**：Controller 层不再校验单文件大小；`spring.servlet.multipart` 放宽至 10GB（实际仍受 Tomcat、反向代理、磁盘限制）。大图（前端默认大于 4MB）走 `/api/v1/media/uploads` 分片，完成后 `POST /api/v1/albums/{id}/photos/from-url` 落库
- **本机静态资源**：`/local-files/**` 由 `LocalFileRangeController` 提供，支持 Accept-Ranges 与 206 Partial Content（利于视频拖拽进度）；已移除仅 ResourceHandlerRegistry 的 file 映射
- **公共分片上传**：`ChunkedMediaUploadService` + `/api/v1/media/uploads`（init / PUT .../chunks/{i} / status / complete / DELETE）；暂存 `_pending/media/{uuid}/`，定时任务清理超时会话；兼容旧路径 `_pending/timeline/` 的未完成会话
- **私密消息**：Spring Security 放行 `/ws/**` 握手；JWT 在 JwtAuthenticationFilter 中对 WebSocket 路径跳过（避免无 Authorization 头导致握手失败）；浏览器端通过 `?token=` 传 JWT，握手后在 ChatWebSocketHandler 内校验
- **聊天页前端**：当前关闭轮询，以 WebSocket 实时增量为主；历史首屏仍用 GET /api/v1/messages；即时发送与定时创建均走 WS JSON（send / scheduled）
- **可选分布式 Session（Redis）**：`spring-session-data-redis`；`lovespace.session.distributed.enabled` 默认 false；启用时需 `spring.session.store-type=redis` 与可用 Redis。登录写入 Spring Security 上下文至 Session（AuthController + HttpSessionSecurityContextRepository），登出 invalidate Session。JwtAuthenticationFilter：Bearer 过期/无效时若已启用分布式 Session则不直接 401，以便回退 Session Cookie；黑名单仍 401。DistributedSessionCorsConfig：启用分布式 Session 时注册带凭证 CORS（直连 API 跨域时需要）。Cookie 名 LOVESPACE_SESSION
- **前端认证**：authStore 增加 authHydrated；AppLayout 在 hydration 完成前不渲染 Outlet（避免刷新时 isAuthed 尚未从 localStorage 恢复即触发 Navigate → /login）。VITE_SESSION_DISTRIBUTED=true 时 http withCredentials；hydrate 无 JWT 时可 getProfile 依赖 Session
- **前端全局样式**：@ant-design/cssinjs StyleProvider hashPriority="high"（与 Tailwind 共存）；index.css 将 prefers-reduced-motion 限定在 #root *，避免破坏挂到 body 的 Ant Design 浮层；ConfigProvider getPopupContainer={() => document.body}；主题 zIndexPopupBase: 1050
- **页面体验**：情侣首页 Hero、快捷入口、解除关系弱化；AI 情书去掉与按钮重复的 Alert；AppLayout flex 布局 + Footer mt-auto 使页脚贴视口底（内容短时）
- **情侣邀请与「消息」**：GET/POST /api/v1/couple/pending-invites、GET .../pending-invites/count；被邀请方在 /inbox 列表一键接受，无需手输绑定码；AppLayout 下拉消息列表 + inboxStore 角标；CoupleHome 以「查看消息」为主流程
- **容器部署**：根目录 docker-compose.yml（MySQL / Redis / MinIO / backend / frontend）、双 Dockerfile、env.example、前端 nginx.conf 反代 /api、/local-files、/ws；说明见 DEPLOYMENT.md
- **Maven 可执行包**：父 POM pluginManagement 为 spring-boot-maven-plugin 指定 ${spring-boot.version}；lovespace-user 显式 repackage 与 mainClass，避免 fat jar 无 Main-Class
- **MinIO 访问 URL**：lovespace.minio.public-base-url 若配置，MinioAvatarStorageService 返回 {publicBaseUrl}/{bucket}/{objectKey}；未配置时用 http://{endpoint主机}/{bucket}/{objectKey}，生产应对外填写 LOVESPACE_MINIO_PUBLIC_BASE_URL；浏览器直链对象需桶匿名读策略，否则 403

---

## 后端（`lovespace-backend`）

### 模块

| 模块 | 说明 |
|------|------|
| `lovespace-common` | 统一响应 ApiResponse 等 |
| **`lovespace-ai`** | LLM 抽象（LLMProvider）、QwenProvider（DashScope）、OpenAiProvider（Spring AI OpenAiChatModel）、AiChatService、AiChatController（POST /api/v1/ai/chat） |
| `lovespace-user` | 可运行用户服务（端口默认 8081）；依赖 lovespace-ai；扫描 com.meng.lovespace.ai 与 AI 相关 @ConfigurationPropertiesScan |

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
- / 首页、/profile 个人资料、/couple 情侣首页、/inbox 消息（待处理情侣邀请）、/timeline 时间轴、/album 相册、/chat 私密消息、/plan 共同计划、**/memorial 纪念日**、/emotion 情感洞察、/love-letter AI 情书
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
- services/emotion.ts — getEmotionReport（GET /api/v1/ai/emotion，超时 120s）
- services/loveLetter.ts — generateLoveLetter（POST /api/v1/ai/love-letter，超时 120s）
- services/memorial.ts — 纪念日 CRUD、getNextMemorial、listUpcomingMemorials（与后端 /api/v1/memorial-days 对齐）
- memorialStore（Zustand）— 列表、nextPayload、countdownAnchor（轮询与本地秒级倒计时对齐）、invalidate

### UI 与主题

- src/theme/antdTheme.ts + App.tsx ConfigProvider：locale=zhCN，暖色 token
- index.css：ls-surface、ls-page-intro、ls-link；Inter 字体
- 顶栏：AppLayout 含「私密消息」「共同计划」「**纪念日**」「情感洞察」「AI情书」等入口

### 页面与组件摘要

- **时间轴**：Timeline.tsx（工具栏日期筛选、Pagination 每页 10 条、无顶部大标题 Card；赞/评变更后 loadPage 刷新当前页）；TimelineForm（谁可见默认双方；mediaChunkUpload 大图/视频分片 + 断点续传）；TimelineItem（图片 + video 展示；作者右上角 ⋯；底部 TimelineRecordSocial：心形赞、评论展开、分页加载、发送/删除自己的评论；列表刷新时依赖 record.commentCount 重拉评论避免列表与条数不一致）
- **共同计划**：Plan.tsx — 主从布局（左侧 PlanListItem 仅摘要：标题、类型、短日期、微型进度；右侧 PlanDetailHero 完整详情、统计卡、预算与消费、进度条）；PlanExpensePanel（消费明细表、记一笔 Modal）；PlanForm（无手工「已花费」）；TaskFormModal、TaskItem、ProgressBar、BudgetTracker；任务变更后 updatePlan({ progress }) 与任务完成比例对齐（computeTaskProgressPercent）。已移除旧版 PlanCard（由侧栏行 + 详情头图替代）
- **情侣**：CoupleHome（邀请弹窗输入对方 **手机号**；Hero、「一起去看看」快捷链、底部弱化解除关系）、CoupleCard、DaysCounter
- **相册**：Album.tsx（uploadAlbumPhotoAuto；saveAlbumName；进入相册后 Pagination（PHOTO_PAGE_SIZE，当前前端常量多为 12；后端单页上限仍为 100）；上传成功后回到第 1 页并刷新）；AlbumCard（仅封面进入；双击名称内联编辑、回车保存、Esc 取消）；PhotoGrid、PhotoViewer（灯箱内编辑信息 Drawer，元数据保存后 patchPhotoMeta）、UploadButton
- **私密消息**：Chat.tsx；MessageBubble、MessageInput、ScheduledPicker（一体化输入区、会话时间格式、滚动加载更早消息）
- **工具**：utils/mediaUrl.ts
- **情感洞察**：EmotionAnalysis.tsx + EmotionAnalysisCharts.tsx（ECharts 环形图 + 折线面积图）；日期区间、仪表盘分数、AI 建议、分布列表
- **AI 情书**：AILoveLetter.tsx — 风格/篇幅/可选回忆、生成结果与复制；表单下仅保留简短耗时提示（已去与按钮重复的 Alert）
- **纪念日**：`Memorial.tsx` + `MemorialDayFormModal.tsx` + `components/memorial/MemorialRomanticDecor.tsx` + `MemorialPhotoWall.tsx` — 标题区（手绘标题 + 波浪线文案）；**倒计时与祝福语在照片墙之上**；照片墙从相册聚合、环绕式双环错落、**点击照片 Ant Design Image 预览放大**（mask 文案「查看」）；FloatButton 新建；折叠「全部纪念日」列表（编辑/删除）；无页内月历与标题下「新建纪念日」主按钮

### 验证

- npm run build 已通过（以当前依赖为准）

---

## 对话任务摘要（按时间线合并）

### 历史基线（任务 1-6）

JavaDoc 与后端多文件注释规范；情侣绑定全链路；情侣首页；恋爱时间轴；图片与 images_json、POST /timeline/upload；Spring 参数名 -parameters + 显式 @RequestParam/@PathVariable；时间轴日期区间后端 startDate/endDate，前端 RangePicker；情侣相册前后端（表、实体、API、相册页、缩略图网格、multipart 配置）；暖色主题与 CODING_RULES.md。

### 私密消息与聊天（任务 7-12）

private_messages 表、PrivateMessage、MessageMapper/Service/Controller、定时任务、消息业务码与 MessageControllerExceptionHandler；前端 Chat.tsx + 气泡/输入/定时选择；路由 /chat 与导航；撤回 2 分钟限制（前后端）；WebSocket：spring-boot-starter-websocket、WebSocketConfig、ChatWebSocketHandler、ChatSessionRegistry；Security 放行 /ws/**；JWT Filter 跳过 /ws/；握手后 subscribe + 业务消息；落库后广播；前端聊天轮询关闭，WS 发送（含定时 scheduled）；Security 修复解决「WebSocket 未连接」；聊天 UI：会话下边线、时间文案（当天/昨天/星期）、自适应高度、底部最新消息、输入区与发送按钮一体化。

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

新增模块 lovespace-ai：依赖 spring-ai-bom 1.0.0、spring-ai-starter-model-openai；通义千问使用 com.alibaba:dashscope-sdk-java（官方 spring-ai-bom 不含 DashScope）。QwenProvider（chat / chatWithSystem）、OpenAiProvider（ChatClient + OpenAiChatModel）、LovespaceAiProperties（lovespace.ai.provider）。父 POM：Spring Boot 3.4.5、Spring Cloud 2024.0.0、lovespace-user 依赖 lovespace-ai；application.yml：spring.ai.dashscope.api-key、spring.ai.model、spring.ai.openai.*、lovespace.ai.provider；POST /api/v1/ai/chat：AiChatController + AiChatService；LoveRecordService.listVisibleRecordsInRange；EmotionAnalysisService + EmotionAnalysisController GET /api/v1/ai/emotion；DTO EmotionAnalysisReport、EmotionTrendPoint；TimelineControllerExceptionHandler 增加 EmotionAnalysisController；前端 /emotion、services/emotion.ts、EmotionAnalysis.tsx、EmotionAnalysisCharts.tsx；顶栏「情感洞察」。

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

按 ui-ux-pro-max 原则（装饰用 SVG、非 emoji 图标、主题色与 `antdTheme` 一致）重设纪念日页：**简约浪漫 + 手绘感**（Caveat / 马善政字体，`index.css` memorial 工具类）；**布局**为顶区标题、**恋爱倒计时与祝福语**、**相册照片墙**（`listAlbums` + `listAlbumPhotos`）、底栏折叠「全部纪念日」列表；**爱心与星光**背景组件 `MemorialRomanticDecor`。**迭代**：页面 **`w-full`** 与顶栏内容区 **`max-w-6xl`** 一致；移除标题区「新建纪念日」与页内 **月历**；照片墙改为 **双环环绕、错落有致**；照片 **Image 预览**（缩略展示 / 原图预览）；倒计时整块 **上移至照片墙上方**；文案微调如照片墙副标题「回忆轻轻围成一圈」、预览 mask「查看」。新建仍以 **FloatButton** 为主。

### 登录注册与情侣邀请重构（任务 60）

**目标**：手机号作为登录账号；邮箱在个人信息页维护；用户名可修改；情侣绑定按手机号邀请。

**后端**：`User` 增加 `phone`；`RegisterRequest`/`LoginRequest` 改为手机号；`UserResponse` 含 `phone`；`JwtUtil` 签发 `CLAIM_PHONE`，`JwtUserPrincipal` 第三字段为 `phone`；`UpdateProfileRequest` 支持 `username`、`email`（资料内校验唯一与邮箱格式）；`CoupleInviteRequest` 改为 `inviteePhone`，`CoupleBindingServiceImpl.invite` 经 `UserService.findByNormalizedPhone` 解析被邀请方；`UserController`/`POST /users` 种子接口同步 phone；`sql/users.sql` 与 `users_add_phone_account.sql` 迁移。

**前端**：`Login`/`Register` 表单为手机号；`Profile` 展示只读手机号、可编辑用户名与邮箱（`updateProfile`）；`HomePage` 展示手机号与邮箱；`couple.ts` 与 `CoupleHome` 邀请走 `inviteePhone`。

**个人资料页小调**：头像上传后 `Avatar` 以 `user.avatarUrl` 为准；`avatarUrl` 与 store 同步逻辑见当前 `Profile.tsx`（用户可继续迭代）。
