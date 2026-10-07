# Pi Agent：入口与核心流程

本文档是阅读 pi 代码库的参考指南。从进程启动到 LLM 调用、再到事件回流，标注每一段真正需要阅读的源码文件。

## 1. 仓库结构

Pi 是一个 pnpm/npm workspaces monorepo。需要关心的包只有下面这些：

```
packages/
  ai/           LLM 客户端库：provider 注册、模型目录、消息类型、
               流式调用助手（@earendil-works/pi-ai）。
  agent/        与 provider 无关的 agent 循环
              （@earendil-works/pi-agent-core）。
  coding-agent/ CLI 可执行文件 `pi`，以及 AgentSession、模式、工具、
               扩展、会话管理、设置、资源加载器。
  tui/          终端 UI 基础组件（@earendil-works/pi-tui）。
  mcp/          Model Context Protocol 集成。
  client/       SDK 客户端（消费方使用）。
  protocol/     RPC 的线协议。
  server/       RPC 与 SDK 消费方共用的无头服务。
  telemetry/    遥测。
  codemode/     沙箱 / codemode 工具。
  chord/        工具包。
  durable/      持久化执行工具。
  env/          环境变量 / 密钥工具。
```

理解核心流程只需要关注三个包：`ai`、`agent`、`coding-agent`。

## 2. 三层结构、三个入口

运行时由下到上分为三层，每一层都有自己的入口，且被上一层复用。

```
第 1 层：pi-ai              streamSimple()      -> AssistantMessageEventStream
第 2 层：pi-agent-core      Agent + runAgentLoop()  （带状态的封装）
第 3 层：pi-coding-agent    AgentSession + InteractiveMode / print-mode / rpc-mode
```

### 2.1 第 1 层 —— `packages/ai`（LLM 客户端）

纯粹的 provider 代码，不包含 agent loop、工具或状态。每个 provider 注册为一个 `Api`：

- `anthropic-messages`
- `openai-responses`
- `openai-completions`
- `google-generative-ai`
- `google-vertex`
- `bedrock-converse-stream`
- `mistral-conversations`
- `azure-openai-responses`
- `openai-codex-responses`
- `pi-messages`

对外暴露的函数是 `streamSimple(model, context, options)`，定义在 `packages/ai/src/compat.ts`。它返回 `AsyncIterable<AssistantMessageEvent>`，流以 `done` 或 `error` 结束，通过 `result()` 拿到最终的 `AssistantMessage`。

此外还有一个 `models` API 和模型注册表（`packages/ai/src/models-store.ts`、`models.ts`），供 coding-agent 运行时按 `(provider, id)` 查找模型，并可从远端目录刷新。

### 2.2 第 2 层 —— `packages/agent`（`@earendil-works/pi-agent-core`）

与 provider 无关的 agent loop。两个关键文件：

- `packages/agent/src/agent.ts` —— `Agent` 类。负责持有消息记录、模型、工具、steering/follow-up 队列、abort signal，以及事件监听器。
- `packages/agent/src/agent-loop.ts` —— `runAgentLoop()` / `runAgentLoopContinue()`。真正的循环：流式取响应、执行工具、重复。

`Agent` 是被 `AgentSession`（第 3 层）包装的对象。它对外暴露：

- `state.{ systemPrompt, model, thinkingLevel, tools, messages }`
- `prompt(text | AgentMessage[])` —— 开启一轮新的用户输入。
- `continue()` —— 在最后一条消息不是 assistant 时续接（例如工具结果之后的重试）。
- `steer()` / `followUp()` —— 把消息排队，插在当前轮次结束后注入。
- `subscribe(listener)` —— 接收完整的生命周期事件流 `AgentEvent`。
- `abort()` / `waitForIdle()`。

### 2.3 第 3 层 —— `packages/coding-agent`（`pi` 命令）

`pi` 是发布的 CLI。`bin` 字段指向 `dist/bundle/cli.js`，但源码入口是 `packages/coding-agent/src/cli.ts`：

```
cli.ts  -> setupCli(); main(process.argv.slice(2))
main.ts -> main() in src/main.ts
```

其余所有逻辑都构建在 `AgentSession` 之上。

## 3. 进程启动（`main()` 做了什么）

文件：`packages/coding-agent/src/main.ts`。`main()` 的整体流程：

