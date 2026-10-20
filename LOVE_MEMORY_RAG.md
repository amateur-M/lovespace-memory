# 情侣记忆 RAG（Love Memory RAG）设计文档与可执行计划

> 本文档基于 2026-06 代码现状与产品定位分析，规划「情侣记忆助手」RAG 的完整设计与分阶段实施计划。
> 关联文档：[love-qa-rag.md](./love-qa-rag.md)（现有恋爱问答 RAG）、[RAG_PHASED_OPTIMIZATION.md](./RAG_PHASED_OPTIMIZATION.md)（现有检索优化）、[decisions.md](./decisions.md)、[progress.md](./progress.md)。

---

## 1. 背景与定位

### 1.1 现状

当前恋爱问答 RAG（`/api/v1/ai/love-qa/*`）是「**恋爱百科问答**」：用户手动粘贴/上传/URL 抓取恋爱文章入库，AI 基于这些文章回答。技术底座已完备：

- Milvus 向量库 + DashScope `text-embedding-v2`（1536 维，COSINE）
- 情侣隔离（`coupleId` metadata filter + `scope=COUPLE|GLOBAL`）
- 混合检索（向量 + MySQL FULLTEXT RRF 融合）
- SSE 流式 + `retrieved` 引用卡片 + 历史快照
- Redis 多轮记忆 + MySQL 持久化

### 1.2 产品错位

| 维度 | 当前实现 | 情侣产品真正需要 |
|------|---------|-----------------|
| 知识来源 | 用户手动入库的恋爱文章 | **系统内已有数据**（时间轴、相册、纪念日、计划） |
| 数据主体 | 通用恋爱文章 | **情侣私有动态数据**（自动生成） |
| 使用门槛 | 需绑定情侣 + 手动入库 | 打开即用，零配置 |
| 回答内容 | 通用建议（引用文章） | **个性化回忆**（"我们第一次去杭州是什么时候？"） |
| 冷启动 | 空知识库 → 拒答 | 数据天然存在（时间轴/相册/纪念日） |

### 1.3 目标形态：双库 RAG

```
┌─────────────────────────────────────────────────────┐
│  LoveSpace RAG（目标架构）                           │
│                                                     │
│  【库 A：情侣私有记忆库】← 自动/半自动生成，核心价值    │
│  ├─ 时间轴记录（love_records）                       │
│  ├─ 相册照片元数据（album_photos）                   │
│  ├─ 纪念日（memorial_days）                          │
│  ├─ 共同计划（couple_plans + plan_tasks）            │
│  └─ 恋爱开始日（couple_binding.start_date）          │
│  → 回答："我们纪念日是什么时候？""上次旅行去了哪？"   │
│  → 效果：个性化、可溯源、不可替代                    │
│                                                     │
│  【库 B：恋爱知识库】← 现有实现，保留为辅助            │
│  └─ 手动入库的恋爱文章（love_qa_documents）          │
│  → 回答："吵架了怎么办？"（通用建议）                │
└─────────────────────────────────────────────────────┘
```

### 1.4 设计原则

1. **零门槛**：绑定情侣后无需手动入库，AI 即可回答关于两人关系的问题
2. **可溯源**：回答引用真实记录，前端可点击跳转到对应时间轴/相册/纪念日
3. **隐私优先**：记忆库严格 `coupleId` 隔离；私密消息**不入库**（对话式数据 + 隐私风险）
4. **复用现有底座**：Milvus、filter、SSE、引用卡片、RRF 融合全部复用，不重复建设
5. **增量同步**：业务数据变更（新增/删除/修改）→ 增量更新向量，避免全量重建

---

## 2. 数据源与字段映射

### 2.1 数据源清单

| 数据源 | 表 | 触发方式 | 入库内容 |
|--------|-----|---------|---------|
| 时间轴记录 | `love_records` | 新增/编辑/删除 | 文字 + 心情 + 地点 + 标签 + 日期 |
| 相册照片 | `album_photos` | 上传/编辑/删除 | 描述 + 地点 + 拍摄日 + 标签 + 相册名 |
| 纪念日 | `memorial_days` | 新增/编辑/删除 | 名称 + 描述 + 日期 |
| 共同计划 | `couple_plans` + `plan_tasks` | 新增/编辑/删除 | 标题 + 描述 + 类型 + 任务 |
| 恋爱开始日 | `couple_binding` | 绑定/修改 start_date | 恋爱天数 + 开始日期 |

### 2.2 统一 chunk 文本模板

每个数据源生成一段结构化文本，作为 Milvus 的 `Document.text`：

