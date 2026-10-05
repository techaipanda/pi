# Pi 运行流程图

> 本文档描绘 Pi 的核心运行时流程：**用户发送一条消息后，Pi 如何构建上下文、调用模型、流式接收响应、执行工具、持久化结果、决定下一步走向终止**。

涵盖三种主要运行模式：交互模式（TUI）、打印 / JSON 模式、RPC 模式。共有的核心是同一套 **agent loop**，不同的只是外壳。

---

## 1. 全局视角：一次对话的端到端流程

```mermaid
flowchart TB
    Start([用户输入]) --> Step0[Step 0: 启动 Pi]
    Step0 --> Step1[Step 1: 加载项目 / 信任决定 / 扩展 / 主题 / Skills]
    Step1 --> Step2[Step 2: 恢复或创建 Session]
    Step2 --> Step3[Step 3: 编辑器获取 prompt]
    Step3 --> Step4[Step 4: 提交 prompt<br/>进 active branch]
    Step4 --> Step5[Step 5: prepareRequest 钩子]
    Step5 --> Step6[Step 6: transformContext]
    Step6 --> Step7[Step 7: convertToLlm]
    Step7 --> Step8[Step 8: 拼装 LLM 请求<br/>sys + history + tools + settings]
    Step8 --> Step9[Step 9: models.streamSimple]
    Step9 --> Step10[Step 10: Auth 解析<br/>getAuth + token refresh]
    Step10 --> Step11[Step 11: Provider 流式发送]
    Step11 --> Step12{Step 12: stopReason?}

    Step12 -- toolUse --> Step13[Step 13: 校验 + beforeToolCall 钩子]
    Step13 --> Step14[Step 14: 串行/并行 execute 工具]
    Step14 --> Step15[Step 15: afterToolCall 钩子]
    Step15 --> Step16{Step 16: 全部结果 terminate?}
    Step16 -- 是 --> Step19[Step 19: turn_end]
    Step16 -- 否 --> Step17[Step 17: 持久化 toolResult]
    Step17 --> Step18[Step 18: 查 Steering / Follow-up 队列]
    Step18 -- 有队列 --> Step6
    Step18 -- 无 --> Step19

    Step12 -- "stop / length / error / aborted" --> Step20[Step 20: finishTurn 钩子]
    Step20 --> Step21{Step 21: finishTurn 决策}
    Step21 -- continue --> Step6
    Step21 -- end --> Step19

    Step19 --> Step22{Step 22: run 结束?}
    Step22 -- 还有 pending/queued --> Step6
    Step22 -- 无 --> Step23[Step 23: agent_end]
    Step23 --> Step24[Step 24: 等待 subscribers]
    Step24 --> Step25[Step 25: 渲染下一帧 / 等待下一条输入]
    Step25 --> Step3
```

---

## 2. 单 turn 的细节时序（interactive mode）

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Editor as Editor (TUI)
    participant Session as Session
    participant Agent as Agent (pi-agent-core)
    participant Hooks as Hook Chain
    participant Tools as Tool Registry
    participant Mcp as MCP / Codemode
    participant Models as pi-ai Models
    participant Provider as Provider
    participant Disk as JSONL on disk

    User->>Editor: 输入 + 附件 (image / @file)
    Editor->>Session: submit(text + attachments)
    Session->>Disk: append user entry<br/>(active branch leaf)
    Session->>Agent: prompt(text)
    Agent-->>Session: agent_start
    Session->>Hooks: prepareRequest({ context })
    Hooks-->>Session: 注入 canonical messages
    Agent->>Agent: transformContext()
    Agent->>Agent: convertToLlm()
    Agent-->>Session: turn_start + message_start(user) + message_end(user)
    Agent->>Models: streamSimple(model, ctx, opts)
    Models->>Models: getAuth(provider) — refresh OAuth if needed
    Models->>Provider: POST /messages (transformed headers)
    loop stream chunks
        Provider-->>Models: SSE chunks
        Models-->>Agent: start / text_delta / thinking_delta / toolcall_delta
        Agent-->>Session: message_update
    end
    Models-->>Agent: done(reason=toolUse)
    Agent->>Tools: validate args (TypeBox)
    Agent->>Hooks: beforeToolCall({ toolCall, args, context })
    Hooks-->>Agent: allow | block
    Agent->>Tools: execute(args)
        Tools->>Mcp: (optional)
        Mcp-->>Tools: result
    Tools-->>Agent: result
    Agent->>Hooks: afterToolCall({ result })
    Hooks-->>Agent: result | terminate
    Agent->>Disk: append assistant + toolResult
    Agent-->>Session: turn_end(message, toolResults)

    Note over Agent,Session: finishTurn() 钩子
    Session->>Hooks: finishTurn({ message, toolResults })
    Hooks-->>Session: { continue } | { end } | undefined