1. **`runAuthCommand(args)`** —— 处理 `pi auth ...` 子命令；打印 API key、执行健康检查、退出。
2. **`handlePackageCommand(args)` / `handleConfigCommand(args)`** —— `pi package ...`、`pi config ...`，一次性管理命令。
3. **`runMcpCommand(args)`** —— `pi mcp ...` 子命令。
4. **`parseArgs(args)`** —— 完整的 CLI 解析（见 `src/cli/args.ts`）。设置 `mode`（`interactive` | `print` | `json` | `rpc`）、`--model`、`--provider`、`--thinking`、`--session`、`--resume`、`--continue`、`--fork`、`--export` 等。
6. **`runMigrations(cwd)`** —— 执行 schema/数据迁移。
7. **`createSessionManager(parsed, cwd, sessionDir, settingsManager)`** —— 把 `--session` / `--resume` / `--continue` / `--fork` / `--session-id` 解析成 `SessionManager`（打开已有或新建，内存中或落盘 JSONL）。
8. **`createAgentSessionRuntime(createRuntime, { cwd, agentDir, sessionManager })`** —— 核心组装步骤，返回 `AgentSessionRuntime`。
   - 内部调用工厂函数，再调用 `createAgentSessionServices()`（`packages/coding-agent/src/core/agent-session-services.ts`），生成 cwd 绑定的服务集合：
     - `ModelRuntime`（`auth.json` + `models.json` + provider 注册表）
     - `SettingsManager`（`settings.json`，项目层 + 全局层叠加）
     - `DefaultResourceLoader`（扩展、技能、提示词模板、主题、上下文文件）
   - 接着 `createAgentSessionFromServices()` 调用 `createAgentSession()`（`packages/coding-agent/src/core/sdk.ts`），真正构造出 `Agent` 和 `AgentSession`。
9. 模式分发（在 `main()` 内）：
   - `appMode === "rpc"` → `runRpcMode(runtime)`（`packages/coding-agent/src/modes/rpc/rpc-mode.ts`）
   - `appMode === "interactive"` → `new InteractiveMode(runtime, ...).run()`（`packages/coding-agent/src/modes/interactive/interactive-mode.ts`）
   - 否则（`print` / `json`） → `runPrintMode(runtime, ...)`（`packages/coding-agent/src/modes/print-mode.ts`）

`AgentSessionRuntime` 这一层是 cwd 绑定的。当 cwd 变化时（例如 `--session` 指向其它项目、`/new`、`/resume`、`/fork`、`/import`），运行时先把当前 `AgentSession` 拆除、跑 `session_shutdown` 扩展事件，再用同一个工厂重建。具体见 `packages/coding-agent/src/core/agent-session-runtime.ts` 中的 `switchSession`、`newSession`、`fork`、`importFromJsonl`。

## 4. `AgentSession` 类

文件：`packages/coding-agent/src/core/agent-session.ts`。它是三种模式（interactive、print、rpc）共用的抽象。

它持有的内容：

- 对底层 `Agent` 的引用（`@earendil-works/pi-agent-core`）。
- `SessionManager`（JSONL 会话文件、分支、压缩、自定义条目 —— 见 `core/session-manager.ts`）。
- `SettingsManager`（项目 + 全局分层设置）。
- `ResourceLoader`（扩展、技能、提示词模板、主题、上下文文件）。
- `ModelRuntime`（认证 + provider 注册表 + 模型目录）。
- `ExtensionRunner`（`core/extensions/runner.ts`）。
- 工具负载状态：内置工具、自定义工具、MCP 工具、工具 allow/exclude 列表、active 与隐藏声明。
- 自动重试、自动压缩、自动会话变量状态。
- `prompt()`（用户输入）、`steer()`、`followUp()`、`abort()`、模型切换、会话导航。

构造过程见 `createAgentSession()`（`core/sdk.ts`）。要点：

- `existingSession = sessionManager.buildSessionContext()` —— 把当前 JSONL 分支回放成 `AgentMessage[]`，作为 `Agent` 的初始状态。
- `thinkingLevel` 从 `thinking_level_change` 条目恢复，回退到设置，再回退到模型能力（`clampThinkingLevel`）。
- 工具选择顺序：`options.tools`（CLI 白名单） → `defaultTools` 设置 → 内置默认（`read, bash, edit, write`）。扩展和自定义工具在 `noTools !== "all"` 时总是开启。
- `convertToLlm: convertToLlmWithBlockImages` —— 当 `blockImages` 开启时丢弃图片部分（作为 prompt-injection 的纵深防御）。
- `streamFn: (model, context, options) => modelRuntime.streamSimple(model, context, requestOptions)` —— 每次 LLM 调用都走这里。
- 若干 provider 流水线钩子：`transformHeaders`（追加 provider attribution）、`onResponse`、`onProviderStreamEvent`、`transformContext`（扩展的 `context` 改写器）。
- `CacheWarmer`（`core/cache-warmer.ts`）在后台保持上一次会话请求的提示缓存条目温热；当 `prepareRequest` 决定换一个模型、或下一次请求让缓存失效时自动取消。

