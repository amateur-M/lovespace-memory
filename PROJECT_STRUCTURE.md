# LoveSpace 项目结构说明

仓库根目录：`meng-lovespace`（示例路径 `d:\Project\meng-lovespace`）。

```
meng-lovespace/
├── memory/                          # 会话衔接：目标、进度、决策、障碍、结构、编码规则
│   ├── goals.md
│   ├── progress.md
│   ├── decisions.md
│   ├── blockers.md
│   ├── PROJECT_STRUCTURE.md         # 本文件
│   └── CODING_RULES.md              # 编码规则（风格、注释、日志）
│
├── lovespace-backend/               # 后端（Maven 多模块）
│   ├── pom.xml                      # 父 POM：Java 21、compiler parameters、依赖管理
│   ├── lovespace-common/            # 公共模块
│   │   └── src/main/java/com/meng/lovespace/common/
│   │       └── web/ApiResponse.java
│   └── lovespace-user/              # 用户服务（可执行 Spring Boot）
│       ├── pom.xml
│       └── src/main/
│           ├── java/com/meng/lovespace/user/
│           │   ├── LoveSpaceUserApplication.java
│           │   ├── config/          # Security、JWT、MyBatis-Plus、OSS/本地存储、上传等
│           │   ├── controller/      # Auth、Profile、User、Health、Couple、Timeline、Album
│           │   ├── couple/
│           │   ├── timeline/
│           │   ├── dto/
│           │   ├── entity/          # User、CoupleBinding、LoveRecord、Album、Photo
│           │   ├── exception/       # Couple*、Timeline*、AlbumBusinessException
│           │   ├── mapper/
│           │   ├── oss/             # AvatarStorageService（avatar / timeline / album）
│           │   ├── security/
│           │   ├── service/ + service/impl/
│           │   └── util/
│           └── resources/
│               ├── application.yml  # 含 spring.servlet.multipart（相册大文件）
│               └── sql/             # users、couple_binding、love_records、images_json 增量、couple_albums
└── lovespace-frontend/              # 前端（Vite + React + TS）
    ├── package.json
    ├── vite.config.ts               # 代理 /api、/local-files → 8081
    ├── index.html                   # Inter 字体、lang=zh-CN
    └── src/
        ├── main.tsx
        ├── App.tsx                  # 路由 + ConfigProvider 主题与 zhCN
        ├── theme/antdTheme.ts       # 暖色 Ant Design token
        ├── index.css                # ls-* 工具类、全局字体与动效降级
        ├── layouts/AppLayout.tsx
        ├── pages/
        │   ├── HomePage.tsx
        │   ├── Login.tsx、Register.tsx
        │   ├── Profile.tsx
        │   ├── CoupleHome.tsx
        │   ├── Timeline.tsx
        │   ├── Album.tsx
        │   └── NotFoundPage.tsx
        ├── components/
        │   ├── AuthPageShell.tsx
        │   ├── EChartsDemo.tsx
        │   ├── CoupleCard.tsx、DaysCounter.tsx
        │   ├── MoodTag.tsx
        │   ├── TimelineItem.tsx、TimelineForm.tsx
        │   ├── AlbumCard.tsx、PhotoGrid.tsx、PhotoViewer.tsx、UploadButton.tsx
        ├── services/
        │   ├── http.ts
        │   ├── auth.ts、couple.ts、timeline.ts、album.ts
        ├── stores/
        ├── utils/mediaUrl.ts
        └── ...
```

## 后端 API 速查（前缀 `/api/v1`）

| 区域 | 路径模式 | 说明 |
|------|----------|------|
| 认证 | `/auth/register` `POST`、`/auth/login` `POST`、`/auth/logout` `POST` | 公开 |
| 资料 | `/user/profile` `GET` `PUT`；`/user/profile/avatar` `POST` | 需登录 |
| 用户管理 | `/users` | 见 `UserController`（路径以类上注解为准） |
| 情侣 | `/couple/invite` `POST`、`/couple/accept` `POST`、`/couple/info` `GET`、`/couple/start-date` `PUT`、`/couple/separate` `POST` | 需登录 |
| 时间轴 | `/timeline/upload` `POST`、`/timeline/records` …、`/timeline/memories` `GET` | 需登录 |
| **相册** | `/albums` `POST` `GET`、`/albums/{id}` `DELETE`、`/albums/{id}/photos` `POST` `GET`、`/albums/{albumId}/photos/{photoId}` `DELETE`、`.../favorite` `PUT` | 需登录 |

> 实际路径均以类上 `@RequestMapping` 为准：`UserController` 为 `/users`，其余多为 `/api/v1/...`。

## 前端路由速查

| 路径 | 页面 |
|------|------|
| `/` | 首页 |
| `/login`、`/register` | 登录、注册（无主布局） |
| `/profile` | 个人资料 |
| `/couple` | 情侣首页 |
| `/timeline` | 恋爱时间轴 |
| `/album` | 情侣相册 |

## 与 `memory/` 的关系

- **`memory/goals.md`**：产品与技术目标。
- **`memory/progress.md`**：已实现功能与对话任务摘要。
- **`memory/decisions.md`**：不宜反复争论的约定（编译参数、状态码、联调、相册与上传等）。
- **`memory/blockers.md`**：环境依赖与风险提示。
- **`memory/CODING_RULES.md`**：**代码风格、注释、日志**规范，后续开发默认遵守。

新会话可先读 **`memory/progress.md`** + **`PROJECT_STRUCTURE.md`** + **`CODING_RULES.md`** 快速恢复上下文。