```

---

## 3. Agent Loop 内部状态机

`agentLoop()` 的核心是下面的状态机。每个状态之间的转换由 LLM 事件、工具结果、队列消息、用户中止共同驱动。

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Prompting: prompt() / continue()
    Prompting --> RequestPrep: prepareRequest 钩子
    RequestPrep --> Streaming: provider.streamSimple()
    Streaming --> Preflight: stopReason=toolUse
    Streaming --> AwaitingDecision: stopReason=stop|length|error|aborted
    Preflight --> Executing: preflight allow
    Preflight --> Executing: preflight block → 仍当成 result 进入
    Executing --> ToolPost: each execute() 完成
    ToolPost --> Executing: 还有工具没跑
    ToolPost --> TurnEnd: 全部 finalize
    TurnEnd --> AwaitingDecision: finishTurn 钩子
    AwaitingDecision --> Streaming: finishTurn → continue
    AwaitingDecision --> SteeringPoll: finishTurn → undefined
    AwaitingDecision --> Idle: finishTurn → end
    SteeringPoll --> Streaming: steering / follow-up 队列有消息
    SteeringPoll --> Idle: 队列空且无 abort
    Idle --> [*]: waitForIdle 退出
```

---

## 4. 流式响应的事件序列（pi-ai 视角）

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Agent / Client
    participant Models as Models
    participant Auth as Auth Resolver
    participant P as Provider

    Agent->>Models: streamSimple(model, ctx, opts)
    Models->>Auth: getAuth(model.provider)
    Auth->>Auth: CredentialStore.modify(<br/>读取 + refresh OAuth if needed)
    Auth-->>Models: auth headers + apiKey
    Models->>Models: mergeProvider.headers +<br/>model.headers +<br/>options.headers +<br/>transformHeaders
    Models->>P: POST /messages (stream=true)
    P-->>Models: stream chunks
    Models-->>Agent: start
    loop content blocks
        Models-->>Agent: text_start / thinking_start / toolcall_start
        Models-->>Agent: text_delta / thinking_delta / toolcall_delta
        Models-->>Agent: text_end / thinking_end / toolcall_end
    end
    Models-->>Agent: done(reason)
    Note right of Agent: stopReason ∈<br/>"stop" | "length" | "toolUse" | "error" | "aborted"
    Models-->>Agent: final AssistantMessage
```

完整事件类型参考：

```text
start                 → 流开始（partial 是初始结构）
text_start            → 文本块开始
text_delta            → 文本 chunk
text_end              → 文本块结束
thinking_start        → 思考块开始
thinking_delta        → 思考 chunk
thinking_end          → 思考块结束
toolcall_start        → 工具调用开始
toolcall_delta        → 工具参数 chunk
toolcall_end          → 工具调用完成
done                  → 流结束，带 reason
error                 → 错误事件，带 reason
```

---

## 5. Tool 执行流程

```mermaid
flowchart TB
    Start([turn 中收到 toolUse]) --> Validate[Step 1: validateToolCall<br/>TypeBox schema 校验]
    Validate -- invalid --> ErrorResult[构造 toolResult<br/>isError=true, 给 LLM 重试]
    Validate -- valid --> Before[Hook: beforeToolCall<br/>{ toolCall, args, context }]
    Before -- block --> BlockResult[返回 block 结果 + terminate?]
    Before -- allow --> Mode{Step 2: 工具执行模式}
    Mode -- sequential --> ExecSeq[按序 execute]
    Mode -- parallel --> Preflight[串行 preflight]
    Preflight --> ExecPar[并行 execute]
    ExecSeq & --> StreamCheck{是否 streaming?}
    StreamCheck -- 是 --> OnUpdate[通过 onUpdate 流式结果]
    StreamCheck -- 否 --> ExecWire[直接结果]
    OnUpdate & --> After[Hook: afterToolCall<br/>{ result, isError }]
    After -- terminate --> FinalResult[Final toolResult + terminate 标记]
    After -- 改写 --> Rewrite[Final toolResult 改写]
    After -- 默认 --> Block
    BlockResult & FinalResult & Rewrite --> ToolMsg[Step 5: 构造 toolResult message]
    ToolMsg --> Persist[Step 6: 持久化到 transcript]