### 4.1 `_installAgent*` 钩子

`AgentSession` 在构造时通过覆盖底层 `Agent` 的四个钩子来织入会话级行为：

- `_installAgentRequestProjection` —— 包装 `prepareRequest`：把最新的压缩投影进上下文、解析虚拟模型、在虚拟选择下重新路由到物理模型。
- `_installAgentNextTurnRefresh` —— 包装 `prepareNextTurn`：当投影出的上下文超过模型的压缩阈值时自动压缩，然后为下一次 assistant 响应重建系统提示。
- `_installAgentBoundaryHooks` —— 包装 `finishTurn`：分发 `turn_end` boundary 事件（扩展可通过 `SessionBoundaryDraft` 追加条目）。
- `_installHiddenDeclarationsProjection` 与 `_installAgentForcedPromptProjection` —— 让声明的工具和强制提示消息与可执行负载保持一致。

这就是会话持久化与回放的关键：agent loop 不感知 JSONL 与压缩，是 `AgentSession` 通过这些钩子在它周围做了投影。

### 4.2 `prompt()` 流程

文件：`packages/coding-agent/src/core/agent-session.ts`，`prompt(text, options)`（约第 1954 行）。高层步骤：

1. `_tryExecuteExtensionCommand(text)` —— 如果输入以 `/` 开头，先尝试斜杠命令（扩展命令优先于 LLM 提示）。
2. `_runInputHandlers(...)` —— 扩展 `input` 钩子可以改写或拦截输入。
3. `_expandSkillCommand(text)` 和 `expandPromptTemplate(...)` —— `/skill:name` 和 `/template` 替换。
4. 若 `isStreaming`，根据 `options.streamingBehavior` 走 `steer()` 或 `followUp()`。
5. `_flushPendingBashMessages()` 与 `_flushPendingCustomMessages()` —— 把上一轮累积的纯上下文消息发出。
6. 校验 model + auth，必要时跑预注入压缩（`_checkCompaction`）。
7. `_extensionRunner.emitBeforeAgentStart(text, images, systemPromptOptions)` —— 扩展可以修改系统提示选项、替换模型等。
8. 组装 `AgentMessage[]` 负载，然后 `await this.agent.prompt(messages)` —— 交还给第 2 层。

## 5. 第 2 层：agent 循环

文件：`packages/agent/src/agent-loop.ts`。`runAgentLoop(prompts, context, config, emit, signal, streamFn)` 是入口。

结构是两层嵌套循环（见 `runLoop`）：

```
外层 while true
    内层 while hasMoreToolCalls || pendingMessages.length > 0
        1. prepareNextTurn（非首次迭代）        -> 可能压缩、刷新 system prompt
        2. declareToolChanges                    （带 toolsAdded/toolsRemoved 的 system 消息）
        3. prepareRequest                        （投影、虚拟模型路由）
        4. streamAssistantResponse               （LLM 调用，产生事件）
        5. executeToolCalls                      （顺序或并行，遵循工具自身的 executionMode）
        6. finishTurn                            （boundary 事件；"continue" / "end"）
        7. 排空 steering 队列
    排空 follow-up 队列
    空则退出
```

要点：

- **事件**（`AgentEvent`）：`agent_start`、`turn_start`、`message_start`、`message_update`、`message_end`、`tool_execution_start`、`tool_execution_update`、`tool_execution_end`、`turn_end`、`agent_end`。按顺序发出；监听器按订阅顺序被等待。
- **工具执行**：`prepareToolCall` 校验参数并调用 `beforeToolCall`，`executePreparedToolCall` 跑工具并通过 `onUpdate` 发出部分结果，`finalizeExecutedToolCall` 调用 `afterToolCall` 允许改写结果。每一步错误都被捕获并以 `isError: true` 的工具结果呈现 —— 工具执行从不抛出到循环。
- **截断处理**：`failToolCallsFromTruncatedMessage` —— 如果 assistant 消息因输出 token 上限被截断（`stopReason === "length"`），里面所有的工具调用都报告为错误，让模型用完整参数重新发起。
- **Steering / follow-up**：`config.getSteeringMessages` 在每轮开头轮询；`getFollowUpMessages` 在内层循环即将退出时轮询。两者默认都是 `one-at-a-time`（只取第一条）；设置中的 `steeringMode` / `followUpMode` 可以切换到 `all`。

