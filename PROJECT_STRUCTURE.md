# LoveSpace 项目结构说明

仓库根目录：`meng-lovespace`（示例路径 `d:\Project\meng-lovespace`）。

```
meng-lovespace/
├── docker-compose.yml               # MySQL / Redis / MinIO / backend / frontend 编排
├── env.example                      # 部署环境变量模板（复制为 .env）
├── memory/                          # 会话衔接：目标、进度、决策、障碍、结构、编码规则、部署
│   ├── goals.md
│   ├── progress.md
│   ├── decisions.md
│   ├── blockers.md
│   ├── PROJECT_STRUCTURE.md         # 本文件
│   ├── CODING_RULES.md              # 编码规则（风格、注释、日志）
│   └── DEPLOYMENT.md                # 容器部署与 MinIO/403 说明
│
├── lovespace-backend/               # 后端（Maven 多模块）
│   ├── Dockerfile                   # 可执行 jar → 镜像（target/*.jar 或 app.jar）
│   ├── pom.xml                      # 父 POM：Java 21、Spring Boot 3.4.x、spring-ai-bom、compiler parameters
│   ├── lovespace-common/            # 公共模块
│   │   └── src/main/java/com/meng/lovespace/common/
│   │       └── web/ApiResponse.java
│   ├── lovespace-ai/                # AI：LLM 抽象、通义千问 / OpenAI、通用对话 Controller
│   │   └── src/main/java/com/meng/lovespace/ai/
│   │       ├── config/LovespaceAiProperties.java
│   │       ├── controller/AiChatController.java
│   │       ├── dto/AiChatRequest、AiChatResponseData
│   │       ├── provider/LLMProvider、QwenProvider、OpenAiProvider
│   │       └── service/AiChatService.java
│   └── lovespace-user/              # 用户服务（可执行 Spring Boot）
│       ├── pom.xml                  # 依赖 lovespace-ai；含 websocket、spring-session-data-redis（可选分布式 Session）
│       └── src/main/
│           ├── java/com/meng/lovespace/user/
│           │   ├── LoveSpaceUserApplication.java   # 扫描 com.meng.lovespace.ai；@EnableScheduling
│           │   ├── config/          # Security、JWT、**LovespaceSessionProperties**、**DistributedSessionCorsConfig**（条件）、MyBatis-Plus、OSS/本地存储、Avatar/TimelineUpload 等
│           │   ├── controller/      # …、EmotionAnalysisController、LoveLetterController、MemorialDayController（含 /api/v1/ai 与 /api/v1/memorial-days）
│           │   ├── couple/
│           │   ├── timeline/
│           │   ├── upload/            # MediaChunkTarget、MediaUploadSessionMeta、AlbumImageUrlValidator
│           │   ├── dto/             # 含 Album*、LoveRecord*、Plan*；**EmotionAnalysisReport**、**LoveLetterGenerateRequest** 等
│           │   ├── entity/          # User、CoupleBinding、LoveRecord、LoveRecordLike、LoveRecordComment、Album、Photo、PrivateMessage、Plan、PlanTask、PlanExpense、MemorialDay
│           │   ├── exception/     # Couple*、Timeline*、Album*、Message*、Plan*、MediaChunkBusinessException、MemorialDayBusinessException
│           │   ├── mapper/
│           │   ├── oss/             # AvatarStorageService（avatar / timeline / album；含 FromLocalFile 合并发布）
│           │   ├── security/
│           │   ├── service/ + service/impl/   # 含 LoveRecordSocialService；**EmotionAnalysisService**、**LoveLetterService**、**MemorialDayService**、**MemorialDayRedisCache**
│           │   ├── task/            # ScheduledMessageDispatchTask、MediaChunkCleanupTask、MemorialDayCacheRefreshTask
│           │   ├── websocket/       # WebSocketConfig、ChatWebSocketHandler、ChatSessionRegistry
│           │   └── util/            # MemorialCountdownCalculator 等
│           └── resources/
│               ├── application.yml
│               └── sql/             # users、couple_binding、love_records、images_json 增量、love_record_social、couple_albums、private_messages、couple_plans、plan_expenses、memorial_days
└── lovespace-frontend/              # 前端（Vite + React + TS）
    ├── Dockerfile                   # nginx 提供 dist + nginx.conf
    ├── nginx.conf                   # 生产：反代 /api、/local-files、/ws → backend
    ├── package.json
    ├── vite.config.ts               # 代理 /api、/local-files → 8081（ws 通常直连 8081）
    ├── index.html
    └── src/
        ├── main.tsx                 # StyleProvider hashPriority；BrowserRouter；antd reset + index.css
        ├── App.tsx                  # 路由含 /inbox、/chat、/plan、/emotion、/love-letter（纪念日 /memorial 可按需注册）
        ├── theme/antdTheme.ts
        ├── index.css
        ├── layouts/AppLayout.tsx    # 导航；authHydrated 前加载态、Outlet；flex 页脚贴底
        ├── pages/
        │   ├── HomePage.tsx
        │   ├── Login.tsx、Register.tsx
        │   ├── Profile.tsx
        │   ├── CoupleHome.tsx
        │   ├── Inbox.tsx            # 待处理情侣邀请（消息）
        │   ├── Timeline.tsx
        │   ├── Album.tsx
        │   ├── Chat.tsx
        │   ├── Plan.tsx
        │   ├── EmotionAnalysis.tsx
        │   ├── AILoveLetter.tsx
        │   ├── Memorial.tsx         # 纪念日页（当前默认未挂路由，见 progress.md）
        │   └── NotFoundPage.tsx
        ├── components/
        │   ├── AuthPageShell.tsx
        │   ├── CoupleCard.tsx、DaysCounter.tsx
        │   ├── MoodTag.tsx
        │   ├── TimelineItem.tsx、TimelineForm.tsx、TimelineRecordSocial.tsx
        │   ├── MessageBubble.tsx、MessageInput.tsx、ScheduledPicker.tsx
        │   ├── AlbumCard.tsx、PhotoGrid.tsx、PhotoViewer.tsx、UploadButton.tsx
        │   ├── PlanListItem.tsx、PlanDetailHero.tsx、PlanForm.tsx、PlanExpensePanel.tsx、TaskFormModal.tsx、TaskItem.tsx、ProgressBar.tsx、BudgetTracker.tsx
        │   ├── EmotionAnalysisCharts.tsx
        │   ├── MemorialDayFormModal.tsx
        ├── services/
        │   ├── http.ts
        │   ├── auth.ts、couple.ts（含 pending-invites）、timeline.ts、mediaChunkUpload.ts、album.ts、plan.ts、emotion.ts、loveLetter.ts、memorial.ts
        ├── stores/                  # authStore、inboxStore、memorialStore；可选 Session 与 VITE_SESSION_DISTRIBUTED
        ├── utils/mediaUrl.ts、timelineMedia.ts、planProgress.ts
        └── ...
```