```

---

## 6. Steering / Follow-up 队列行为

```mermaid
flowchart LR
    subgraph WhileRunning["运行中"]
        S1["agent.prompt('X')<br/>入 steering 队列"]
        S2["agent.steer({...})<br/>入 steering 队列"]
        S3["agent.followUp({...})<br/>入 follow-up 队列"]
    end

    S1 & S2 --> SteeringPool[Steering Pool]
    S3 --> FollowupPool[Follow-up Pool]

    SteeringPool -- turn_end 后 --> RunNext{steeringMode}
    RunNext -- "one-at-a-time" --> TakeOne[取 1 条进入下一 turn]
    RunNext -- "all" --> TakeAll[全部进入下一 turn]

    FollowupPool -- run 结束 / queue 空 --> RunNext2{followUpMode}
    RunNext2 -- "one-at-a-time" --> RunOnce[取 1 条启动新 run]
    RunNext2 -- "all" --> RunAll[全部启动新 run]
```

> - **Steering**：当前助手 turn 完成后注入（一次消息，立刻采）
> - **Follow-up**：run 完全结束才进入（一次消息，新 run 开始时采）

---

## 7. 不同运行模式的入口流

### 7.1 Interactive (TUI)

```mermaid
sequenceDiagram
    actor User
    participant Shell as pi (binary)
    participant Tui as TuiLoop
    participant Edit as Editor
    participant Agent as Agent
    participant Disk as JSONL

    User->>Shell: pi
    Shell->>Tui: 启动 TUI，绑定 stdio
    Tui->>Disk: 恢复 / 新建 session
    Tui->>Edit: focus
    loop main loop
        User->>Edit: 输入 + Enter
        Edit->>Tui: submit(text)
        Tui->>Agent: prompt(text)
        Agent-->>Tui: events
        Tui->>Tui: 差分渲染
        User->>Tui: 按键 (Ctrl+T / Ctrl+L / Escape)
        Tui->>Agent: 切换 thinking / follow-up / abort
    end
```

### 7.2 Print Mode (`-p`)

```mermaid
sequenceDiagram
    actor User
    participant Cli as pi -p "..."
    participant Agent as Agent
    participant Models as Models
    participant Stdout as stdout

    User->>Cli: pi -p "fix the build"
    Cli->>Cli: 加载项目、解析 args
    Cli->>Agent: 创建 Session + prompt
    Agent->>Models: stream()
    Models-->>Agent: events
    Agent-->>Cli: agent_end
    Cli->>Stdout: 输出最终 assistant text
    Cli-->>User: exit code
```

### 7.3 JSON Mode (`--mode json`)

```mermaid
sequenceDiagram
    actor User
    participant Cli as pi --mode json -p "..."
    participant Agent as Agent
    participant Stdout as stdout (JSONL)

    User->>Cli: pi --mode json -p "..."
    Cli->>Agent: prompt
    Agent-->>Cli: events
    Cli->>Stdout: 每事件一行的 JSONL
    Cli-->>User: agent_end
```

### 7.4 RPC Mode (`--rpc`)

```mermaid
sequenceDiagram
    actor Caller
    participant Stdin as stdin (JSONL)
    participant Cli as pi --rpc
    participant Agent as Agent
    participant Stdout as stdout (JSONL)

    loop RPC loop
        Caller->>Stdin: 命令 (prompt / setModel / abort / setSteeringMode / ...)
        Stdin->>Cli: parseCommand
        Cli->>Agent: 执行命令
        Agent-->>Cli: events
        Cli->>Stdout: response / event lines
        Stdout-->>Caller: 拉取
    end
```

### 7.5 SDK (in-process)

```mermaid
sequenceDiagram
    participant App as Application
    participant Session as createAgentSession()
    participant Agent as Agent

    App->>Session: createAgentSession({ model, tools, ... })
    Session->>Agent: 实例化
    App->>Session: subscribe(event => render(event))
    App->>Session: prompt("...")
    Agent-->>Session: events
    Session-->>App: events (subscription)
    App->>App: 等 agent_end / waitForIdle
    App->>Session: abort() | dispose()
```

---

## 8. 持久化与会话生命周期

### 8.1 JSONL Session 文件结构

```mermaid
flowchart LR
    subgraph Session["Session JSONL File"]
        E1["Entry 1<br/>{ id: parent=null, type: user }"]
        E2["Entry 2<br/>{ id, parent=E1, type: assistant }"]
        E3["Entry 3<br/>{ id, parent=E2, type: toolResult }"]
        E4["Entry 4<br/>{ id, parent=E2, type: assistant (branch) }"]
        E5["Entry 5<br/>{ id, parent=E4, type: user }"]
    end
    Active["active leaf = Entry 5"]
    Branch["active branch = E1 → E2 → E4 → E5"]
