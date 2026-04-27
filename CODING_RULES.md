# LoveSpace 编码规则

> 后续开发请遵守本规范；与 `decisions.md` 架构约定一并生效。

## 代码风格

### 后端（Java / Spring Boot）
- **版本与模块**：Java 21；包名 `com.meng.lovespace.*`；业务在 `lovespace-user`，公共类型在 `lovespace-common`
- **分层**：controller（参数校验+编排）→ service/impl（业务）→ entity+mapper（持久化）；入参出参用 dto/record
- **命名**：类大驼峰、方法字段小驼峰、常量全大写下划线；数据库列 snake_case，实体用 `@TableField` 映射
- **格式**：4空格缩进；优先使用 Lombok `@Data`；Spring MVC 路径变量与 `@RequestParam` 写显式名称（如 `@RequestParam("coupleId")`）
- **响应**：对外 JSON 用 `ApiResponse`；业务错误用领域异常（继承 `lovespace-common` 的 `ApiBusinessException`），由 `UserGlobalExceptionHandler` 等 `@RestControllerAdvice` 统一转换

### 前端（TypeScript / React）
- **栈**：React 18 函数组件 + Hooks；类型明确避免滥用 `any`；路由与布局分离
- **UI**：Ant Design 主题由 `src/theme/antdTheme.ts` 与 `ConfigProvider` 统一；页面级布局类名与 `index.css` 中 `ls-*` 工具类一致
- **列表操作**：时间轴 `TimelineItem`、共同计划 `PlanListItem` 采用右上角 ⋯（`MoreOutlined`）+ `Dropdown`；菜单项编辑/删除用图标；删除前 `Modal.confirm`
- **请求**：经 `services/http.ts` 的 Axios 实例；API 路径与后端 `/api/v1` 对齐；AI 长耗时接口（情感分析、情书）timeout ≥120s；认证接口使用 **手机号**（`login`/`register`），`User` 含 `phone`，邮箱在资料页 `updateProfile` 维护
- **上传**：大文件分片用 `services/mediaChunkUpload.ts`（`/api/v1/media/uploads`，target: TIMELINE|ALBUM）；单文件直传用 `timeline.uploadTimelineMedia` / `album.uploadAlbumPhoto`
- **分页**：时间轴记录与相册照片等长列表统一服务端分页 + Ant Design Pagination，默认每页 10 条
- **WebSocket**：原生 `WebSocket`；URL 由 `VITE_API_BASE_URL` 推导 ws/wss；token 以 query 传递，禁止日志打印完整 token
- **状态**：认证、情侣等跨页状态用 Zustand；局部列表用组件内 useState/useCallback；`authStore` 使用 `authHydrated`，在 `AppLayout` 完成 hydrate 后再挂载 `Outlet`
- **样式**：入口 `StyleProvider hashPriority="high"`；`prefers-reduced-motion` 勿对全局 `*` 施加会破坏 body 上浮层动画的规则
- **纪念日照片墙**：相册 URL 经 `utils/mediaUrl.resolveMediaUrl`；大图预览使用 Ant Design `Image` 的 `preview`（列表与预览 URL 区分见 `decisions.md`「纪念日页面」）

## 注释规范

### 后端
- **类与公开接口**：公共 Controller、Service 接口、业务强相关领域模型使用 JavaDoc 说明职责、关键参数与异常语义
- **复杂逻辑**：非一眼能看懂的校验、状态机、并发或日期规则，用简短行注释说明「为什么」，避免复述代码「做什么」
- **不宜**：显而易见的 getter/setter、重复文档式的废话注释；不要删掉他人已有注释除非已失效

### 前端
- **导出组件/工具**：对非自解释的 props 或副作用，用 JSDoc 或文件头一两句说明用途
- **复杂交互**：如拖拽刷新、乐观更新回滚等，在关键 useEffect/事件处理旁注明业务意图

## 日志规范

### 后端（SLF4J + @Slf4j）
- **级别**：debug（分页查询、高频只读）、info（写操作成功、上传完成、状态迁移等可审计节点）、warn（业务异常进入 ExceptionHandler 前或预期外但可恢复）、error（未捕获异常或需告警的失败，慎用）
- **内容**：包含业务主键（userId、albumId 等无敏感信息的 trace 信息）；禁止打印密码、token、完整证件号等敏感字段
- **风格**：使用占位符 `{}`，避免字符串拼接

### 前端
- **生产环境**：不依赖 console.log 做功能；必要时仅 console.error 上报开发期问题，合并前尽量移除
- **用户提示**：成功/失败以 Ant Design message 等与现有页面一致的方式呈现

## 修订

本文件随项目演进可增补小节；重大风格变更应同步更新 `decisions.md` 并在 PR/提交说明中点名。

