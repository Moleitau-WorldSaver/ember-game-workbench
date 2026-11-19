# Plan #001：一句话生成计划的 Agent 最小切片

## 目标

验证一条最短闭环：用户输入一句话，事件触发 Pi Agent Core + Pi Durable，Agent 捕捉意图并生成一份可审阅的计划。

## 技术决策

- **应用形态**：TypeScript 全栈应用，前端输入界面和后端 Agent API 位于同一项目。
- **事件入口**：`POST /api/requests` 接受一句话，先写入 `request.received`，再触发 Agent 任务。
- **运行时基础**：直接依赖 `@earendil-works/pi-agent-core`、`@earendil-works/pi-durable` 和 `@earendil-works/pi-ai`，在工作台内封装 `TaskRuntimeAdapter`，产品层只依赖自己的任务接口。
- **Pi 分层分工**：
  - 工作台编排层负责读取请求并提取结构化意图。
  - Pi Agent Core 负责运行 Agent loop 和工具调用，Pi Durable 负责持久化会话、任务和检查点。
  - 计划生成 Worker 先作为受限任务运行，后续再扩展为真实文件 Worker。
- **模型提供方**：使用 `@earendil-works/pi-ai` 的 DeepSeek provider，服务端读取 `DEEPSEEK_API_KEY`，模型选择通过 Pi 的模型配置完成。
- **业务事件**：工作台维护 `request.received`、`intent.captured`、`plan.created` 和 `plan.failed`。Pi 运行时事件只作为内部执行记录。
- **存储**：实现 `EventStore` 和 `RunStore` 接口，第一版使用轻量本地实现。

## 实施步骤

### 1. 定义最小领域类型

定义 `Request`、`Intent`、`PlanDraft`、`DomainEvent` 和 `AgentRun`。事件 envelope 包含 `event_id`、`event_type`、`occurred_at`、`request_id`、`caused_by`、`actor` 和 `payload`。

### 2. 实现事件驱动入口

提供以下 API：

- `POST /api/requests` 接收一句话并追加 `request.received`。
- `GET /api/requests/:id` 返回请求、事件和当前计划状态。
- `POST /api/requests/:id/retry` 从最近检查点重新运行失败任务。

请求创建完成后，由事件处理器启动 Agent run。页面通过查询请求状态读取结果，业务状态来自事件投影。

### 3. 接入 Pi Agent Core + Pi Durable 适配层

实现 `TaskRuntimeAdapter` 的最小接口：

- `startRun(requestId)`
- `checkpoint(runId, step)`
- `observe(runId)`
- `resume(runId)`

运行步骤保持固定：读取 `request.received` → 编排层生成 `intent.captured` → 计划生成 Worker 生成 `plan.created`。任一步骤失败都追加 `plan.failed` 并保存检查点。

### 4. 接入 DeepSeek

定义统一 `ModelProvider` 接口。服务端向 DeepSeek 发送原始请求和固定 schema，要求返回结构化意图或计划。解析失败时保留错误上下文并产生 `plan.failed`。本地开发可以使用 mock provider，但生产路径使用 DeepSeek 环境变量配置。

### 5. 完成单页面工作台

页面只保留四个区域：

- 一句话输入框。
- Agent 正在捕捉或生成计划的状态。
- 意图摘要。
- 计划草案和原始请求引用。

不加入游戏画布、会话列表、复杂导航和执行按钮。

### 6. 验证

覆盖以下场景：

- 一句话能产生 `request.received`。
- 编排层输出目标、对象和约束。
- 计划生成 Worker 输出完整计划字段。
- 计划回链到原始请求。
- 失败后可以从检查点重试。
- DeepSeek key 未写入前端包和日志。

## 完成定义

- 用户可以从单页面输入一句话并得到计划草案。
- 事件流能还原请求、意图和计划的因果关系。
- Pi Agent Core + Pi Durable 通过适配层参与实际运行流程。
- 计划生成失败时有真实失败事件和可重试状态。
- 后续 issue 可以在这条链路上增加试玩上下文、Worker 改动和构建验证。

## 后续拆分候选

- Issue #002：把试玩场景和构建版本作为请求上下文。
- Issue #003：让 Worker 生成受限 change set。
- Issue #004：增加验证 Worker 和构建报告。
- Issue #005：接入长任务暂停、恢复和回滚。
