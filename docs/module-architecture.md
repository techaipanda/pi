# Pi 模块架构图

> 本文档描绘 Pi monorepo 的**包依赖关系**、**分层架构**与**运行时组件拓扑**，并说明每个模块的职责边界。

仓库 `packages/` 目录下的 15 个包在功能上分属 5 层：基础设施、模型与协议、Agent 运行时、嵌入式运行时、终端 UI 与 CLI。每一层有明确的依赖方向 — **永远只依赖更低层，不反向依赖**。

**交互式架构图**：[module-architecture.html](module-architecture.html) — 12 个 npm 包按 5 层组织的依赖连线视图。

---

## 1. 包清单速览

| 包名 | 路径 | 角色 |
| --- | --- | --- |
| `@earendil-works/chord` | `packages/chord` | 应用组合运行时（facets、services、replicated state） |
| `@earendil-works/pi-protocol` | `packages/protocol` | CBOR 二进制协议与帧 |
| `@earendil-works/pi-telemetry` | `packages/telemetry` | 供应商中立遥测合约 |
| `@earendil-works/pi-ai` | `packages/ai` | 多供应商 LLM 统一 API |
| `@earendil-works/pi-codemode` | `packages/codemode` | QuickJS 沙箱内运行模型代码 |
| `@earendil-works/pi-mcp` | `packages/mcp` | MCP 客户端（stdio / Streamable HTTP） |
| `@earendil-works/pi-agent-core` | `packages/agent` | 状态化 Agent + 工具执行 + 事件流 |
| `@earendil-works/pi-durable` | `packages/durable` | Durable conversation / task / document runtime |
| `@earendil-works/pi-tui` | `packages/tui` | 终端 UI 框架（差分渲染） |
| `@earendil-works/pi-coding-agent` | `packages/coding-agent` | Pi CLI（终端 + RPC + 工具集 + 扩展） |
| `@earendil-works/pi-server` | `packages/server` | 本地 CBOR 服务（路由 Sessions） |
| `@earendil-works/pi-client` | `packages/client` | 远端 Pi 会话的传输中性客户端 |
| `@earendil-works/pi-evals` | `packages/evals` | 评测工具（私有） |

---

## 2. 包依赖图

### 2.1 总览（分层）

```mermaid
flowchart TB
    classDef infra fill:#e8f4fd,stroke:#3b82f6,color:#1e3a8a
    classDef model fill:#fef3c7,stroke:#f59e0b,color:#78350f
    classDef agent fill:#dcfce7,stroke:#22c55e,color:#14532d
    classDef embed fill:#fae8ff,stroke:#a855f7,color:#581c87
    classDef ui fill:#fee2e2,stroke:#ef4444,color:#7f1d1d

    %% 基础设施层
    chord["@earendil-works/chord<br/><i>facets / replicated state / RPC</i>"]:::infra
    proto["@earendil-works/pi-protocol<br/><i>CBOR + framing</i>"]:::infra
    telem["@earendil-works/pi-telemetry<br/><i>TelemetryContext contract</i>"]:::infra

    %% 模型层
    ai["@earendil-works/pi-ai<br/><i>Unified LLM API</i>"]:::model
    codemode["@earendil-works/pi-codemode<br/><i>QuickJS sandbox</i>"]:::model
    mcp["@earendil-works/pi-mcp<br/><i>MCP client</i>"]:::model

    %% Agent 运行时
    agent["@earendil-works/pi-agent-core<br/><i>Stateful agent + tool execution</i>"]:::agent

    %% 嵌入式运行时
    durable["@earendil-works/pi-durable<br/><i>Durable Harness</i>"]:::embed
    server["@earendil-works/pi-server<br/><i>Local CBOR router</i>"]:::embed
    client["@earendil-works/pi-client<br/><i>Remote client</i>"]:::embed

    %% UI 与 CLI
    tui["@earendil-works/pi-tui<br/><i>Terminal UI kit</i>"]:::ui
    codingAgent["@earendil-works/pi-coding-agent<br/><i>Pi CLI + extensions</i>"]:::ui

    %% 依赖关系
    proto --> chord
    ai --> telem
    agent --> ai
    mcp -.->|独立无依赖| mcp
    codemode -.->|独立无依赖| codemode
    durable --> ai
    durable --> chord
    server --> proto
    server --> chord
    client --> proto
    client --> chord
    codingAgent --> agent
    codingAgent --> ai
    codingAgent --> mcp
    codingAgent --> codemode
    codingAgent --> chord
    codingAgent --> tui
```

