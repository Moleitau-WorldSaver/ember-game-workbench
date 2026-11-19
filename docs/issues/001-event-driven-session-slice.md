# Issue #001：一句话生成计划的 Agent 最小切片

## 类型

Feature / Vertical Slice

## 状态

Draft

## 背景

工作台的第一价值是让开发者只说一句话，Agent 自动捕捉其中的意图，并产出一份可以继续执行的计划。第一版先验证这条最短链路，暂不引入试玩会话、游戏运行时和真实文件改动。

实现采用 TypeScript 全栈，基于 Pi Agent Core + Pi Durable 做二次开发。事件是流程的驱动语言，Pi 负责可恢复的 Agent 运行过程，工作台负责请求、意图和计划的数据语义。模型提供方使用 DeepSeek。

## 用户故事

作为独立游戏开发者，我希望输入一句自然语言描述，系统自动理解我要改变什么并生成一份清晰的执行计划，这样我可以先审阅计划，再决定是否进入后续实现。

## 最小范围

1. 提供一个单句输入入口。
2. 用户提交一句话后，系统追加 `request.received` 事件。
3. Pi Agent Core + Pi Durable 驱动一次 Agent 处理流程。
   - 工作台编排层读取请求，提取目标、对象、约束和待确认信息。
   - Pi Agent Core 运行 Agent loop，Pi Durable 保存会话和任务检查点。
   - 计划生成 Worker 将结构化意图整理为一份计划草案。
4. 系统追加 `intent.captured` 和 `plan.created` 事件。
5. 页面展示原始输入、Agent 捕捉到的意图和计划草案。
6. DeepSeek API 通过服务端运行时环境变量接入。

## 计划输出结构

计划草案至少包含：

- 目标
- 需要修改的对象
- 执行步骤
- 成功标准
- 验证方式
- 风险或需要确认的问题

计划必须保留原始请求引用，便于后续追溯。

## 事件协议

本 issue 至少实现以下事件：

- `request.received`
- `intent.captured`
- `plan.created`
- `plan.failed`

事件至少包含事件类型、时间、请求标识、产生者、因果引用和 payload。原始输入保持不变，意图和计划通过新事件表达。

## 非目标

- 不接入 Phaser、Godot、Unity 或 Unreal。
- 不创建或管理试玩会话。
- 不要求一次输入多条想法。
- 不执行计划，不修改项目文件，不生成构建。
- 不做语音、截图、多 Worker 编排和多人协作。
- 不把 Pi 的内部事件直接当作工作台业务事件。

## 验收标准

- 输入一句自然语言后，可以看到对应的请求事件。
- 编排层能输出结构化意图，至少包括目标和对象。
- 计划生成 Worker 能基于结构化意图输出计划草案。
- 页面能同时展示原始输入、意图和计划。
- 计划可以回链到原始请求。
- DeepSeek key 只存在服务端运行时环境变量中。
- Agent 失败时追加 `plan.failed`，页面显示失败状态和原因。
- Pi 运行过程通过适配层调用，后续可以从检查点恢复。

## 技术约束

- 全栈 TypeScript。
- 基于 Pi Agent Core + Pi Durable 二次开发，并通过适配层隔离 Pi API 的变化。
- 使用事件驱动流程，不用页面按钮直接串联业务状态。
- DeepSeek provider 只在服务端调用。
- 先使用可替换的轻量事件存储，后续再接入正式数据库。

## 相关文档

- `docs/01-product-brief.md`
- `docs/04-event-driven-agent-model.md`
- `docs/09-open-source-landscape-and-selection.md`
