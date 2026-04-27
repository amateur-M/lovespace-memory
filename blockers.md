# 当前障碍

## 阻碍状态

**无硬性阻塞**；主流程在 MySQL + Redis 可用、已执行所需 DDL 的前提下可开发与联调。

## 环境依赖

1. **JDK**：Maven 编译/运行需 Java 21（避免 `mvn -v` 仍指向旧 JDK）
2. **MySQL**：库表需按 `sql/` 脚本初始化；若库早于 `images_json` 字段，需执行 `love_records_add_images_json.sql`；时间轴点赞/评论需执行 `love_record_social.sql`；相册需执行 `couple_albums.sql`；私密消息需执行 `private_messages.sql`；共同计划需执行 `couple_plans.sql`；若需计划消费流水与预算汇总，另执行 `plan_expenses.sql`；纪念日需执行 `memorial_days.sql`。**若 `users` 表尚无 `phone` 列（早于手机号账号改造）**，需执行 `users_add_phone_account.sql` 并核对占位手机号或改为真实号码后再约束 NOT NULL
3. **Redis**：登出黑名单依赖；不可用则相关接口可能异常。可选分布式 Session 亦依赖 Redis（`spring.session.store-type=redis`）；未启用时保持 `store-type: none`，勿误开 `lovespace.session.distributed.enabled` 而无 Redis Session 配置

## 已知注意事项（非阻塞）

1. **配置与密钥**：`application.yml` 中 JWT secret、数据库密码等应为本地/环境自有配置，勿提交敏感信息到公共仓库。AI 功能（对话、情感分析、情书）依赖 `spring.ai.dashscope.api-key`（通义千问）；未配置时相关接口将失败或回退为纯统计/错误提示。**恋爱问答 RAG** 已随 **`lovespace-ai`** 打入，需 **Milvus** 可达、`lovespace.ai.embedding.*` 与 **`spring.ai.vectorstore.milvus.*`**（含 **`embedding-dimension`**）一致；部署时可设 **`LOVESPACE_MILVUS_ENSURE_LOVE_KNOWLEDGE_SCHEMA=true`** 与 **`SPRING_AI_VECTORSTORE_MILVUS_INITIALIZE_SCHEMA=true`** 确保集合就绪。Compose 见根目录 `docker-compose.yml`（**milvus-etcd**、**milvus**、共用 **minio**）。嵌入与聊天共用 **`SPRING_AI_DASHSCOPE_API_KEY`**，一般不再需要仅为 RAG 配置 OpenAI Key
2. **前端构建**：Vite 可能对主 chunk 体积告警，不影响功能
3. **邀请流程**：已实现 `/inbox` 与 `pending-invites` API，被邀请方在消息页接受；若仍手调 API，需 `bindingId` 调用 `POST /couple/accept`
4. **时间轴媒体与静态访问**：直传依赖 `POST /api/v1/timeline/upload`；本机访问依赖 `LocalFileRangeController` 的 `GET /local-files/**`（含 Range）。生产需配置正确的 `public-base-url` 或 OSS；反向代理需正确转发 Range/If-Range，避免破坏视频分段缓冲
5. **MinIO 浏览器访问**：`endpoint=http://minio:9000` 时浏览器无法解析 `minio`；应配置 `lovespace.minio.public-base-url`（或 Compose `LOVESPACE_MINIO_PUBLIC_BASE_URL`）为公网可访问基址。直链对象匿名 GET 需桶策略允许读，否则 403；详见 `DEPLOYMENT.md`
6. **相册/时间轴大文件**：multipart 与分片 PUT 均可能受网关 `client_max_body_size`、磁盘空间、代理对 Content-Length 的改写影响；分片 API 为 `/api/v1/media/uploads`（已替代旧路径 `/api/v1/timeline/uploads`，第三方客户端需同步）
7. **相册灯箱**：`PhotoViewer` 仅加载当前分页内的照片，左右切换不跨页；需跨页浏览时先换页再点开
8. **相册照片元数据 PUT**：`PUT /api/v1/albums/{albumId}/photos/{photoId}` 为四字段整单替换（`description`、`locationJson`、`takenDate`、`tagsJson`，均可为 null 表示清空）；第三方客户端须显式传齐字段，避免误清空
9. **Python（可选）**：本地若需运行 ui-ux-pro-max 的 `search.py`，需已安装 Python；不影响主程序编译运行
10. **WebSocket 与部署**：本地开发已在 `vite.config.ts` 将 **`/ws`** 代理到 8081（含 `ws: true`）；`VITE_API_BASE_URL` 为空时私密消息页用同源 `ws(s)://当前 host/ws/chat`。生产需 **nginx** 等对 `/ws/**` 配置 **Upgrade / Connection**（见 `lovespace-frontend/nginx.conf`）。私密消息发送在 WS 未就绪时可 **回退 HTTP**（`POST /api/v1/messages/send` / `/scheduled`），见 `progress.md` 对话摘要
11. **私密消息 JWT**：WebSocket 使用 query `token` 传参，需注意 URL 长度与日志泄露风险（生产建议仅 HTTPS/WSS + 短链或子协议方案演进）
12. **共同计划预算**：若历史数据仅在 `couple_plans.budget_spent` 中、无 `plan_expenses` 流水，接口汇总 `budgetSpent` 可能为 0；需补录消费或自行迁移
13. **分布式 Session 前端**：须设置 `VITE_SESSION_DISTRIBUTED=true` 并重启 dev server 后 `withCredentials` 才生效；`VITE_API_BASE_URL` 指向另一 origin（如直连 8081）时需后端 CORS 与浏览器 Cookie 策略配合，推荐开发时 baseURL 为空走 Vite `/api` 代理
14. **Session Cookie**：`LOVESPACE_SESSION` 为 HttpOnly，勿用 `document.cookie` 排查，应使用开发者工具 Application → Cookies

## 可选后续增强

- Refresh token、统一全局异常处理
- 情侣「冻结」态的前端展示与运营规则
- 相册删除时物理文件清理策略（当前主要删库记录）
- 私密消息 WebSocket 发送 ack、断线重连策略与离线补偿
- 集成测试 / E2E 冒烟
- 分片合并失败时的重试策略与客户端 localStorage 续传键冲突处理（同文件多标签页）