> 关键约束：**chord / protocol / telemetry / codemode / mcp / tui 不依赖任何其他 pi 包**。它们是基础设施层，可被任何上层单独消费。

### 2.2 Workspace 内的精确依赖

```mermaid
flowchart LR
    classDef infra fill:#e8f4fd,stroke:#3b82f6
    classDef model fill:#fef3c7,stroke:#f59e0b
    classDef agent fill:#dcfce7,stroke:#22c55e
    classDef embed fill:#fae8ff,stroke:#a855f7
    classDef ui fill:#fee2e2,stroke:#ef4444

    chord[chord]:::infra
    proto[protocol]:::infra
    telem[telemetry]:::infra

    ai[ai]:::model
    codemode[codemode]:::model
    mcp[mcp]:::model

    agent[agent]:::agent

    durable[durable]:::embed
    server[server]:::embed
    client[client]:::embed

    tui[tui]:::ui
    codingAgent[coding-agent]:::ui

    ai --> telem
    agent --> ai
    durable --> chord
    durable --> ai
    server --> chord
    server --> proto
    client --> chord
    client --> proto
    codingAgent --> chord
    codingAgent --> ai
    codingAgent --> agent
    codingAgent --> mcp
    codingAgent --> codemode
    codingAgent --> tui
    proto --> chord
```

> 没有任何包依赖 `coding-agent`（它是产品层），没有任何包跨层依赖「同层」包。

---

## 3. 分层架构（从底层到顶层）

```mermaid
flowchart TB
    subgraph L0["L0 · 通用基础设施 (无 pi 包依赖)"]
        Chord["@earendil-works/chord"]
        Proto["@earendil-works/pi-protocol"]
        Telem["@earendil-works/pi-telemetry"]
    end

    subgraph L1["L1 · 模型 & 沙箱原语"]
        Ai["@earendil-works/pi-ai"]
        CM["@earendil-works/pi-codemode"]
        Mcp["@earendil-works/pi-mcp"]
    end

    subgraph L2["L2 · Agent 运行时"]
        Agent["@earendil-works/pi-agent-core"]
    end

    subgraph L3["L3 · 嵌入式运行时"]
        Durable["@earendil-works/pi-durable"]
        Server["@earendil-works/pi-server"]
        Client["@earendil-works/pi-client"]
    end

    subgraph L4["L4 · 产品层"]
        Tui["@earendil-works/pi-tui"]
        Ca["@earendil-works/pi-coding-agent"]
    end

    L0 --> L1
    L1 --> L2
    L1 --> L3
    L2 --> L4
    L3 --> L4
    L0 --> L3
    Tui -.->|独立 UI 库| L3
```

| 层 | 职责 | 反向依赖约束 |
| --- | --- | --- |
| **L0 基础设施** | 通用原语 | 不依赖任何 pi 包 |
| **L1 模型 & 沙箱** | 把 LLM 与外部世界封装成统一接口 | 不依赖 L2+ |
| **L2 Agent 运行时** | 单进程 Agent 调度 | 不依赖 L3+ |
| **L3 嵌入式运行时** | 跨进程、跨主机的运行时 | 允许依赖 L0、L1 |
| **L4 产品层** | TUI、CLI、扩展 | 允许依赖所有层 |

---

## 4. 包内模块与运行时组件

### 4.1 `@earendil-works/chord` — 应用组合运行时

