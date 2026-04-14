# 最终目标

构建 **LoveSpace** 情侣向全栈应用：可持续迭代的基础工程，覆盖 **认证与资料、情侣绑定、恋爱时间轴、情侣相册、私密消息（HTTP + WebSocket 实时）、共同计划（含子任务、消费流水与预算汇总、进度）、AI 对话与情感类能力（通义千问 / OpenAI）** 等核心链路，支持本地开发、前后端联调与后续扩展。

## 目标拆解

### 基础与认证（已完成方向）

1. 后端：Spring Boot 多模块（`lovespace-common` + `lovespace-user`），Java 21。
2. 用户模型：MyBatis-Plus + MySQL，JWT 无状态认证、Redis 登出黑名单。
3. 前端：React 18 + TS + Vite + Ant Design + 路由 + Zustand + Axios；`/api` 代理至后端。

### 情侣与记录（当前产品范围）

4. **情侣绑定**：邀请 / 接受 / 信息查询 / 恋爱开始日 / 解除；恋爱天数自动计算（业务时区 Asia/Shanghai）。
5. **恋爱时间轴**：记录的 CRUD、分页、可见性（仅自己 / 情侣）、心情枚举、位置与标签 JSON；**`images_json`** 存 **图片与视频** 的 URL 数组（按扩展名区分展示）；回忆推送接口；列表支持 **日期区间筛选**；前端支持 **编辑/删除（作者）**、**Ant Design 分页（默认每页 10 条）**、表单内 **谁可见**（默认双方可见）；大图/视频可走 **公共分片上传**（见 `progress` / `decisions`）；**点赞与评论**（双方在与记录可见性一致的前提下互动，见 `progress`）。
6. **情侣相册**：相册与照片的增删查、本地上传（`albums/` 存储前缀）；**相册内照片列表服务端分页**（默认每页 10 张）；前端等比网格、**底部分页组件**、灯箱预览与收藏（灯箱仅浏览当前页照片）；**大图可与时间轴共用分片上传 API**，合并后 **`POST .../photos/from-url`** 登记记录；**相册改名**、**照片元数据编辑**（描述、拍摄日、地点、标签，见 `progress` / `decisions`）。
7. **私密消息**：表 `private_messages`；REST 发送/列表/已读/撤回/定时创建；**WebSocket** `/ws/chat` 实时推送（落库后广播）；**撤回仅发送后 2 分钟内**；定时任务派发到期消息；前端 **`Chat.tsx`**（会话列表 + 聊天区、一体化输入、历史分页加载、最新消息在底部）。
8. **前端页面**：情侣首页、时间轴、相册、**私密消息**、**共同计划（`/plan`）**、**AI 情感洞察（`/emotion`）**、**AI 情书（`/love-letter`）**；全站 **暖色极简** UI（Inter + Ant Design 主题）。
9. **AI 能力**：大模型对话（通义千问 / OpenAI 可切换）、恋爱记录**情感分析**、**情书生成**（均需情侣上下文与后端密钥配置，见 `memory/decisions.md`）。
10. **纪念日（可选入口）**：情侣维度 **`memorial_days`** + **`/api/v1/memorial-days`**（含倒计时 **`/next`**）；前端 **`Memorial.tsx`** 已实现，**路由/导航可按产品选择挂载**（见 `memory/progress.md`）。

### 质量与运维

11. 编译与构建可重复通过（Maven `mvn -pl lovespace-user -am test`、`npm run build`）。
12. 关键配置与约定写入 `memory/`（含 **`CODING_RULES.md` 编码规则**）与本仓库 `PROJECT_STRUCTURE.md`，便于新会话快速对齐。
