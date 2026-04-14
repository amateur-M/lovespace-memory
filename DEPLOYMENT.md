# LoveSpace 容器化部署说明

本文汇总仓库根目录 **`docker-compose.yml`**、**`lovespace-backend/Dockerfile`**、**`lovespace-frontend/Dockerfile`**、**`lovespace-frontend/nginx.conf`**、**`env.example`** 的用法与常见问题。详细命令以服务器实际路径为准（示例：`/projects/lovespace/`）。

---

## 1. 目录结构（建议）

```
/projects/lovespace/
├── docker-compose.yml
├── env.example                    # 复制为 .env 后修改密钥与端口
├── lovespace-backend/
│   ├── Dockerfile
│   ├── target/
│   │   └── lovespace-user-0.0.1-SNAPSHOT.jar   # 仅保留一个可执行 fat jar
│   └── app.jar                    # 可选：与 Dockerfile 同级放置 fat jar
├── lovespace-frontend/
│   ├── Dockerfile
│   ├── dist/                      # npm run build 产物
│   └── nginx.conf
└── data/
    ├── mysql/
    ├── redis/
    ├── minio/
    └── uploads/                   # 后端本地存储回退目录挂载
```

**宿主机无需单独安装 Nginx**：前端镜像内已使用 `nginx:alpine`；仅在同机多站点、HTTPS 证书、统一网关等场景再考虑宿主机 Nginx。

---

## 2. 构建产物准备

### 后端（多模块）

在 **`lovespace-backend`** 父目录执行：

```bash
mvn -pl lovespace-user -am package -DskipTests
```

将 **`lovespace-user/target/lovespace-user-0.0.1-SNAPSHOT.jar`**（版本号以实际为准）复制到部署目录 **`lovespace-backend/target/`**，且**只保留一个** jar，或复制为 **`lovespace-backend/app.jar`**。

**可执行 jar 要求**：`META-INF/MANIFEST.MF` 中含 Spring Boot **`JarLauncher`**（父 POM 已在 **`pluginManagement`** 为 **`spring-boot-maven-plugin`** 指定 **`${spring-boot.version}`**，`lovespace-user` 模块执行 **`repackage`**）。若出现 **`no main manifest attribute`**，多为拷错模块 jar（如 common/ai）或未完整执行 **`package`**。

### 前端

```bash
npm run build
```

将 **`dist/`** 同步到 **`lovespace-frontend/dist/`**。

---

## 3. 环境变量

复制 **`env.example`** 为 **`.env`**（与 `docker-compose.yml` 同级）。Docker Compose 变量插值语法为 **`${VAR:-默认值}`**（**`:-`**），不能写成 **`${VAR:值}`**。

首次部署需在 **MySQL** 中执行 **`lovespace-user/src/main/resources/sql/`** 下脚本建库表；**MinIO** 需在控制台或 **`mc`** 创建与 **`LOVESPACE_MINIO_BUCKET`** 一致的桶。

---

## 4. 编排服务说明

| 服务 | 说明 |
|------|------|
| **mysql** | 数据卷 `./data/mysql`；健康检查通过后 **`backend`** 再启动。 |
| **redis** | 数据卷 `./data/redis`；分布式 Session 与黑名单等依赖。 |
| **minio** | API 默认映射 **9000**，控制台 **9001**；数据卷 `./data/minio`。 |
| **backend** | 环境变量覆盖 **`application.yml`**：如 **`SPRING_DATASOURCE_URL`** 指向 `mysql:3306`、**`LOVESPACE_MINIO_ENDPOINT`=`http://minio:9000`**，等。 |
| **frontend** | 反代 **`/api/`**、**`/local-files/`**、**`/ws/`** → **`backend:8081`**；**`client_max_body_size`** 与后端大文件上传对齐。 |

仅启动基础依赖：

```bash
docker compose up -d mysql redis minio
```

全量启动：

```bash
docker compose build
docker compose up -d
```
看日志：
```bash
compose logs backend --tail 80
```

---

## 5. 前端与 WebSocket（同源部署）

生产环境 **`VITE_API_BASE_URL` 留空** 时，请求走同源 **`/api`**。**`Chat.tsx`** 在 `VITE_API_BASE_URL` 为空时使用 **`ws://`/`wss://` + `window.location.host`** 连接 **`/ws/chat`**，需保证 Nginx 对 **`/ws/`** 配置 **`Upgrade`** / **`Connection`**（仓库内 `nginx.conf` 已配置）。

---

## 6. MinIO 与图片 403

### 6.1 返回给前端的 URL 形态

**`MinioAvatarStorageService.buildUrl`**（`lovespace.minio`）：

- 若配置了 **`lovespace.minio.public-base-url`**，返回 **`{publicBaseUrl}/{bucket}/{objectKey}`**（**`publicBaseUrl` 末尾不要带桶名**，避免重复；代码已含 **`bucket`** 段）。
- 未配置时，使用 **`http://{strip(endpoint)}/{bucket}/{objectKey}`**（**`endpoint` 为 `http://minio:9000` 时浏览器无法解析主机名 `minio`**，生产应配置 **`LOVESPACE_MINIO_PUBLIC_BASE_URL`** 为浏览器可访问地址，如 **`http://公网IP:9000`** 或域名）。

### 6.2 浏览器 403 Forbidden

直链 **`http://IP:9000/bucket/object`** 时，匿名 **GET** 需桶策略允许读；默认桶为**私有**，会 **403**。处理：

- 在 MinIO 控制台为桶设置 **匿名下载 / readonly**（或 **`mc anonymous set download`**），或
- 使用 **预签名 URL**（当前代码未实现，需后续扩展）。

**仅加 Nginx 反代不能消除 403**，除非反代到后端并由服务端带签名拉流（当前未实现）。

---

## 7. 故障排查简表

| 现象 | 可能原因 |
|------|----------|
| 后端 **`no main manifest attribute`** | 非 `lovespace-user` fat jar 或未 **`repackage`**。 |
| Compose 插值报错 | 使用 **`${VAR:-default}`** 语法。 |
| 图片 403 | 桶未开放匿名读；或 URL 缺少 **`/{bucket}/`** 段导致路径错误。 |
| 图片无法加载（内网 `minio` 主机名） | 配置 **`LOVESPACE_MINIO_PUBLIC_BASE_URL`** 为公网可访问 MinIO 基址。 |

---

## 8. 相关代码与配置

- 后端存储：**`MinioAvatarStorageService`**、**`MinioProperties`**（`lovespace.minio.*`）。
- 前端媒体 URL：**`utils/mediaUrl.ts`**（`resolveMediaUrl`）。
- 编排与镜像：仓库根 **`docker-compose.yml`**、**`lovespace-backend/Dockerfile`**、**`lovespace-frontend/Dockerfile`**、**`lovespace-frontend/nginx.conf`**、**`env.example`**。