```mermaid
flowchart LR
    subgraph ChordRuntime["Chord 运行时"]
        Facets["Facets<br/><i>声明服务、依赖</i>"]
        Services["Services<br/><i>singleton / keyed tokens</i>"]
        ReplState["Replicated State<br/><i>overlay commits</i>"]
        Delta["Delta Tracker<br/><i>JSON op coalescing</i>"]
        Remote["Remote Bindings<br/><i>$chord.service vocabulary</i>"]
        Context["Context<br/><i>Go-like cancellation</i>"]
        Bundler["Bundler<br/><i>esbuild facet 编译</i>"]
        Loader["Node Loader<br/><i>SHA-256 校验 + vm</i>"]
    end
    ChordRuntime --> AppPlugins["应用插件 / facet"]
```

Chord 不是 Pi 包；它是 **Pi 与 Pi 协作应用之间的共享运行时**，可被任何应用使用。

### 4.2 `@earendil-works/pi-ai` — 统一 LLM API

```mermaid
flowchart TB
    subgraph AiCore["pi-ai Core"]
        Models["Models Collection<br/><i>createModels() / builtinModels()</i>"]
        Auth["CredentialStore<br/><i>api_key, oauth</i>"]
        Store["ModelsStore<br/><i>动态目录持久化</i>"]
        Context["Context<br/><i>system + history + tools</i>"]
        Frame["AssistantMessageFrame<br/><i>compact event encoding</i>"]
    end

    subgraph Providers["Provider Factory 层"]
        P1["anthropic-provider"]
        P2["openai-provider"]
        P3["google / vertex / bedrock"]
        P4["openrouter / groq / mistral"]
        P5["... 30+ providers"]
    end

    subgraph ApiLayer["Api 实现层 (lazy)"]
        A1["anthropic-messages"]
        A2["openai-completions"]
        A3["openai-responses"]
        A4["openai-codex-responses"]
        A5["azure-openai-responses"]
        A6["google-generative-ai"]
        A7["google-vertex"]
        A8["mistral-conversations"]
        A9["bedrock-converse-stream"]
    end

    Models --> Auth
    Models --> Store
    Models --> Providers
    Providers --> ApiLayer
    Models --> Context
    Models --> Frame
```

设计原则：**wire protocol 复用**。例如 `openai-completions` 协议被 xAI、Groq、Cerebras、OpenRouter、Together 等多个供应商复用。

### 4.3 `@earendil-works/pi-agent-core` — Agent 运行时

```mermaid
flowchart TB
    subgraph AgentCore["Agent 运行时"]
        Agent["Agent class<br/><i>事件订阅 / state / queue</i>"]
        Loop["agentLoop / agentLoopContinue<br/><i>async iterator API</i>"]
        ToolExec["Tool Execution<br/><i>parallel / sequential</i>"]
        Hooks["Hooks<br/><i>beforeToolCall, afterToolCall</i><br/><i>prepareRequest, finishTurn</i>"]
        State["State<br/><i>model / tools / messages</i>"]
        Custom["Custom Message Types<br/><i>declaration merging</i>"]
    end

    LlmStream["models.streamSimple()"] --> Agent
    Agent --> Loop
    Agent --> ToolExec
    Agent --> Hooks
    Agent --> State
    Agent --> Custom
```

`Agent` 类基于 `transformContext` → `convertToLlm` → LLM 的管道：

```text
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
                    (optional)              (required for custom roles)
```

### 4.4 `@earendil-works/pi-durable` — Durable Harness

```mermaid
flowchart TB
    subgraph Durable["Durable Harness"]
        Harness["Harness.open()<br/><i>storage + models + registry</i>"]
        Conv["Conversation<br/><i>id + entries + docs</i>"]
        Tasks["Tasks<br/><i>generation / tool / compaction</i>"]
        Submissions["Submissions<br/><i>input / write / steer / follow-up</i>"]
        Watch["watch() / viewState() / watchEvents()"]
        Storage["Storage backends<br/><i>Memory / SQLite / JSONL</i>"]
    end

    Harness --> Conv
    Harness --> Tasks
    Harness --> Submissions
    Harness --> Watch
    Harness --> Storage
```

核心循环（one answered input）：

```text
submit(input) → pi.user
  pi.generation → pi.system (only if prompt/tools changed), pi.assistant (tool calls)
    pi.tool × n → pi.tool-result × n   (owned by pi.generation, which waits)
  pi.generation → pi.assistant (answer) → submission done
```

