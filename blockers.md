# 当前障碍

## 阻碍状态

**无硬性阻塞**；主流程在 MySQL + Redis 可用、已执行所需 DDL 的前提下可开发与联调。

## 环境依赖

1. **JDK**：Maven 编译/运行需 **Java 21**（避免 `mvn -v` 仍指向旧 JDK）。
2. **MySQL**：库表需按 `sql/` 脚本初始化；若库早于 `images_json` 字段，需执行 **`love_records_add_images_json.sql`**；**相册**需执行 **`couple_albums.sql`**（含 `couple_albums` 与 `album_photos`）。
3. **Redis**：登出黑名单依赖；不可用则相关接口可能异常。

## 已知注意事项（非阻塞）

1. **配置与密钥**：`application.yml` 中 JWT secret、数据库密码等应为本地/环境自有配置，勿提交敏感信息到公共仓库。
2. **前端构建**：Vite 可能对主 chunk 体积告警，不影响功能。
3. **邀请流程**：被邀请方需 **`bindingId`** 调用 accept；产品层可再优化为站内通知/链接。
4. **时间轴图片**：依赖 `/api/v1/timeline/upload` 与静态资源映射；生产环境需配置正确的 `public-base-url` 或 OSS。
5. **相册大图上传**：应用层虽不限大小，**网关 / 反向代理**（如 Nginx `client_max_body_size`）与**磁盘空间**仍可能限制实际上传；若失败需在链路各层排查。
6. **Python（可选）**：本地若需运行 **ui-ux-pro-max** 的 `search.py`，需已安装 Python（例如 `D:\Software\Python`）；不影响主程序编译运行。

## 可选后续增强

- Refresh token、统一全局异常处理、时间轴记录编辑/删除前端入口。
- 情侣「冻结」态的前端展示与运营规则。
- 相册删除时**物理文件**清理策略（当前主要删库记录）。
- 集成测试 / E2E 冒烟。
