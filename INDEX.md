# memory/ 索引（按需加载）

本目录用于会话衔接与项目上下文。**新开 Cursor 窗口时请先读本文件**，再按任务只打开需要的文档，不必一次加载全部。

## 按任务快速路由

| 场景 | 打开文件 |
|------|------------|
| 代码风格、分层、前后端约定、注释与日志 | [CODING_RULES.md](./CODING_RULES.md)（项目规则已摘要至 `.cursor/rules/lovespace-coding-standards.mdc`） |
| 仓库目录、模块职责、API/路由速查 | [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md) |
| 架构决策、库表、认证、WebSocket、业务规则细节 | [decisions.md](./decisions.md) |
| 当前实现进度、功能清单、联调说明 | [progress.md](./progress.md) |
| 环境依赖、脚本顺序、已知风险与非阻塞项 | [blockers.md](./blockers.md) |
| 产品与技术目标、范围 | [goals.md](./goals.md) |
| Docker、nginx、MinIO、构建与 403 等问题 | [DEPLOYMENT.md](./DEPLOYMENT.md) |
| 恋爱问答 RAG、Milvus、前后端路径与注意点 | [love-qa-rag.md](./love-qa-rag.md) |
| RAG 优化专项（Embedding 缓存、Prompt 压缩、Latency 埋点、检索可视化） | [RAG_OPTIMIZATION.md](./RAG_OPTIMIZATION.md) |

## 文件一览（一句话）

| 文件 | 用途 |
|------|------|
| **INDEX.md** | 本索引：路由表 + 文件清单，供新窗口按需拉取上下文 |
| **CODING_RULES.md** | 编码规则：后端/前端风格、注释、日志；与 `decisions.md` 一并生效 |
| **PROJECT_STRUCTURE.md** | 根目录与前后端目录说明、API 与前端路由速查 |
| **decisions.md** | 关键决策：模块划分、DB、JWT/Session、WS、纪念日与前端 IA 等 |
| **progress.md** | 实时进度：已落地功能、账号手机号、分片上传、部署要点、**对话任务摘要**（含近期联调修复） |
| **blockers.md** | 障碍与依赖：JDK/MySQL/Redis 脚本、注意事项 |
| **goals.md** | 最终目标与拆解 |
| **DEPLOYMENT.md** | 容器化部署（含 **Milvus/etcd**、与业务 **共用 MinIO**、后端 fat jar 构建说明）、MinIO 公网 URL 与桶策略 |
| **love-qa-rag.md** | 恋爱知识库 RAG：`lovespace-ai`（含 RAG）默认打入 user、**DashScope 嵌入** + Milvus、集合启动自检；含 **多轮记忆（多消息 LLM + Redis/MySQL 回填）**、**SSE 流式 `/chat/stream`**、前端页布局与滚动约定；**2026-05 新增**：Embedding 缓存（Redis）、Prompt 压缩（去重+截断）、Latency 埋点（6 阶段计时）、检索可视化（SSE `retrieved` 事件） |
| **RAG_OPTIMIZATION.md** | **2026-05 新增**：RAG 模块四项优化详解——Embedding 缓存、Prompt 压缩、Latency 埋点、检索可视化的设计决策、实现要点与预期收益 |

## 建议加载顺序（典型任务）

1. **改接口或 Controller**：`PROJECT_STRUCTURE.md`（路径速查）→ 必要时 `decisions.md`（安全/参数约定）。
2. **改表或迁移**：`decisions.md`（表与字段约定）→ `blockers.md`（脚本依赖顺序）。
3. **改前端页面或 API 调用**：`CODING_RULES.md`（请求/超时/状态）→ `PROJECT_STRUCTURE.md`（路由与服务文件位置）。
4. **部署或线上资源 URL**：`DEPLOYMENT.md` → `blockers.md`（MinIO/代理）。
5. **恋爱问答 / Milvus RAG**：`love-qa-rag.md` → 必要时 `PROJECT_STRUCTURE.md`（路径）与 `lovespace-user` 的 `application.yml`。
6. **RAG 优化专项（缓存/压缩/埋点/可视化）**：`RAG_OPTIMIZATION.md`（详细设计）→ `love-qa-rag.md`（接口与配置）→ `decisions.md`（架构决策）。

## 与 Cursor 的配合

- 项目级编码规则：`.cursor/rules/lovespace-coding-standards.mdc`（`alwaysApply`，与本文档的 CODING_RULES 对齐）。
- 需要完整条文或更新规范时，再打开 `memory/CODING_RULES.md`。