```
【时间轴】2024-05-20 心情：开心
我们第一次去杭州旅行，在西湖边散步，吃了楼外楼。
地点：杭州西湖
标签：旅行、第一次
```

```
【相册】相册：杭州之旅
照片描述：断桥残雪，两人合影
拍摄日期：2024-05-21
地点：杭州断桥
标签：旅行
```

```
【纪念日】名称：在一起纪念日
描述：2023-05-20 在一起
日期：每年 5 月 20 日
```

```
【计划】类型：旅行
标题：杭州三日游
描述：五一去杭州，住西湖边
任务：订酒店（已完成）、买高铁票（未完成）
```

```
【恋爱】开始日期：2023-05-20
在一起：已 1000 天
```

### 2.3 统一 metadata 规范

| 字段 | 类型 | 说明 |
|------|------|------|
| `documentId` | String | UUID（记忆条目 ID，与台账一致） |
| `coupleId` | String | 情侣绑定 ID（**必填**，隔离依据） |
| `scope` | String | 固定 `COUPLE` |
| `sourceType` | String | `TIMELINE` \| `ALBUM_PHOTO` \| `MEMORIAL` \| `PLAN` \| `COUPLE_INFO` |
| `sourceId` | String | 业务记录主键（如 `love_records.id`），用于前端跳转 |
| `title` | String | 条目标题（如"杭州之旅"） |
| `date` | String | 业务日期（ISO，如 `2024-05-20`） |
| `mood` | String | 心情（仅 TIMELINE） |
| `category` | String | 分类（如"旅行""纪念日"） |
| `ingestedAt` | String | 入库时间（ISO 8601） |
| `chunkIndex` / `totalChunks` | Int | 分片序号（长文本时） |

### 2.4 分片策略

- 单条记录通常较短（< 800 字符），**默认不分片**（1 条记录 = 1 个 chunk）
- 超长记录（如长文时间轴、计划描述）按现有 `TokenTextSplitter`（800/100）分片
- 复用 `DocumentIngestPipeline`，但增加 `sourceType`/`sourceId` 到 metadata

---

## 3. 架构设计

### 3.1 模块归属

| 模块 | 职责 |
|------|------|
| `lovespace-ai` | 记忆库检索（复用 `RagRetrievalPipeline`）、双库融合、Prompt 构建 |
| `lovespace-user` | 记忆条目生成（`MemoryEntryBuilder`）、增量同步（事件/定时）、台账表、Controller |

### 3.2 核心组件

```
┌──────────────────────────────────────────────────────────────┐
│  lovespace-user                                              │
│                                                              │
│  MemoryEntryBuilder（数据源 → 结构化文本 + metadata）          │
│  MemorySyncService（增量同步：变更事件 / 定时全量兜底）         │
│  LoveMemoryDocumentService（台账 CRUD，复用 LoveQaDocument 模式）│
│  LoveMemoryController（/api/v1/ai/love-memory/*）             │
└──────────────────────────────────────────────────────────────┘
        │ 调用 ingestDocumentWithTracking（复用）
        ▼
┌──────────────────────────────────────────────────────────────┐
│  lovespace-ai                                                │
│                                                              │
│  LoveQAService（现有，扩展双库检索）                           │
│  RagRetrievalPipeline（现有，复用）                            │
│  MemoryRetrievalRouter（新增：query 路由到记忆库/知识库/双库）  │
└──────────────────────────────────────────────────────────────┘
```

### 3.3 检索路由策略

**方案 A（推荐，简单可靠）**：双库并行检索 + RRF 融合

```
query
  ├─→ 记忆库检索（filter: coupleId == X && scope == COUPLE）
  │     → 记忆 chunks
  ├─→ 知识库检索（filter: coupleId == X || scope == GLOBAL）
  │     → 知识 chunks
  └─→ RRF 融合（复用 RagReciprocalRankFusion，按 documentId 合并）
        → topK 进 Prompt
```

**方案 B（进阶）**：query 意图路由

```
query → 轻量分类（规则/LLM）
  ├─ 涉及"我们/上次/纪念日/旅行/回忆" → 仅记忆库
  ├─ 涉及"怎么办/建议/技巧" → 仅知识库
  └─ 混合 → 双库
```

> **建议**：先做方案 A（零额外 LLM 成本、实现简单），上线后根据日志判断是否需要方案 B。

### 3.4 Prompt 增强

在现有 `RAG_SYSTEM_PREFIX` 基础上，增加记忆库来源说明：

```
【检索到的上下文】可能包含两类来源：
- 情侣记忆（【记忆】前缀）：来自你们的时间轴、相册、纪念日、计划等真实记录
- 恋爱知识（【知识】前缀）：来自恋爱知识库文章
回答时：
1. 涉及你们俩的具体事实（日期、地点、事件）时，优先引用【记忆】来源，并明确"根据你们的记录"
2. 涉及通用建议时，引用【知识】来源
3. 记忆与知识冲突时，以【记忆】为准（真实记录优先）
```

