# LoveSpace 项目结构说明

仓库根目录：`meng-lovespace`

## 目录结构

```
meng-lovespace/
├── docker-compose.yml               # MySQL / Redis / MinIO / backend / frontend 编排
├── env.example                      # 部署环境变量模板（复制为 .env）
├── memory/                          # 会话衔接：目标、进度、决策、障碍、结构、编码规则、部署
│   ├── goals.md                     # 产品与技术目标
│   ├── progress.md                  # 已实现功能与对话任务摘要
│   ├── decisions.md                 # 架构约定（编译参数、安全、WebSocket、消息与上传等）
│   ├── blockers.md                  # 环境依赖与风险提示
│   ├── PROJECT_STRUCTURE.md         # 本文件
│   ├── CODING_RULES.md              # 编码规则（风格、注释、日志）
│   └── DEPLOYMENT.md                # 容器部署与 MinIO/403 说明
│
├── lovespace-backend/               # 后端（Maven 多模块）
│   ├── Dockerfile                   # 可执行 jar → 镜像
│   ├── pom.xml                      # 父 POM：Java 21、Spring Boot 3.4.x、spring-ai-bom、compiler parameters
│   ├── lovespace-common/            # 公共模块（ApiResponse 等）
│   ├── lovespace-ai/                # AI：LLM 抽象、通义千问/OpenAI、通用对话 Controller
│   └── lovespace-user/              # 用户服务（可执行 Spring Boot；依赖 lovespace-ai）
│       ├── src/main/java/com/meng/lovespace/user/
│       │   ├── config/              # Security、JWT、Session、MyBatis-Plus、OSS/本地存储等
│       │   ├── controller/          # Auth、User、Couple、Timeline、Album、Message、Plan、EmotionAnalysis、LoveLetter、MemorialDay
│       │   ├── entity/              # User、CoupleBinding、LoveRecord、Album、Photo、PrivateMessage、Plan、PlanTask、PlanExpense、MemorialDay
│       │   ├── service/ + impl/     # 业务逻辑层
│       │   ├── mapper/              # MyBatis-Plus Mapper
│       │   ├── util/                # JwtUtil、PhoneNormalizer 等
│       │   ├── oss/                 # AvatarStorageService（avatar/timeline/album）
│       │   ├── websocket/           # WebSocketConfig、ChatWebSocketHandler、ChatSessionRegistry
│       │   └── task/                # 定时任务（消息派发、分片清理、纪念日缓存刷新）
│       └── src/main/resources/
│           ├── application.yml
│           └── sql/                 # DDL 脚本
└── lovespace-frontend/              # 前端（Vite + React + TS）
    ├── Dockerfile                   # nginx 提供 dist + nginx.conf
    ├── nginx.conf                   # 生产：反代 /api、/local-files、/ws → backend
    ├── package.json
    ├── vite.config.ts               # 代理 /api、/local-files → 8081
    └── src/
        ├── main.tsx                 # StyleProvider hashPriority；BrowserRouter
        ├── App.tsx                  # 路由配置
        ├── theme/antdTheme.ts       # Ant Design 暖色主题
        ├── index.css                # 全局样式与工具类
        ├── layouts/AppLayout.tsx    # 导航布局；authHydrated 前加载态
        ├── pages/                   # HomePage、Login、Register、Profile、CoupleHome、Inbox、Timeline、Album、Chat、Plan、EmotionAnalysis、AILoveLetter、Memorial
        ├── components/              # 复用组件（AuthPageShell、TimelineItem、AlbumCard、MessageBubble、PlanListItem 等）
        │   └── memorial/            # 纪念日：MemorialRomanticDecor（装饰）、MemorialPhotoWall（相册环绕照片墙 + Image 预览）
        ├── services/                # API 封装（http、auth、couple、timeline、mediaChunkUpload、album、plan、emotion、loveLetter、memorial）
        ├── stores/                  # Zustand（authStore、inboxStore、memorialStore）
        └── utils/                   # 工具函数（mediaUrl、timelineMedia、planProgress）
```

## 后端 API 速查（前缀 `/api/v1`）

| 区域 | 路径模式 | 说明 |
|------|----------|------|
| 认证 | `/auth/register` POST（**phone**、username、password）、`/auth/login` POST（**phone**、password）、`/auth/logout` POST | 公开 |
| 资料 | `/user/profile` GET PUT（可含 **username**、**email**；邮箱可空）；`/user/profile/avatar` POST | 需登录 |
| 用户管理 | `/users` | 见 UserController |
| 情侣 | `/couple/invite` POST（body：**inviteePhone**）、accept、info、start-date、separate；**GET /couple/pending-invites**、**GET .../pending-invites/count** | 需登录 |
| 时间轴 | `/timeline/upload` POST（图/视频 multipart）、`/timeline/records` CRUD、`/timeline/memories` GET；**POST /records/{id}/like**；**GET/POST /records/{id}/comments**、**DELETE .../comments/{commentId}** | 需登录 |
| **媒体分片** | **`/media/uploads/init` POST**、**PUT /{uploadId}/chunks/{i}**、**GET .../status**、**POST .../complete**、**DELETE .../{uploadId}**；**target**=TIMELINE\|ALBUM | 需登录 |
| 相册 | `/albums` POST GET；**PUT /albums/{id}** 改名称；**DELETE /albums/{id}**；**POST /albums/{id}/photos/from-url**（分片后登记）；**PUT /albums/{albumId}/photos/{photoId}** 改照片元数据；**GET /albums/{id}/photos?page&pageSize** → **AlbumPhotoPageResponse**；删照片、收藏等同前 | 需登录 |
| **静态** | **GET /local-files/{*path}**（支持 Range / 206） | 匿名（Security 放行） |
| **消息** | `/messages/send` POST、`/messages` GET、`/messages/{id}/read` PUT、`/messages/{id}/retract` POST、`/messages/scheduled` POST | 需登录 |
| **共同计划** | `/plans` POST GET；`/plans/{id}` PUT DELETE；`/plans/{id}/tasks` POST；`/plans/{id}/tasks/{taskId}` PUT（PlanTaskReplaceRequest）、DELETE；`/plans/{id}/expenses` GET POST；`/plans/{id}/expenses/{expenseId}` PUT（PlanExpenseReplaceRequest）、DELETE | 需登录 |
| **纪念日** | **`/memorial-days`** POST GET GET/{id} PUT/{id} DELETE/{id}；**`/memorial-days/next`**、**`/memorial-days/upcoming`** | 需登录 |
| **AI** | **POST /ai/chat** 通用对话；**GET /ai/emotion?coupleId&startDate&endDate** 情感分析报告；**POST /ai/love-letter** 情书生成 | 需登录；长耗时接口前端超时建议 ≥120s |

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
| **`/memorial`** | **纪念日**（`Memorial.tsx`：倒计时在上、相册环绕照片墙、FloatButton 新建、折叠列表管理） |

## 快速恢复上下文

新会话建议阅读顺序：
1. **`progress.md`** - 了解已实现功能与最新进展
2. **本文件** - 掌握项目结构与 API 路由
3. **`CODING_RULES.md`** - 遵守编码规范
4. **`decisions.md`** - 理解架构约定与技术选型
5. **`blockers.md`** - 注意环境依赖与已知问题
6. **`DEPLOYMENT.md`** - 部署与联调指南（需要时）