### 4.5 `@earendil-works/pi-coding-agent` — Pi CLI

```mermaid
flowchart LR
    subgraph CodingAgent["coding-agent"]
        CLI["CLI Entrypoint<br/><i>bin: pi</i>"]
        Modes["Modes<br/><i>interactive / print / json / rpc</i>"]
        TuiLoop["TUI Loop"]
        RpcLoop["RPC Loop"]
        Session["Session Manager<br/><i>JSONL 持久化</i>"]
        Tools["Built-in Tools<br/><i>read, write, edit, bash</i>"]
        Cfg["Configuration<br/><i>settings.json + .pi/*</i>"]
        Ext["Extension Loader<br/><i>extensions / skills / themes</i>"]
        Mcp["MCP Integration"]
        Cmdb["Codemode Tool"]
        Telemetry["Telemetry Hooks"]
    end

    CLI --> Modes
    Modes --> TuiLoop
    Modes --> RpcLoop
    TuiLoop --> Session
    RpcLoop --> Session
    Session --> Tools
    Session --> Ext
    Ext --> Mcp
    Ext --> Cmdb
    Session --> Telemetry
    Session --> Cfg
```

CLI 在 `dist/bundle/` 内使用 esbuild 打包，单文件可执行。

### 4.6 `@earendil-works/pi-server` + `@earendil-works/pi-client` + `@earendil-works/pi-protocol`

```mermaid
flowchart LR
    subgraph ServerSide["Server 进程"]
        Coord["Coordinator<br/><i>RoutedServerServiceHost</i>"]
        Attach["attachClient()<br/><i>connection-scoped attachment</i>"]
        SessionDir["SessionDirectory<br/><i>私有 catalog → 公开 Replicated State</i>"]
        Routing["Route installation<br/><i>{serverId, sessionId, attachmentId}</i>"]
        SessHandle["RoutedSessionHandle<br/><i>presentation-scoped</i>"]
    end

    subgraph ClientSide["Client 进程"]
        C["pi-client"]
        Decoder["ServerMessageDecoder<br/><i>ClientMessageEncoder</i>"]
    end

    subgraph Transport["Transport"]
        Unix["Unix Socket"]
        CBOR["CBOR + framing<br/><i>(pi-protocol v8)</i>"]
    end

    C --> Decoder
    Decoder <--> CBOR
    CBOR <--> Coord
    Coord --> Attach
    Coord --> SessionDir
    Coord --> Routing
    Coord --> SessHandle
    SessHandle --> Worker["Worker process<br/>(owns Harness + Storage)"]
```

协议使用 4 字节大端 payload length + 然后是一个 definite-length CBOR 项。

### 4.7 `@earendil-works/pi-mcp` — MCP 客户端

```mermaid
flowchart LR
    subgraph Mcp["pi-mcp"]
        Core["McpClient<br/><i>JSON-RPC correlation</i>"]
        Stdio["StdioTransport"]
        Http["StreamableHttpTransport"]
        Oauth["OAuth subset<br/><i>PKCE / refresh</i>"]
        Test["InMemoryTransport (testing)"]
    end
    Core --> Stdio
    Core --> Http
    Core --> Oauth
    Core --> Test
```

支持的协议面：MCP `2025-11-25`，接受 `2025-06-18` / `2025-03-26` / `2024-11-05`。

### 4.8 `@earendil-works/pi-codemode` — 沙箱

```mermaid
flowchart LR
    subgraph Cm["pi-codemode"]
        Sand["CodemodeSandbox"]
        Wasm["QuickJS wasm"]
        Worker["Worker Thread"]
        HostBridge["Host Bridge<br/><i>唯一出口</i>"]
        Tools["tools / ALL_TOOLS"]
        Store["store / load"]
        Globals["globals"]
    end

    Sand --> Wasm
    Sand --> Worker
    Worker --> HostBridge
    HostBridge --> Tools
    HostBridge --> Store
    HostBridge --> Globals
```