```

- 每个 entry 有 `parentId`，可建树状分支
- `current` 指向 active leaf，session 上下文 = active branch
- Fork / Clone 复制选定 branch 到新文件

### 8.2 写入流

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Agent
    participant Session as Session Manager
    participant Jsonl as JSONL File
    participant Buf as 写入缓冲

    Agent->>Session: 产生 entry
    Session->>Buf: append (in-memory)
    Buf-->>Session: ack
    Session->>Jsonl: flush on turn_end / debounced
    Jsonl-->>Session: ok | retry on EAGAIN
    Note right of Session: 原子写入<br/>保证 entry 的<br/>parent 总是存在
```

---

## 9. Compaction 流程

当上下文超过 `contextWindow - reserveTokens` 时触发，可手动 `/compact` 或自动。

```mermaid
flowchart TB
    Start([context 超阈值]) --> Type{触发类型}
    Type -- 手动 /compact --> Manual[CompactTask]
    Type -- 自动 / overflow --> Auto[GenerationTask 检测]
    Manual & Auto --> Snap[Step 1: 收集需保留的 entry<br/>新 reset 之后的所有 entry]
    Snap --> Summary[Step 2: 调摘要 LLM<br/>按 keepRecentTokens 截断]
    Summary --> Place[Step 3: 放置 pi.compaction entry<br/>包含 summary + head 引用]
    Place --> Done([下次请求用 pi.compaction 之后的部分])

    Overflow -.->|provider 拒绝后 -->|手动 retry| Manual
    Overflow -.->|一次自动 compact| Auto
```

每次 compact 在与前 reset 之间的「stale」 summary 会 settle 为 `stale`，留下最远的 cut 生效。

---

## 10. Abort 行为流

```mermaid
flowchart TB
    A[user 按 Esc / agent.abort] --> AbortSignal[设置 abort signal]
    AbortSignal --> Stream[正在流式响应中断]
    Stream --> ToolCancel[在跑的 tool 也被 signal 取消]
    ToolCancel --> PushBack[Queue 中 messages 退到编辑器]
    PushBack --> AgentEnd[发 agent_end]
    AgentEnd --> Render[UI 渲染终止状态]
```

---

## 11. 启动流程（Pi CLI）

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Shell as pi binary
    participant Args as Args parser
    participant Trust as Project Trust
    participant Cfg as Config
    participant Ext as Extension Loader
    participant Skills as Skills Loader
    participant Themes as Theme Loader
    participant Tui as TuiLoop / RpcLoop
    participant Session as Session

    User->>Shell: pi [args]
    Shell->>Args: parse args / flags
    Args->>Trust: 检查 .pi + 项目根
    Trust-->>Args: 信任决议
    Args->>Cfg: 加载 settings.json
    Cfg->>Ext: 加载扩展 (TypeScript 模块)
    Ext->>Skills: 枚举 skills 描述
    Ext->>Themes: 加载主题
    Args->>Session: 恢复 / 新建
    Session->>Tui: 渲染 + 设置 focus
    Tui-->>User: 进入主循环
```

---

## 12. Token 用量与计费累计

```mermaid
flowchart LR
    Stream[Provider stream] --> Usage[AssistantMessage.usage<br/>input/output/cacheRead/cacheWrite]
    Usage --> Accum[Session.usage 累计]
    Accum --> PerModel[per (provider, modelId)]
    Accum --> PerTool[per tool name<br/>(toolResult.reportUsage)]
    Footer[Footer UI] -->|render| PerModel
```

---

## 13. MCP 工具调用流

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Agent
    participant Wrapped as MCP-wrapped AgentTool
    participant Client as McpClient
    participant Transport as Stdio / StreamableHTTP
    participant Srv as MCP Server

    Agent->>Wrapped: toolCall(callMcpTool, args)
    Client->>Transport: initialize / tools/list 缓存
    Wrapped->>Client: callTool(name, args)
    Client->>Transport: JSON-RPC tools/call
    Transport->>Srv: POST + frame
    Srv-->>Transport: response + progress
    Transport-->>Client: result
    Client-->>Wrapped: CallToolResult
    Wrapped->>Wrapped: toLlmContent(result)
    Wrapped-->>Agent: content blocks
```

MCP tool result 通过 `toLlmContent` 转为文本 / 图像块；`structuredContent` 透传；`isError: true` 走 LLM 重试。