`Agent.runWithLifecycle`（第 2 层的封装）把循环变成一次可观测的运行：`isStreaming`、abort signal、错误兜底（产生 `stopReason: "error"` 的 `agent_end`），以及 `agent_settled` 语义。

## 6. 三种模式

三种模式都包装同一个 `AgentSession`，只在 I/O 层各自实现。

### 6.1 Interactive —— `packages/coding-agent/src/modes/interactive/interactive-mode.ts`

`new InteractiveMode(runtime, options).run()`。

- 用 `@earendil-works/pi-tui` 构造 TUI，包括聊天视区、底部状态栏、编辑器、状态指示器和一棵树形组件。
- 订阅 `session.subscribe(...)` 渲染事件：`message_start` / `message_update` / `message_end` → 流式 assistant 组件；`tool_execution_*` → 工具组件；`compaction_*` → 状态消息；等等。
- `keybindings.ts` + `default-keybindings.ts` 定义热键；`core/extensions/runner.ts` 让扩展可以注册额外的按键、底部小部件和 UI 命令。
- 编辑器支持多行输入、`@file` 内容引用（在 `cli/file-processor.ts` 中解析）、bash 模式（`!cmd`）和斜杠命令。提交后调用 `session.prompt(text, options)`。
- bash 模式走 `core/bash-executor.ts`（`executeBashWithOperations`），把 `bash_execution_message` 条目写入会话，独立于 LLM 工具调用。

### 6.2 Print —— `packages/coding-agent/src/modes/print-mode.ts`

`runPrintMode(runtime, { mode: "text" | "json", messages, initialMessage, initialImages })`。

- 订阅会话，在 JSON 模式下把事件流式输出到 stdout；文本模式下只是 `await prompt(initialMessage)`，再打印最终 assistant 文本。
- 用于 `pi -p "..."`（文本）、`pi --mode json "..."`（JSON 事件）、CI / 自动化。
- 处理 SIGTERM / SIGHUP：杀掉派生的子进程，释放运行时。

### 6.3 RPC —— `packages/coding-agent/src/modes/rpc/rpc-mode.ts`

基于 stdio 的 JSON-RPC。线协议见 `packages/coding-agent/src/modes/rpc/rpc-types.ts`。RPC 模式让外部进程驱动同一个 `AgentSession`：事件以 JSON-RPC notification 形式返回；命令（`prompt`、`steer`、`followUp`、`abort`、`set_model`、`cycle_model`、`set_thinking_level`、`compact`、`get_state`、`switch_session`、`new_session`、`fork`、`navigate_tree`、`reload` 等）以 JSON-RPC request 形式进入。

## 7. 工具、扩展、资源加载

这三个子系统共同喂给 `AgentSession` 和 agent loop。

### 7.1 工具 —— `packages/coding-agent/src/core/tools/`

每个工具都是一个 `AgentTool`（`@earendil-works/pi-agent-core`）：

```ts
{
  name: string;
  description: string;
  parameters: JSONSchema;
  execute: (args, onUpdate, signal) => Promise<ToolResult>;
  // 可选：label, executionMode: "parallel" | "sequential"
}
```

内置工具：`read`、`write`、`edit`、`bash`（Windows 上还有 `powershell`）、`grep`、`find`、`ls`。在 `core/tools/index.ts` 创建，由 `createCodingTools`、`createReadOnlyTools` 等暴露（也通过 `core/sdk.ts` 重新导出给 SDK 消费方）。

`withFileMutationQueue`（`core/tools/file-mutation-queue.ts`）串行化文件变更，避免并行工具调用触及同一文件时发生竞态。

工具可通过 `ctx.executeTool(...)` 调用其它工具（嵌套工具调用）。嵌套调用走相同的 `beforeToolCall` / `afterToolCall` 流水线；见 `core/nested-tool-calls.ts` 以及 `AgentSession` 里的 `_executeNestedToolCall`。

系统提示列出每个 active 工具的描述和片段 —— 见 `core/system-prompt.ts`（`buildSystemPrompt`、`diffSystemPromptSections`）。声明但被隐藏的工具（例如通过 `before_agent_start` 移除的）会从声明中过滤，但仍然可执行；见 `_installHiddenDeclarationsProjection`。

### 7.2 扩展 —— `packages/coding-agent/src/core/extensions/`

`ExtensionRunner`（`runner.ts`）是扩展的事件总线。它持有一个 `loadExtensionsResult`，按注册顺序把事件分发给扩展处理器。处理器可以：