VM 中**唯一可用能力**是调用已注入函数。无 `fetch`、无 `require`、无定时器 — 与宿主完全隔离。

### 4.9 `@earendil-works/pi-tui` — 终端 UI

```mermaid
flowchart LR
    subgraph Tui["pi-tui"]
        TuiInterface["TUI interface<br/><i>lifecycle, focus, overlay</i>"]
        Main["TuiMainScreen<br/><i>主屏 + 保留 scrollback</i>"]
        Alt["TuiAltScreen<br/><i>备用屏 + 应用自有滚动</i>"]
        Comp["Component Library<br/><i>Text/Editor/Markdown/...</i>"]
        Term["Terminal Interface<br/><i>ProcessTerminal/VirtualTerminal</i>"]
        Native["Native clipboard<br/><i>darwin/linux/win32 prebuilds</i>"]
    end

    TuiInterface --> Main
    TuiInterface --> Alt
    Main --> Comp
    Alt --> Comp
    Comp --> Term
    Term --> Native
```

### 4.10 `@earendil-works/pi-telemetry` — 遥测合约

```mermaid
flowchart LR
    subgraph Telemetry["pi-telemetry"]
        Ctx["TelemetryContext<br/><i>startSpan</i>"]
        Span["TelemetrySpan<br/><i>attributes/events/status</i>"]
        Noop["NOOP_TELEMETRY_CONTEXT"]
        InMem["InMemoryTelemetryContext<br/><i>参考实现</i>"]
        Schema["defineTelemetrySchema()<br/><i>类型化 schema</i>"]
        Conf["createTelemetryAdapterConformance()"]
    end

    Ctx --> Span
    Ctx --> Noop
    Ctx --> InMem
    Ctx --> Schema
    Ctx --> Conf
    Noop -.->|可选实现| Span
```

Telemetry 不依赖任何后端，Pi 包定义自己的 schema (`pi.ai.*`、`pi.harness.*`、`pi.session.*`)。

---

## 5. 端到端运行时组件拓扑

下面是从用户启动 `pi` 命令开始、到一次 agent loop 完成的所有运行时组件。

```mermaid
flowchart TB
    subgraph UserSpace["用户终端"]
        Term[("Stdin / TTY")]
    end

    subgraph Process["pi 进程"]
        CLI["CLI (bin: pi)"]
        Mode[Mode Selector<br/><i>interactive / print / json / rpc</i>"]
        Session["Session Manager"]
        ToolsLayer["Tool Registry"]
        Hooks["Hook Chain"]
        Loader["Extension Loader"]
        ThemeLoader["Theme Loader"]
    end

    subgraph CoreEngine["Core Engine"]
        Agent["Agent (pi-agent-core)"]
        Ai["Models Collection (pi-ai)"]
        Conv[("Session JSONL<br/>(on disk)")]
        Emits["Agent Event Stream"]
    end

    subgraph External["外部服务 / Provider"]
        P1["Anthropic / OpenAI / ..."]
        P2["Vertex / Bedrock"]
        P3["OpenRouter / Groq"]
        P5["Local server"]
    end

    subgraph ToolSide["Tool / Sandbox 层"]
        Coding[("Built-in CodingTools")]
        Mcp["MCP Servers"]
        Cmdb["Codemode (QuickJS)"]
    end

    subgraph UiLayer["UI 层"]
        Tui[("TUI 框架<br/>(pi-tui)")]
        DiffRender["Synchronized output / Diff render"]
        Comp[("Component tree")]
    end

    Term --> CLI
    CLI --> Mode
    Mode --> Session
    Session --> Loader
    Loader --> ThemeLoader
    Session --> ToolsLayer
    ToolsLayer --> Coding
    ToolsLayer --> Mcp
    ToolsLayer --> Cmdb
    Session --> Hooks
    Session --> Agent
    Agent --> Ai
    Ai --> P1
    Ai --> P2
    Ai --> P3
    Agent --> Emits
    Session --> Conv
    Emits --> Tui
    Tui --> DiffRender
    DiffRender --> Comp
    Comp --> Term
    Session -.->|RPC mode| P5
    P5 -.->|CBOR| Session
```

