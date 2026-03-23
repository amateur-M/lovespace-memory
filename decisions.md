# 关键决策

## 架构与模块

1. 后端采用 Spring Boot 多模块 Maven；当前以 `lovespace-user` 为可启动服务。
2. 公共能力在 `lovespace-common`（如 `ApiResponse`）。

## 版本与构建

1. Java **21**（父 POM + Enforcer）。
2. **`maven-compiler-plugin`**：`<parameters>true</parameters>`，保留方法参数名供 Spring MVC 使用。
3. Controller 层对易踩坑参数仍使用 **显式名称**：`@RequestParam("coupleId")`、`@PathVariable("id")` 等，避免 IDE/非 Maven 编译链路缺参数名。

## 数据库

1. 排序规则：`utf8mb4_general_ci`（`users.sql` 等）。
2. **情侣绑定 `status`**：业务扩展 **`0=待接受邀请`**，文档与 `CoupleBindingStatus` 一致；接受后双方 ID **字典序**规范化存储。
3. 时间轴 **`images_json`**：存图片 URL 的 JSON 数组字符串；与头像共用 **OSS/本地** 上传实现，`AvatarStorageService.uploadTimelineImage` 使用 `timeline/` 路径前缀。
4. **相册**：表 **`couple_albums`**、**`album_photos`**；封面 `cover_image_url`；照片含 `thumbnail_url`、`location_json`、`tags_json`、`is_favorite` 等；脚本见 **`couple_albums.sql`**。

## 认证与安全

1. JWT（HS256）+ Spring Security **STATELESS**。
2. 登出：Redis 黑名单，TTL 对齐 token 剩余寿命。
3. 公开路径：`/api/v1/auth/register|login|logout`、`/health/**`、Swagger、`/local-files/**` 等；其余默认需认证。

## 业务规则摘录

1. **恋爱天数**：`ChronoUnit.DAYS.between(startDate, today) + 1`（含首日），业务日期 **Asia/Shanghai**。
2. **时间轴可见性**：`1` 仅作者，`2` 情侣双方；列表/回忆按当前用户过滤。
3. **时间轴列表日期筛选**：`startDate`/`endDate` 可选、闭区间；仅一端亦可；`startDate > endDate` 返回业务错误码 **40054**。
4. **相册**：创建/列表/删相册、上传/列表/删照片、收藏均需 **交往或冻结** 情侣成员校验；**删照片**仅 **上传者**；相册删除级联删照片记录（库记录，文件清理策略后续可补强）。

## 前后端联调

1. API 前缀 **`/api/v1`**。
2. 前端 Vite **`server.proxy`**：`/api`、`/local-files` → `127.0.0.1:8081`。
3. 前端情侣主键：使用 **`GET /couple/info` 返回的 `bindingId`** 作为时间轴与相册接口的 **`coupleId`**。
4. 前端静态资源：相对路径 `/local-files/**` 可通过 **`VITE_API_BASE_URL`** 与 **`resolveMediaUrl`** 拼完整 URL。

## 异常与响应

1. 情侣业务：`CoupleBindingBusinessException` + `CoupleControllerExceptionHandler`（限定 `CoupleController`）。
2. 时间轴：`TimelineBusinessException` + `TimelineControllerExceptionHandler`（限定 `TimelineController`）。
3. 相册：`AlbumBusinessException` + `AlbumControllerExceptionHandler`（限定 `AlbumController`）。
4. 未绑定情侣查询 info：**HTTP 200** + `code=40442`（前端常量 `COUPLE_NOT_BOUND_CODE`）。

## 上传与 Multipart

1. **头像 / 时间轴上传**：仍受 **`lovespace.avatar.max-size-bytes`** 与扩展名配置约束（应用层校验）。
2. **相册照片上传**：**不在 `AlbumController` 校验文件大小**；仅校验非空与扩展名（与 `lovespace.avatar.allowed-extensions` 一致，默认可扩展）。
3. **Spring Multipart**：`spring.servlet.multipart.max-file-size` / `max-request-size` 设为 **10GB**，避免默认过小在框架层拒绝；生产若经 Nginx 等，需同步 **`client_max_body_size`** 等。

## 前端视觉

1. **主题**：暖色（玫瑰主色 `#e11d48`、页面背景 `#fff1f2`、文字偏暖棕/玫瑰层次）；字体 **Inter**；Ant Design token 集中在 **`src/theme/antdTheme.ts`**。
2. **相册网格**：缩略图 **统一 1:1** 裁切展示；大图仍在灯箱中按原图比例 **`object-contain`**。

## 工程规范

1. 代码风格、注释与日志要求见 **`memory/CODING_RULES.md`**，后续开发默认遵守。
