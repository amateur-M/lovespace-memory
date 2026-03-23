# 实时进度

## 当前总体状态

- 用户认证、个人资料（含头像上传）、**情侣绑定**、**恋爱时间轴（含图片与区间筛选）**、**情侣相册（后端 API + 前端页面）** 均已落地并可联调。
- 接口前缀统一 `/api/v1`；Vite 开发代理 `/api`、`/local-files` → 后端 `8081`。
- 父 POM 已启用 **`maven-compiler-plugin` `parameters=true`**，且 Controller 中关键 `@RequestParam`/`@PathVariable` 已写显式名称，避免 Spring 绑定报错。
- 前端已统一 **暖色主题**（玫瑰主色 + 浅粉背景，Inter 字体），相册照片列表为 **等尺寸网格**（`aspect-square` + `object-cover`）。
- **相册上传**：Controller 层**不再校验**单文件大小；`spring.servlet.multipart` 放宽至 **10GB**（实际仍受 Tomcat、反向代理、磁盘限制）。

---

## 后端（`lovespace-backend`）

### 模块

| 模块 | 说明 |
|------|------|
| `lovespace-common` | 统一响应 `ApiResponse` 等 |
| `lovespace-user` | 可运行用户服务（端口默认 8081） |

### 数据表脚本（`lovespace-user/src/main/resources/sql/`）

- `users.sql` — 用户表
- `couple_binding.sql` — 情侣绑定（含 status：`0` 待接受、`1` 交往、`2` 冻结、`3` 解除）
- `love_records.sql` — 时间轴记录（含 `images_json`）
- `love_records_add_images_json.sql` — **已有库**仅缺图片列时执行
- **`couple_albums.sql`** — **`couple_albums`**、**`album_photos`**（情侣相册与照片）

### 主要 API 分组

- **认证 / 资料**：`/api/v1/auth/*`，`/api/v1/user/profile`（含头像）
- **情侣**：`/api/v1/couple/*`（invite、accept、info、start-date、separate）
- **时间轴**：`/api/v1/timeline/*`
  - `POST /upload` — 时间轴配图上传 → URL（仍有应用层大小与类型限制）
  - `GET/POST /records`，`GET/PUT/DELETE /records/{id}`
  - `GET /records?coupleId&page&pageSize&startDate&endDate` — **可选日期区间**
  - `GET /memories` — 回忆推送
- **相册**：`/api/v1/albums/*`
  - `POST /albums` — 创建相册
  - `GET /albums?coupleId` — 相册列表
  - `DELETE /albums/{id}` — 删除相册
  - `POST /albums/{id}/photos` — 上传照片（**无应用层大小上限**；类型仍按配置扩展名）
  - `GET /albums/{id}/photos` — 照片列表
  - `DELETE /albums/{albumId}/photos/{photoId}` — 删除照片
  - `PUT /albums/{albumId}/photos/{photoId}/favorite` — 收藏

### 领域要点

- `CoupleBindingService.findActiveOrFrozenMembership`：校验 `coupleId` 是否为当前用户有效情侣（相册与时间轴均复用）。
- 时间轴列表可见性：`visibility=2` 或（`visibility=1` 且作者为当前用户）；改删仅作者。
- 心情：`LoveMood`（happy/sad/excited/calm/loved/missed）。
- 存储：头像、时间轴图、相册图共用 `AvatarStorageService` 策略；路径前缀分别为头像配置、`timeline/`、`albums/`。

---

## 前端（`lovespace-frontend`）

### 路由（`App.tsx`）

- `/login`、`/register`（`AuthPageShell` 暖色渐变背景）
- `/` 首页、`/profile` 个人资料、`/couple` 情侣首页、`/timeline` 时间轴、**`/album` 相册**
- `*` → 404

### 状态与服务

- `authStore` + `services/auth.ts`、`http.ts`
- `coupleStore` + `services/couple.ts`（`bindingId` 作 `coupleId` 调时间轴与相册）
- `services/timeline.ts` — 列表（含区间）、创建、上传图
- **`services/album.ts`** — 相册与照片 CRUD、上传、收藏

### UI 与主题

- **`src/theme/antdTheme.ts`** + **`App.tsx`** `ConfigProvider`：`locale=zhCN`，暖色 token（主色玫瑰、背景 `#fff1f2` 等）。
- **`index.css`**：`ls-surface`、`ls-page-intro`、`ls-link`；**Inter** 字体（`index.html` 引入）。
- **顶栏**：浅色 + 底边线 + `AppLayout` 菜单含「相册」。

### 页面与组件摘要

- **时间轴**：`Timeline.tsx`、`TimelineForm`、`TimelineItem`、`MoodTag`
- **情侣**：`CoupleHome`、`CoupleCard`、`DaysCounter`
- **相册**：`Album.tsx`；**`AlbumCard`**、**`PhotoGrid`**（统一正方形缩略图）、**`PhotoViewer`**（灯箱）、**`UploadButton`**
- **工具**：`utils/mediaUrl.ts` — 相对路径图片 URL 拼接

### 验证

- `npm run build` 已通过（以当前依赖为准）。

---

## 对话任务摘要（按时间线合并）

### 历史基线（更早会话）

1. JavaDoc 与后端多文件注释规范。
2. **情侣绑定**全链路；**情侣首页**前端；**恋爱时间轴**前后端；**图片**与 `images_json`、`POST /timeline/upload`。
3. **Spring 参数名**：`-parameters` + 显式 `@RequestParam`/`@PathVariable`。
4. **时间轴日期区间**：后端 `startDate`/`endDate`，前端 RangePicker。

### 近期本会话

5. **情侣相册后端**：`couple_albums.sql`（`couple_albums`、`album_photos`）、实体 `Album`/`Photo`、`AlbumMapper`/`PhotoMapper`、`AlbumService`/`AlbumServiceImpl`、`AlbumController` + `AlbumBusinessException`/`AlbumControllerExceptionHandler`；`AvatarStorageService.uploadAlbumPhoto`（本地/OSS）。
6. **情侣相册前端**：`Album.tsx`、`AlbumCard`、`PhotoGrid`、`PhotoViewer`、`UploadButton`、`services/album.ts`；路由 `/album` 与导航。
7. **UI**：先极简石色系，后经 **ui-ux-pro-max**（Python `search.py`）调整为 **暖色**（玫瑰 + 浅粉 + Inter）；全站相关页面与组件配色统一。
8. **相册缩略图**：由瀑布流改为 **CSS Grid + `aspect-square` + `object-cover`**，解决高矮不一。
9. **相册上传**：去掉 Controller 内大小校验；`application.yml` 配置 `spring.servlet.multipart` **10GB** 上限。
10. **工程文档**：更新 `memory/*`；新建 **`memory/CODING_RULES.md`**（风格、注释、日志）。