### 3.5 前端增强

- 引用卡片区分「记忆」与「知识」：记忆卡片显示 📅 日期 + 来源类型图标，点击跳转到对应页面（时间轴/相册/纪念日/计划）
- 空状态文案："问问你们的回忆，比如：我们上次旅行去了哪？"
- 新增「记忆」Tab 或与现有「知识库」并列，展示已同步的记忆条目（可手动触发同步）

---

## 4. 增量同步设计

### 4.1 同步触发方式

| 方式 | 说明 | 适用场景 |
|------|------|---------|
| **事件驱动**（推荐） | 业务 Service 变更后调用 `MemorySyncService.sync(sourceType, sourceId)` | 实时性要求高的场景（时间轴、纪念日） |
| **定时全量兜底** | 每日扫描各表，对比 `updated_at` 增量同步 | 防止事件丢失、数据修复 |
| **手动触发** | 前端「同步记忆」按钮 / 管理端 | 用户主动刷新 |

### 4.2 事件接入点

在现有业务 Service 的写操作后调用同步（**不侵入业务逻辑**，用 Spring 事件或直接调用）：

| 业务操作 | 现有 Service | 同步动作 |
|---------|-------------|---------|
| 时间轴新增/编辑/删除 | `LoveRecordServiceImpl` | upsert/delete `TIMELINE` 条目 |
| 相册照片上传/编辑/删除 | `AlbumServiceImpl` | upsert/delete `ALBUM_PHOTO` 条目 |
| 纪念日新增/编辑/删除 | `MemorialDayServiceImpl` | upsert/delete `MEMORIAL` 条目 |
| 计划新增/编辑/删除 | `PlanServiceImpl` | upsert/delete `PLAN` 条目 |
| 绑定/修改开始日 | `CoupleBindingServiceImpl` | upsert `COUPLE_INFO` 条目 |

### 4.3 台账表设计

复用 `love_qa_documents` 模式，新增记忆台账表（或复用同一张表 + `source_type` 区分）：

**方案 1（推荐）：复用 `love_qa_documents`**

- 增加 `source_type`、`source_id` 列（可空，仅记忆条目使用）
- 记忆条目 `scope=COUPLE`、`couple_id` 必填
- 优点：复用现有 list/delete/reingest API 与前端知识库管理
- 缺点：知识库与记忆库混在一起，检索需按 `source_type` 区分

**方案 2：独立表 `love_memory_documents`**

- 字段与 `love_qa_documents` 对齐，增加 `source_type`、`source_id`
- 优点：职责清晰，检索隔离天然
- 缺点：需新增一套 CRUD API 与前端管理

> **建议**：先做方案 1（复用台账表 + 加列），检索时按 `source_type` 路由；若后续记忆条目量级大、需要独立管理，再迁移方案 2。

### 4.4 同步幂等与补偿

- 以 `(couple_id, source_type, source_id)` 为业务唯一键，upsert 台账
- 删除业务记录 → 按 `source_id` 删除台账 + Milvus 向量（复用 `deleteVectorsByDocumentId`）
- 同步失败 → 台账 `status=FAILED` + `error_message`，定时任务重试
- 全量兜底任务：扫描各表 `updated_at > last_sync_time` 的记录增量同步

---

## 5. API 设计

### 5.1 新增 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/ai/love-memory/sync` | 手动触发全量/增量同步（body：`sourceType` 可选） |
| GET | `/api/v1/ai/love-memory/entries` | 记忆条目分页列表（按 coupleId） |
| GET | `/api/v1/ai/love-memory/entries/{id}` | 记忆条目详情 |
| DELETE | `/api/v1/ai/love-memory/entries/{id}` | 删除记忆条目（含向量） |
| POST | `/api/v1/ai/love-memory/entries/{id}/reingest` | 重入库 |

### 5.2 复用现有 API

- `POST /api/v1/ai/love-qa/chat` / `/chat/stream`：**扩展**支持双库检索（内部路由），请求体不变
- `GET /api/v1/ai/love-qa/documents`：扩展返回 `sourceType` 字段（记忆条目）

### 5.3 检索请求体扩展（可选）

```json
{
  "message": "我们上次旅行去了哪？",
  "conversationId": "...",
  "coupleId": "...",
  "retrieveMode": "auto"
}
```

`retrieveMode`：`auto`（默认，双库）| `memory`（仅记忆库）| `knowledge`（仅知识库）