## 后端 API 速查（前缀 `/api/v1`）

| 区域 | 路径模式 | 说明 |
|------|----------|------|
| 认证 | `/auth/register` `POST`、`/auth/login` `POST`、`/auth/logout` `POST` | 公开 |
| 资料 | `/user/profile` `GET` `PUT`；`/user/profile/avatar` `POST` | 需登录 |
| 用户管理 | `/users` | 见 `UserController` |
| 情侣 | `/couple/invite` `POST`、…；**`GET /couple/pending-invites`**、**`GET /couple/pending-invites/count`** | 需登录 |
| 时间轴 | `/timeline/upload` `POST`（图/视频 multipart）、`/timeline/records` …、`/timeline/memories` `GET`；**`POST /timeline/records/{id}/like`**；**`GET/POST /timeline/records/{id}/comments`**、**`DELETE .../comments/{commentId}`** | 需登录 |
| **媒体分片** | **`/media/uploads/init` `POST`**、**`PUT /media/uploads/{id}/chunks/{i}`**、**`GET .../status`**、**`POST .../complete`**、**`DELETE .../{id}`**；**`target`**=`TIMELINE`\|`ALBUM` | 需登录 |
| 相册 | `/albums` `POST` `GET`；**`PUT /albums/{id}`** 改名称；**`DELETE /albums/{id}`**；**`POST /albums/{id}/photos/from-url`**（分片后登记）；**`PUT /albums/{albumId}/photos/{photoId}`** 改照片元数据；**`GET /albums/{id}/photos?page&pageSize`** → **`AlbumPhotoPageResponse`**；删照片、收藏等同前 | 需登录 |
| **静态** | **`GET /local-files/{*path}`**（支持 **Range** / **206**） | 匿名（Security 放行） |
| **消息** | `/messages/send` `POST`、`/messages` `GET`、`/messages/{id}/read` `PUT`、`/messages/{id}/retract` `POST`、`/messages/scheduled` `POST` | 需登录 |
| **共同计划** | `/plans` `POST` `GET`；`/plans/{id}` `PUT` `DELETE`；`/plans/{id}/tasks` `POST`；`/plans/{id}/tasks/{taskId}` `PUT`（`PlanTaskReplaceRequest`）、`DELETE`；`/plans/{id}/expenses` `GET` `POST`；`/plans/{id}/expenses/{expenseId}` `PUT`（`PlanExpenseReplaceRequest`）、`DELETE` | 需登录 |
| **纪念日** | **`/memorial-days`** `POST` `GET` `GET/{id}` `PUT/{id}` `DELETE/{id}`；**`/memorial-days/next`**、**`/memorial-days/upcoming`** | 需登录 |
| **AI** | **`POST /ai/chat`** 通用对话；**`GET /ai/emotion?coupleId&startDate&endDate`** 情感分析报告；**`POST /ai/love-letter`** 情书生成 | 需登录；长耗时接口前端超时建议 ≥120s |

