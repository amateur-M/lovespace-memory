# 最终目标

构建 **LoveSpace** 情侣向全栈应用：可持续迭代的基础工程，覆盖 **认证与资料、情侣绑定、恋爱时间轴、情侣相册** 等核心链路，支持本地开发、前后端联调与后续扩展。

## 目标拆解

### 基础与认证（已完成方向）

1. 后端：Spring Boot 多模块（`lovespace-common` + `lovespace-user`），Java 21。
2. 用户模型：MyBatis-Plus + MySQL，JWT 无状态认证、Redis 登出黑名单。
3. 前端：React 18 + TS + Vite + Ant Design + 路由 + Zustand + Axios；`/api` 代理至后端。

### 情侣与记录（当前产品范围）

4. **情侣绑定**：邀请 / 接受 / 信息查询 / 恋爱开始日 / 解除；恋爱天数自动计算（业务时区 Asia/Shanghai）。
5. **恋爱时间轴**：记录的 CRUD、分页、可见性（仅自己 / 情侣）、心情枚举、位置与标签 JSON、多图 URL（`images_json`）；回忆推送接口；列表支持 **日期区间筛选**。
6. **情侣相册**：相册与照片的增删查、本地上传（`albums/` 存储前缀）；前端相册列表、等比网格缩略图、灯箱预览与收藏。
7. **前端页面**：情侣首页、时间轴（按月分组、分页加载、下拉刷新、日期区间、记录表单含图片上传）、**相册**（`Album.tsx` 与相关组件）；全站 **暖色极简** UI（Inter + Ant Design 主题）。

### 质量与运维

8. 编译与构建可重复通过（Maven `mvn -pl lovespace-user -am test`、`npm run build`）。
9. 关键配置与约定写入 `memory/`（含 **`CODING_RULES.md` 编码规则**）与本仓库 `PROJECT_STRUCTURE.md`，便于新会话快速对齐。