---

## 6. 分阶段实施计划

### 阶段 0：数据源接入（P0，最高价值）

**目标**：记忆库可检索，回答个性化回忆问题。

| 任务 | 内容 | 验收 |
|------|------|------|
| P0-1 | `MemoryEntryBuilder`：5 类数据源 → 结构化文本 + metadata | 单测覆盖各数据源模板 |
| P0-2 | `LoveMemoryDocumentService`：台账 CRUD（复用 `love_qa_documents` + `source_type` 列） | 入库/列表/删除可用 |
| P0-3 | `MemorySyncService`：增量同步（事件驱动 + 定时兜底） | 新增时间轴记录 → 向量可检索 |
| P0-4 | 业务 Service 接入同步事件（时间轴/相册/纪念日/计划/绑定） | 各写操作后触发同步 |
| P0-5 | 检索路由：双库并行 + RRF 融合（`MemoryRetrievalRouter`） | 记忆 + 知识混合回答 |
| P0-6 | Prompt 增强：记忆/知识来源区分 | 回答引用真实记录 |
| P0-7 | 前端：引用卡片区分记忆/知识，点击跳转 | 记忆卡片可跳转时间轴/相册 |

**里程碑**：绑定情侣后无需手动入库，可回答"我们上次旅行去了哪？""我们纪念日是什么时候？"

### 阶段 1：体验完善（P1）

| 任务 | 内容 | 验收 |
|------|------|------|
| P1-1 | 记忆条目管理页（列表/删除/重入库/手动同步） | 前端可管理记忆条目 |
| P1-2 | 空状态引导文案优化 | 未同步时提示"同步你们的回忆" |
| P1-3 | 记忆条目更新策略：编辑业务记录 → 向量更新 | 编辑后检索到新内容 |
| P1-4 | 同步失败重试与告警 | FAILED 条目可重试 |

### 阶段 2：检索增强（P2，可选）

| 任务 | 内容 | 验收 |
|------|------|------|
| P2-1 | query 意图路由（方案 B） | 记忆/知识问题分别路由 |
| P2-2 | 时间衰减：旧记忆权重降低（复用 `RagL1Reranker` recency） | 近期记忆优先 |
| P2-3 | 记忆库独立评估集（golden） | 50 条记忆问答评测 |

### 阶段 3：规模化（P3，远期）

| 任务 | 内容 | 验收 |
|------|------|------|
| P3-1 | 记忆条目独立表 `love_memory_documents`（从方案 1 迁移） | 独立管理 |
| P3-2 | 私密消息摘要入库（可选，需隐私评估） | 消息级记忆 |
| P3-3 | 记忆库 Partition 隔离（`couple_{id}`） | 大规模下检索性能 |

---

## 7. 风险与对策

| 风险 | 说明 | 对策 |
|------|------|------|
| 数据隐私 | 记忆库含情侣私密数据 | 严格 `coupleId` filter；私密消息不入库；删除业务记录级联删向量 |
| 数据一致性 | 业务变更与向量不同步 | 事件驱动 + 定时兜底 + 幂等 upsert |
| 检索噪声 | 记忆条目多，topK 被无关记忆占满 | 时间衰减 + category 匹配 + 阈值过滤（复用现有） |
| 冷启动 | 新情侣无数据 | 恋爱开始日 + 纪念日天然存在；引导用户补充 |
| 成本 | 全量同步 Embedding 调用 | 复用 `CachedEmbeddingModel`（需补批量路径缓存）；增量同步只处理变更 |
| 与现有知识库混淆 | 用户分不清记忆/知识 | 前端卡片区分 + Prompt 明确来源 |

---

## 8. 与现有文档的关系

| 文档 | 关系 |
|------|------|
| [love-qa-rag.md](./love-qa-rag.md) | 现有恋爱问答 RAG 实现，本设计在其上扩展双库 |
| [RAG_PHASED_OPTIMIZATION.md](./RAG_PHASED_OPTIMIZATION.md) | 现有检索优化计划，记忆库复用其检索流水线 |
| [decisions.md](./decisions.md) | 表结构/API 变更须同步 |
| [progress.md](./progress.md) | 实施进度记录 |

---

## 9. 会话接续 checklist

新开窗口继续本任务时：

1. 读 [INDEX.md](./INDEX.md) 确认路由
2. 读本文件（设计 + 计划）
3. 读 [progress.md](./progress.md) 确认上次进度
4. 读 [love-qa-rag.md](./love-qa-rag.md) 了解现有实现（改代码时）
5. 按阶段 0 任务顺序实施，每完成一个任务更新 progress.md

---

**最后更新：2026-06-16**（初版设计）