- 改写上下文（`context` 事件）。
- 拦截请求和工具调用（`tool_call`、`tool_result`）。
- 改写消息（`message_end`）。
- 添加斜杠命令、按键、底部小部件、自定义工具。
- 注册 provider（`pendingProviderRegistrations`、`pendingNativeProviderRegistrations`、`pendingVirtualModelRegistrations`）。
- 响应会话生命周期事件（`session_start`、`session_shutdown`、`session_before_switch`、`session_before_fork`、`session_before_tree`、`session_before_compact`、`session_compact`、`session_compact_failed`）。
- 通过 `SessionBoundaryDraft` 在 boundary 注入自定义消息与条目。
- 注入 UI 命令（`input`、`ui_command`）。

扩展加载发生在 `DefaultResourceLoader.reload()`（`core/resource-loader.ts`）。扩展是 TypeScript 文件，通过 `jiti` 加载；内置的 CJS/ESM 扩展通过 `core/extensions/index.ts` 的 `builtInExtensions` 注册。

### 7.3 资源加载器 —— `packages/coding-agent/src/core/resource-loader.ts`

`DefaultResourceLoader` 发现扩展、技能、提示词模板、主题和上下文文件。技能位于 `<cwd>/.pi/skills/`，提示词模板位于 `.pi/prompts/`，主题位于 `.pi/themes/`，上下文文件位于 `.pi/context/`，全局对应物位于 `agentDir` 下。CLI 标志可追加额外路径（`--extension`、`--skill`、`--prompt-template`、`--theme`）。

## 8. 会话存储 —— `packages/coding-agent/src/core/session-manager.ts`

`SessionManager` 持有一个 JSONL 会话文件，文件中是一棵分支化的条目树。条目类型（`SessionEntry`）：

- `message` —— user / assistant / toolResult / system / bashExecution / custom
- `model_change` —— 虚拟或物理模型变更
- `compaction` —— 被压缩前缀的摘要
- `branch_summary` —— 挂在分支切点上的摘要
- `context_edit` —— 对目标条目的替换
- `custom` / `custom_message` —— 扩展定义的条目与消息
- `thinking_level_change`、`session_info`

`buildSessionContext()` 把当前叶子回放成 `Message[]`。`buildSessionProjection()` 是规范化的「LLM 应看到的内容」—— 按顺序应用压缩、分支摘要和上下文编辑。

`/fork`（基于条目的分支）、`/new`、`/resume`、`/import` 都走 `AgentSessionRuntime`（`core/agent-session-runtime.ts`），并使用 `SessionManager.forkFrom`、`create`、`open`、`createBranchedSession`。

## 9. 模型与认证 —— `core/model-runtime.ts`、`core/auth-storage.ts`、`core/auth-guidance.ts`

`ModelRuntime` 是 coding-agent 对「我有哪些模型、哪些认证」的视图：

- `models.json`（按 agent-dir）保存本地目录（默认值 + 自定义注册）。
- `auth.json`（按 agent-dir）保存凭据（API key、OAuth token、headers、base URL、自定义环境变量）。
- `ModelRegistry`（`core/model-registry.ts`）合并内置目录 + 扩展注册的 provider + 自定义虚拟模型。
- `streamSimple(model, context, options)` 解析认证与 headers，然后进入第 1 层（`pi-ai`）。

部分 provider 原生支持 OAuth（`packages/ai/src/bun-oauth.ts`、`packages/ai/src/oauth.ts`）。CLI 命令 `pi auth login <provider>` 跑 OAuth 流程；运行时把拿到的凭据存到 `auth.json`，`ModelRuntime` 在过期前自动刷新。

## 10. 下一步读什么

如果只读四个文件，按顺序读下面这些：

1. `packages/coding-agent/src/main.ts` —— 进程启动、模式分发。
2. `packages/coding-agent/src/core/sdk.ts` 中的 `createAgentSession` —— 组装 `Agent` + `AgentSession`。
3. `packages/coding-agent/src/core/agent-session.ts` 中的 `prompt`、`_handleAgentEvent` —— 会话级编排。
4. `packages/agent/src/agent-loop.ts` 中的 `runLoop` —— 内层 LLM / 工具循环。

之后：

- `packages/coding-agent/src/modes/interactive/interactive-mode.ts` —— TUI 集成。
- `packages/coding-agent/src/core/extensions/runner.ts` —— 扩展事件总线。
- `packages/coding-agent/src/core/session-manager.ts` —— JSONL 持久化与分支模型。