---

## 6. Agent Loop 的运行时组件流

```mermaid
sequenceDiagram
    autonumber
    participant U as User / CLI
    participant S as Session
    participant H as Hooks
    participant A as Agent
    participant M as Models (pi-ai)
    participant P as Provider
    participant T as Tools
    participant J as JSONL

    U->>S: prompt("...")
    S->>J: append user entry
    S->>H: prepareRequest()
    H->>A: agentLoop()
    A->>A: transformContext()
    A->>A: convertToLlm()
    A->>M: streamSimple(model, ctx)
    M->>P: POST /messages
    P-->>M: stream events
    M-->>A: start/text_delta/done
    A-->>S: agent events
    A->>T: beforeToolCall()
    T->>T: execute(args)
    T-->>A: result
    A->>T: afterToolCall()
    A->>J: append assistant + toolResult
    A->>A: finishTurn()
    A-->>S: turn_end
    A-->>S: agent_end
```

---

## 7. Durable Harness 的写入与重放流

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant H as Harness
    participant S as Storage
    participant T as Tasks
    participant M as Models

    App->>H: root.submit(input, ctx)
    H->>S: commit(pi.user, requestId)
    H->>T: createTask(pi.generation)
    T->>S: commit(checkpoint)
    T->>M: stream()
    M-->>T: events
    T->>S: commit(pi.assistant + tool calls)
    T->>T: createTask(pi.tool × n)
    T->>S: commit(pi.tool-result × n)
    T-->>H: generation done
    H->>S: commit(submission done)
    Note over H,S: Process 崩溃后<br/>
H->>S: resume() 读取 pending tasks
    H->>T: 重新跑 pending → checkpoint
```

---

## 8. RPC 模式的进程拓扑

```mermaid
flowchart LR
    subgraph Client["Client 进程"]
        UI["Web / CLI / Bot"]
        WM->>UI
        C["pi-client"]
    end

    subgraph Server["Server 进程"]
        Coord["Coordinator<br/>(pi-server)"]
        Sess["SessionDirectory"]
    end

    subgraph Worker["Worker 进程"]
        Harness["Harness"]
        Storage[("SQLite / JSONL")]
        Models["McpServer"]
    end

    Client <-->|CBOR frames| Server
    Server -->|attachClient + route install| Worker
    Worker <--> Storage
    Worker --> Models
```

---

## 9. 关键设计决策

| 决策 | 选择 | 理由 |
| --- | --- | --- |
| 包边界 | 每包一个 npm 包，独立 publish | 允许 SDK / 服务端 / 客户端各自消费；避免庞大单体 |
| chord 独立 | 不属于 `@earendil-works/pi-*`，命名空间隔离 | Chord 是应用组合通用工具，可被非 Pi 项目复用 |
| protocol 用二进制 CBOR | 不用 JSON | 性能、字节级确定性、可携带 |
| telemetry 没 API 过多 | 提供合约 + Noop + 参考实现 + conformance | 不依赖特定 exporter，给应用留选择空间 |
| codemode 强制隔离 | QuickJS + 唯一宿主桥 | 模型代码即便出错也只影响产出，不会逃逸 |
| mcp 不依赖官方 SDK | 重写 core + transport | 体积可控、与官方无关；保留兼容 |
| durable 的 chain 是实验性 | 显式标注 | API 仍在演进 |
| TUI 框架独立 | 选 marketplace 可独立消费 | 其他应用也能复用差分渲染 |
| 进程内 extensions | 不引外部进程 | 简单，避免 RPC 开销；代价是权限同进程 |
| Pin all deps | `.npmrc` + `save-exact=true` | 供应链：直接导入对应部署改动 |
| 无 bundle lockfile | 安装器用 install-lock | 锁定传递依赖；npm 包不锁定 |

---

## 10. 与本系列文档的关联

- **功能全景图** [feature-landscape.md](feature-landscape.md)：解释每个能力由哪个包提供。
- **运行流程图** [runtime-flow.md](runtime-flow.md)：解释这些组件按什么顺序协作完成一次对话。