# LoveSpace 编码规则

后续开发请遵守本文件；与 `memory/decisions.md` 中的架构约定一并生效。

---

## （1）代码风格

### 后端（Java / Spring Boot）

- **版本与模块**：Java 21；包名 `com.meng.lovespace.`*；多模块下业务在 `lovespace-user`，公共类型在 `lovespace-common`。
- **分层**：`controller` 只做参数校验与编排；业务在 `service` / `impl`；持久化在 `entity` + `mapper`；入参出参使用 `dto` / record。
- **命名**：类名大驼峰；方法与字段小驼峰；常量全大写下划线；数据库列 snake_case，实体用 `@TableField` 映射。
- **格式**：4 空格缩进；单文件保持与周边现有代码一致；优先使用 Lombok `@Data` 等与项目已有实体一致。
- **Spring MVC**：路径变量、常用 `@RequestParam` 写**显式名称**（如 `@RequestParam("coupleId")`），与父 POM `parameters=true` 策略一致。
- **统一响应**：对外 JSON 使用 `ApiResponse`；业务错误用领域异常 + 对应 `*ControllerExceptionHandler`，避免在 Controller 散落魔法数字。

### 前端（TypeScript / React）

- **栈**：React 18 函数组件 + Hooks；类型明确，避免滥用 `any`；路由与布局分离。
- **UI**：Ant Design 主题由 `src/theme/antdTheme.ts` 与 `ConfigProvider` 统一；页面级布局类名与 `index.css` 中 `ls-`* 工具类保持一致。
- **请求**：经 `services/http.ts` 的 Axios 实例；API 路径与后端 `/api/v1` 对齐，不在业务组件里硬编码完整域名（除 `resolveMediaUrl` 等工具）。
- **状态**：认证、情侣等跨页状态用 Zustand；局部列表状态用组件内 `useState` / `useCallback` 即可。
- **格式**：遵循仓库已有 ESLint / Prettier 配置；Tailwind 工具类顺序可读即可，不必强行排序插件。

---

## （2）注释

- **后端**
  - **类、公开接口**：公共 `Controller`、`Service` 接口、领域模型中与业务强相关的类，使用 **JavaDoc**（`/** ... */`）说明职责、关键参数与异常语义。
  - **复杂逻辑**：非一眼能看懂的校验、状态机、并发或日期规则，用**简短行注释**说明「为什么」，避免复述代码「做什么」。
  - **不宜**：显而易见的 getter/setter、重复文档式的废话注释；不要删掉他人已有注释除非已失效。
- **前端**
  - **导出组件 / 工具**：对非自解释的 props 或副作用，用 **JSDoc** 或文件头一两句说明用途。
  - **复杂交互**：如拖拽刷新、乐观更新回滚等，在关键 `useEffect` / 事件处理旁注明业务意图。

---

## （3）日志

### 后端（SLF4J + `@Slf4j`）

- **级别**
  - `debug`：分页查询、列表条件、高频只读路径（生产可按包级关闭）。
  - `info`：写操作成功（创建/更新/删除）、上传完成、状态迁移等**可审计**节点。
  - `warn`：业务异常进入 `ExceptionHandler` 前或预期外但可恢复的情况。
  - `error`：未捕获异常或需告警的失败（慎用，避免刷屏）。
- **内容**：包含**业务主键**（如 userId、albumId、无敏感信息的 trace 信息）；**禁止**打印密码、token、完整证件号等敏感字段。
- **风格**：与现有 `TimelineController`、`AlbumServiceImpl` 等一致，使用占位符 `{}`，避免字符串拼接。

### 前端

- **生产环境**：不依赖 `console.log` 做功能；必要时仅 `console.error` 上报开发期问题，合并前尽量移除。
- **用户提示**：成功/失败以 Ant Design `message` 等与现有页面一致的方式呈现。

---

## 修订

- 本文件随项目演进可增补小节；重大风格变更应同步更新 `memory/decisions.md` 并在 PR/提交说明中点名。