**WebSocket（非 `/api` 前缀）**：`ws://<host>:8081/ws/chat?token=<JWT>`，消息体 JSON。

> 实际路径均以类上 `@RequestMapping` 为准：`UserController` 为 `/users`，其余多为 `/api/v1/...`。

## 前端路由速查

| 路径 | 页面 |
|------|------|
| `/` | 首页 |
| `/login`、`/register` | 登录、注册 |
| `/profile` | 个人资料 |
| `/couple` | 情侣首页 |
| **`/inbox`** | **消息（待处理情侣邀请）** |
| `/timeline` | 恋爱时间轴 |
| `/album` | 情侣相册 |
| **`/chat`** | **私密消息** |
| **`/plan`** | **共同计划** |
| **`/emotion`** | **情感洞察**（AI 分析） |
| **`/love-letter`** | **AI 情书** |
| **`/memorial`** | **纪念日**（页面 **`Memorial.tsx`** 已实现；**默认未在 `App.tsx` 注册**，见 **`progress.md`**） |

## 与 `memory/` 的关系

- **`memory/goals.md`**：产品与技术目标。
- **`memory/progress.md`**：已实现功能与对话任务摘要。
- **`memory/decisions.md`**：不宜反复争论的约定（编译参数、安全、WebSocket、消息与上传等）。
- **`memory/blockers.md`**：环境依赖与风险提示。
- **`memory/CODING_RULES.md`**：**代码风格、注释、日志**规范，后续开发默认遵守。
- **`memory/DEPLOYMENT.md`**：**Docker Compose**、镜像构建、**MinIO 公网 URL / 桶策略 403**、**WebSocket 同源** 等运维说明。

新会话可先读 **`memory/progress.md`** + **`PROJECT_STRUCTURE.md`** + **`CODING_RULES.md`** 快速恢复上下文；**部署与联调**见 **`memory/DEPLOYMENT.md`**。

## 本次会话相关摘要（截至 2026-04）

- **前端**：Ant Design 浮层（`StyleProvider`、减少动画作用域、`getPopupContainer`、zIndex）；**`authHydrated`** 修复刷新误跳转登录；情侣页/AI 情书/页脚体验；**`http` `withCredentials`** 与 **`VITE_SESSION_DISTRIBUTED`**。
- **后端**：可选 **Redis 分布式 Session**（默认关）、**JWT 失败回退 Session**、**带凭证 CORS**（条件启用）。详见 **`memory/progress.md`** 对话任务 **39–49** 与 **`decisions.md`**「认证与安全」「前端全局样式与登录态」。
- **情侣邀请与部署**：**`/inbox`**、**pending-invites API**；**Docker** 与 **MinIO** 见 **`memory`** 中 **`progress.md`**（任务 **50–54**）与 **`DEPLOYMENT.md`**。
- **纪念日**：后端 **`/api/v1/memorial-days`** + Redis + DDL；前端 **`Memorial.tsx`** 等；**路由/顶栏入口可按需恢复**（见 **`progress.md`** 任务 **55–58**）。