---

## 14. Codemode 工具调用流

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Agent
    participant CmTool as codemode AgentTool
    participant Sandbox as CodemodeSandbox
    participant Worker as Worker thread
    participant VM as QuickJS VM
    participant Tools as 注入的工具

    Agent->>CmTool: toolCall(code)
    CmTool->>Sandbox: new + execute(code, signal)
    Sandbox->>Worker: 启动 worker thread
    Worker->>VM: 创建 wasm 实例 + 桥接
    VM->>Worker: 调用 tools.x(args)
    Worker->>Tools: relay (host bridge)
    Tools-->>Worker: result
    Worker-->>VM: JSON 结果
    VM-->>Worker: text() / image() / return
    Worker-->>Sandbox: output + result
    Sandbox-->>CmTool: { output, value, calls }
    CmTool-->>Agent: content + isError
```

模型代码里的嵌套调用（如 `await tools.read({...})`）**永远不进入 LLM 上下文**；只有最终 `output` / `return` 进。

---

## 15. 与 RPC 模式的协同

```mermaid
sequenceDiagram
    autonumber
    participant Caller
    participant Stdin
    participant Cli as pi --rpc
    participant Agent
    participant Stdout

    Caller->>Stdin: {"cmd": "prompt", "text": "fix build"}
    Stdin->>Cli: parsed command
    Cli->>Agent: prompt(text)
    Agent-->>Cli: events
    Cli->>Stdout: {"type": "event", "event": {...}}
    Cli->>Stdout: {"type": "response", "id": "1", "ok": true}
    Caller->>Stdin: {"cmd": "setModel", "provider": "anthropic", "modelId": "claude-sonnet-4-6"}
    Cli->>Agent: state.model = ...
    Caller->>Stdin: {"cmd": "abort"}
    Cli->>Agent: agent.abort()
    Agent-->>Stdout: event batch
```

RPC 命令集覆盖：prompt、setModel、setThinkingLevel、abort、setSteeringMode、setFollowUpMode、setTools、订阅事件等。

---

## 16. 异常 / 错误恢复流

```mermaid
flowchart TB
    Event[LLM event / tool error] --> Reason{stopReason}
    Err["error" / "aborted"] --> StreamErr[流式错误 → partial AssistantMessage]
    StreamErr --> Persist[持久化 partial + errorMessage]
    Persist --> Decide{finishTurn?}
    Decide -- continue --> Retry[同上下文再发]
    Decide -- end --> Idle
    Retry --> ApiErr{API 错误?}
    ApiErr -- 401 --> Refresh[OAuth refresh]
    Refresh --> Resume[重发]
    ApiErr -- 429 --> Backoff[指数退避]
    Backoff --> Resume
    ApiErr -- context overflow --> Compact[触发 compaction]
    Compact --> Resume
    ApiErr -- abort --> Idle
```

---

## 17. 子会话与 Background subagent（durable）

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Root as Root Conversation
    participant Gen as pi.generation
    participant Sub as pi.subagent tool
    participant Child as child conversation
    participant Reporter as reporter (background)

    User->>Root: "实现 X"
    Root->>Gen: turn
    Gen->>Sub: tool call (subagent)
    Sub->>Child: tx.createConversation( ownership: {kind:"task", taskId} )
    Sub->>Child: submit(input)
    Child->>Gen: model run (own cwd / extensions)
    Child-->>Reporter: answer event
    Reporter->>Root: follow-up input (把答案带回)
    Sub-->>Gen: toolResult text
    Gen->>Root: turn_end
    Note over Reporter: background anchor task<br/>survives Root abort
```

---

## 18. 总结

| 阶段 | 关键组件 | 关键产物 |
| --- | --- | --- |
| 启动 | CLI / Tui / 扩展加载 / 信任 | Session + Active branch |
| 输入 | Editor + prompt 模板 + 附件 | User entry |
| 请求 | prepareRequest / transformContext / convertToLlm | LLM Context |
| 调用 | pi-ai / Provider / Auth | Assistant stream |
| 持久化 | Session.append / JSONL | turn 完结 |
| 队列 | Steering / Follow-up | 下一 turn 输入 |
| 终止 | finishTurn / queue empty | agent_end |

**整套机制的同一性**：无论 interactive、RPC、还是 SDK（in-process），都跑同一套 agent loop —— 区别只在「谁在驱动输入流」与「事件流被谁消费」。

下一节 [模块架构图](module-architecture.md) 给出实现这些流程的代码层级与包依赖。