# Pi Agent: Entry Points and Core Flow

This document is a reference for reading the pi codebase. It maps the runtime from process start to the LLM call and back, naming the files you actually need to read.

## 1. Repository layout

Pi is a pnpm/npm workspaces monorepo. The packages you need are:

```
packages/
  ai/           LLM client library: provider registry, model catalog,
               message types, stream helpers (@earendil-works/pi-ai).
  agent/        Provider-agnostic agent loop (@earendil-works/pi-agent-core).
  coding-agent/ CLI binary `pi`, AgentSession, modes, tools, extensions,
               session manager, settings, resource loader.
  tui/          Terminal UI primitives (@earendil-works/pi-tui).
  mcp/          Model Context Protocol integration.
  client/       SDK client (SDK consumable).
  protocol/     Wire protocol for RPC.
  server/       Headless server used by RPC and SDK consumers.
  telemetry/    Telemetry.
  codemode/     Sandbox / codemode tools.
  chord/        Utility package.
  durable/      Durable execution utilities.
  env/          Environment / secrets utilities.
```

Only `ai`, `agent`, and `coding-agent` matter for understanding the core flow.

## 2. Three layers, three entry points

The runtime is built in three layers. Each layer has its own entry point and is reused by the layer above.

```
Layer 1:  pi-ai             streamSimple()  -> AssistantMessageEventStream
Layer 2:  pi-agent-core     Agent + runAgentLoop()  (stateful wrapper)
Layer 3:  pi-coding-agent   AgentSession + InteractiveMode / print-mode / rpc-mode
```

### 2.1 Layer 1 — `packages/ai` (LLM client)

Pure provider code. No agent loop, no tools, no state. Each provider is registered as an `Api` (`anthropic-messages`, `openai-responses`, `openai-completions`, `google-generative-ai`, `google-vertex`, `bedrock-converse-stream`, `mistral-conversations`, `azure-openai-responses`, `openai-codex-responses`, `pi-messages`).

The exposed function is `streamSimple(model, context, options)` in `packages/ai/src/compat.ts`. It returns an `AsyncIterable<AssistantMessageEvent>` that ends with `done` or `error` and yields a final `AssistantMessage` via `result()`.

There is also a `models` API and a model registry (`packages/ai/src/models-store.ts`, `models.ts`) used by the coding-agent runtime to look up models by `(provider, id)` and refresh from a remote catalog.

### 2.2 Layer 2 — `packages/agent` (`@earendil-works/pi-agent-core`)

Provider-agnostic agent loop. The two files that matter:

- `packages/agent/src/agent.ts` — `Agent` class. Owns transcript state, model, tools, steering/follow-up queues, abort signal, and event listeners.
- `packages/agent/src/agent-loop.ts` — `runAgentLoop()` / `runAgentLoopContinue()`. The actual loop: stream a response, run tool calls, repeat.

`Agent` is what `AgentSession` (Layer 3) wraps. It exposes:

- `state.{ systemPrompt, model, thinkingLevel, tools, messages }`
- `prompt(text | AgentMessage[])` — start a new user turn.
- `continue()` — resume after a non-assistant last message (e.g. retry after tool result).
- `steer()` / `followUp()` — queue messages to inject after the current turn.
- `subscribe(listener)` — receives the full lifecycle `AgentEvent` stream.
- `abort()` / `waitForIdle()`.

### 2.3 Layer 3 — `packages/coding-agent` (the `pi` CLI)

`pi` is the published CLI. Its bin entry is `dist/bundle/cli.js`, but the source entry is `packages/coding-agent/src/cli.ts`:

```
cli.ts  -> setupCli(); main(process.argv.slice(2))
main.ts -> main() in src/main.ts
```

Everything else builds on top of `AgentSession`.

## 3. Process startup (what `main()` does)

File: `packages/coding-agent/src/main.ts`. The shape of `main()`:

1. **`runAuthCommand(args)`** — handles `pi auth ...` subcommands; prints API keys, runs health checks, exits.
2. **`handlePackageCommand(args)` / `handleConfigCommand(args)`** — `pi package ...`, `pi config ...`. These are one-shot admin commands.
3. **`runMcpCommand(args)`** — `pi mcp ...` subcommand.
4. **`parseArgs(args)`** — full CLI parse (see `src/cli/args.ts`). Sets `mode` (`interactive` | `print` | `json` | `rpc`), `--model`, `--provider`, `--thinking`, `--session`, `--resume`, `--continue`, `--fork`, `--export`, etc.
5. **`runMigrations(cwd)`** — runs schema/data migrations.
7. **`createSessionManager(parsed, cwd, sessionDir, settingsManager)`** — resolves `--session` / `--resume` / `--continue` / `--fork` / `--session-id` to a `SessionManager` (open existing or create new, in-memory or on-disk JSONL).
8. **`createAgentSessionRuntime(createRuntime, { cwd, agentDir, sessionManager })`** — the core composition step. Returns an `AgentSessionRuntime`.

   - Internally calls the factory which calls `createAgentSessionServices()` (`packages/coding-agent/src/core/agent-session-services.ts`) to produce cwd-bound services:
     - `ModelRuntime` (auth.json + models.json + provider registry)
     - `SettingsManager` (settings.json, project + global layered)
     - `DefaultResourceLoader` (extensions, skills, prompt templates, themes, context files)
   - Then `createAgentSessionFromServices()` calls `createAgentSession()` (`packages/coding-agent/src/core/sdk.ts`) which actually builds the `Agent` and `AgentSession`.
9. Mode dispatch in `main()`:
   - `appMode === "rpc"` → `runRpcMode(runtime)` (`packages/coding-agent/src/modes/rpc/rpc-mode.ts`)
   - `appMode === "interactive"` → `new InteractiveMode(runtime, ...).run()` (`packages/coding-agent/src/modes/interactive/interactive-mode.ts`)
   - else (`print`/`json`) → `runPrintMode(runtime, ...)` (`packages/coding-agent/src/modes/print-mode.ts`)

The `Service` layer (`AgentSessionRuntime`) is cwd-bound. When the cwd changes (e.g. `--session` in a different project, `/new`, `/resume`, `/fork`, `/import`), the runtime tears down the current `AgentSession`, runs `session_shutdown` extensions, and rebuilds via the same factory. See `packages/coding-agent/src/core/agent-session-runtime.ts` for `switchSession`, `newSession`, `fork`, `importFromJsonl`.

## 4. The `AgentSession` class

File: `packages/coding-agent/src/core/agent-session.ts`. It is the shared abstraction used by all three modes (interactive, print, rpc).

What it owns:
- A reference to the underlying `Agent` (`@earendil-works/pi-agent-core`).
- A `SessionManager` (the JSONL session file, branching, compaction, custom entries — see `core/session-manager.ts`).
- A `SettingsManager` (project + global layered settings).
- A `ResourceLoader` (extensions, skills, prompt templates, themes, context files).
- A `ModelRuntime` (auth + provider registry + model catalog).
- An `ExtensionRunner` (`core/extensions/runner.ts`).
- Tool loadout state: built-in tools, custom tools, MCP tools, tool allow/exclude lists, active vs hidden declarations.
- Auto-retry, auto-compaction, branch summary state.
- `prompt()` (user input), `steer()`, `followUp()`, `abort()`, model cycling, session navigation.

Construction: see `createAgentSession()` in `core/sdk.ts`. The interesting bits:

- `existingSession = sessionManager.buildSessionContext()` — replays the JSONL branch into `AgentMessage[]` for the `Agent`'s initial state.
- `thinkingLevel` is restored from a `thinking_level_change` entry, falling back to settings and the model's capabilities (`clampThinkingLevel`).
- Tool selection is `options.tools` (CLI allowlist) → `defaultTools` setting → built-in defaults (`read, bash, edit, write`). Extension and custom tools are always enabled unless `noTools === "all"`.
- `convertToLlm: convertToLlmWithBlockImages` — drops image parts if `blockImages` is on (defense-in-depth against prompt-injection).
- `streamFn: (model, context, options) => modelRuntime.streamSimple(model, context, requestOptions)` — every LLM call flows through here.
- A handful of provider-pipeline hooks are wired: `transformHeaders` (adds provider attribution), `onResponse`, `onProviderStreamEvent`, `transformContext` (extension `context` rewriter).
- A `CacheWarmer` (`core/cache-warmer.ts`) keeps the prompt-cache entry for the current session request warm in the background; it cancels itself once `prepareRequest` decides a different model or a deeper request makes the cache stale.

### 4.1 The `_installAgent*` hooks

`AgentSession` overrides parts of the underlying `Agent` via four `prepareRequest` / `prepareNextTurn` / `finishTurn` overrides installed in the constructor:

- `_installAgentRequestProjection` — wraps `prepareRequest` to project the latest compaction into the context, resolve virtual models, and re-route to a physical model under a virtual selection.
- `_installAgentNextTurnRefresh` — wraps `prepareNextTurn` to auto-compact when the projected context exceeds the model's compaction threshold, then rebuild the system prompt for the next assistant response.
- `_installAgentBoundaryHooks` — wraps `finishTurn` to dispatch the `turn_end` boundary event (extension handlers can append entries via `SessionBoundaryDraft`).
- `_installHiddenDeclarationsProjection` and `_installAgentForcedPromptProjection` — keep declared tools and forced prompt messages in sync with the executable loadout.

These are how the session persists and replays state — the agent loop does not know about JSONL or compaction; the session re-projects around it.

### 4.2 The `prompt()` flow

File: `packages/coding-agent/src/core/agent-session.ts`, `prompt(text, options)` (around line 1954). High-level:

1. `try `_tryExecuteExtensionCommand(text)`` — if the input starts with `/`, try slash commands (extension commands take priority over LLM prompting).
3. `_runInputHandlers(...)` — extension `input` handlers may rewrite / block input.
4. `_expandSkillCommand(text)` and `expandPromptTemplate(...)` — `/skill:name` and `/template` substitution.
5. If `isStreaming`, queue as `steer` or `followUp` based on `options.streamingBehavior`.
6. `_flushPendingBashMessages()` and `_flushPendingCustomMessages()` — emit any context-only messages accumulated during the previous turn.
7. Check model + auth, optionally run pre-injection compaction (`_checkCompaction`).
8. `_extensionRunner.emitBeforeAgentStart(text, images, systemPromptOptions)` — extensions can edit the system prompt options, replace the model, etc.
9. Build the `AgentMessage[]` payload, then `await this.agent.prompt(messages)` — hands off to Layer 2.

## 5. The agent loop (Layer 2)

File: `packages/agent/src/agent-loop.ts`. `runAgentLoop(prompts, context, config, emit, signal, streamFn)` is the entry point.

The structure is two nested loops (see `runLoop`):

```
outer: while true
    inner: while hasMoreToolCalls || pendingMessages.length > 0
        1. prepareNextTurn (if not first iteration)   -> may compact, may refresh system prompt
        2. declareToolChanges  (system message with toolsAdded/toolsRemoved)
        3. prepareRequest   (projection, virtual model routing)
        4. streamAssistantResponse   (LLM call, yields events)
        5. executeToolCalls   (sequential or parallel; honors per-tool `executionMode`)
        6. finishTurn   (boundary event; "continue" / "end")
        7. drain steering queue
    drain follow-up queue
    exit if empty
```

Key details:

- **Events** (`AgentEvent`): `agent_start`, `turn_start`, `message_start`, `message_update`, `message_end`, `tool_execution_start`, `tool_execution_update`, `tool_execution_end`, `turn_end`, `agent_end`. Emitted in order; listeners are awaited in subscription order.
- **Tool execution**: `prepareToolCall` validates args + calls `beforeToolCall`, then `executePreparedToolCall` runs the tool with `onUpdate` partial-result emission, then `finalizeExecutedToolCall` calls `afterToolCall` to allow result rewriting. Errors are caught at every step and surfaced as `isError: true` tool results — tool execution never throws to the loop.
- **Truncation handling**: `failToolCallsFromTruncatedMessage` — if the assistant message was cut off by an output token limit (`stopReason === "length"`), every tool call in it is reported as an error so the model can re-issue with full arguments.
- **Steering / follow-up**: `config.getSteeringMessages` is polled at the top of each turn; `getFollowUpMessages` is polled when the inner loop would otherwise exit. Both default to `one-at-a-time` (drain first item only); `steeringMode` / `settingsMode` from settings can switch to `all`.

`Agent.runWithLifecycle` (Layer 2's wrapper) turns the loop into a single observable run: `isStreaming`, abort signal, error capture (`agent_end` with `stopReason: "error"`), and `agent_settled` semantic.

## 6. The three modes

All three modes wrap the same `AgentSession` and add only an I/O layer.

### 6.1 Interactive — `packages/coding-agent/src/modes/interactive/interactive-mode.ts`

`new InteractiveMode(runtime, options).run()`.

- Builds a TUI (`TUI` from `@earendil-works/pi-tui`) with chat viewport, footer, editor, status indicators, and a tree of components.
- Subscribes to `session.subscribe(...)` and renders events: `message_start` / `message_update` / `message_end` → streaming assistant component, `tool_execution_*` → tool components, `compaction_*` → status, etc.
- `keybindings.ts` + `default-keybindings.ts` define hotkeys; `core/extensions/runner.ts` lets extensions register additional keybindings, footer widgets, and UI commands.
- The editor accepts multi-line input with `@file` content references (resolved in `cli/file-processor.ts`), bash mode (`!cmd`), and slash commands. Submitted text goes to `session.prompt(text, options)`.
- Bash mode runs through `core/bash-executor.ts` (`executeBashWithOperations`) and emits `bash_execution_message` entries into the session, separately from LLM tool calls.

### 6.2 Print — `packages/coding-agent/src/modes/print-mode.ts`

`runPrintMode(runtime, { mode: "text" | "json", messages, initialMessage, initialImages })`.

- Subscribes to the session to stream JSON events on stdout in JSON mode; in text mode it just awaits `prompt(initialMessage)` and prints the final assistant text.
- Used by `pi -p "..."` (text), `pi --mode json "..."` (JSON events), and CI/automation.
- Handles SIGTERM/SIGHUP by killing detached children and disposing the runtime.

### 6.3 RPC — `packages/coding-agent/src/modes/rpc/rpc-mode.ts`

JSON-RPC over stdio. See `packages/coding-agent/src/modes/rpc/rpc-types.ts` for the wire protocol. The RPC mode lets an external process drive the same `AgentSession`; events come back as JSON-RPC notifications, and commands (`prompt`, `steer`, `followUp`, `abort`, `set_model`, `cycle_model`, `set_thinking_level`, `compact`, `get_state`, `switch_session`, `new_session`, `fork`, `navigate_tree`, `reload`, etc.) come in as JSON-RPC requests.

## 7. Tools, extensions, and resource loading

These three subsystems feed into `AgentSession` and the agent loop.

### 7.1 Tools — `packages/coding-agent/src/core/tools/`

Each tool is an `AgentTool` (`@earendil-works/pi-agent-core`):

```ts
{
  name: string;
  description: string;
  parameters: JSONSchema;
  execute: (args, onUpdate, signal) => Promise<ToolResult>;
  // optional: label, executionMode: "parallel" | "sequential"
}
```

Built-in tools: `read`, `write`, `edit`, `bash` (also `powershell` on Windows), `grep`, `find`, `ls`. Created in `core/tools/index.ts` and exposed via `createCodingTools`, `createReadOnlyTools`, etc. (also re-exported from `core/sdk.ts` for SDK consumers).

`withFileMutationQueue` (from `core/tools/file-mutation-queue.ts`) serializes file mutations to prevent race conditions between parallel tool calls touching the same files.

Tools can call other tools via `ctx.executeTool(...)` (nested tool calls). Nested calls go through the same `beforeToolCall` / `afterToolCall` pipeline; see `core/nested-tool-calls.ts` and `_executeNestedToolCall` in `AgentSession`.

The system prompt lists each active tool with a description and a snippet — see `core/system-prompt.ts` (`buildSystemPrompt`, `diffSystemPromptSections`). Declared-but-hidden tools (e.g. removed via `before_agent_start`) are filtered from the declaration but stay executable; see `_installHiddenDeclarationsProjection`.

### 7.2 Extensions — `packages/coding-agent/src/core/extensions/`

`ExtensionRunner` (`runner.ts`) is the event bus for extensions. It owns a `loadExtensionsResult` and dispatches events to extension handlers in registration order. Handlers can:
- Transform context (`context` event).
- Intercept requests and tool calls (`tool_call`, `tool_result`).
- Rewrite messages (`message_end`).
- Add slash commands, keybindings, footer widgets, custom tools.
- Register providers (`pendingProviderRegistrations`, `pendingNativeProviderRegistrations`, `pendingVirtualModelRegistrations`).
- React to session lifecycle (`session_start`, `session_shutdown`, `session_before_switch`, `session_before_fork`, `session_before_tree`, `session_before_compact`, `session_compact`, `session_compact_failed`).
- Inject custom messages and entries at boundaries via `SessionBoundaryDraft`.
- Inject UI commands (`input`, `ui_command`).

Extension loading happens in `DefaultResourceLoader.reload()` (`core/resource-loader.ts`). Extensions are TypeScript files loaded via `jiti`, plus built-in CJS/ESM extensions registered through `core/extensions/index.ts` (`builtInExtensions`).

### 7.3 Resource loader — `packages/coding-agent/src/core/resource-loader.ts`

`DefaultResourceLoader` discovers extensions, skills, prompt templates, themes, and context files. Skills live in `<cwd>/.pi/skills/`, prompt templates in `.pi/prompts/`, themes in `.pi/themes/`, context files in `.pi/context/`, plus global equivalents under `agentDir`. CLI flags can add extra paths (`--extension`, `--skill`, `--prompt-template`, `--theme`).

## 8. Session storage — `packages/coding-agent/src/core/session-manager.ts`

A `SessionManager` owns a JSONL session file with a branching tree of entries. Entry types (`SessionEntry`):
- `message` — user / assistant / toolResult / system / bashExecution / custom
- `model_change` — virtual or physical model change
- `compaction` — summary of a compacted prefix
- `branch_summary` — summary attached at a branch cut
- `context_edit` — replacement for a target entry
- `custom` / `custom_message` — extension-defined entries and messages
- `thinking_level_change`, `session_info`

`buildSessionContext()` replays the current leaf into `Message[]`. `buildSessionProjection()` is the canonical "what the LLM should see" — applies compactions, branch summaries, and context edits in order.

`/fork` (entry-based fork), `/new`, `/resume`, `/import` all go through `AgentSessionRuntime` (`core/agent-session-runtime.ts`) and use `SessionManager.forkFrom`, `create`, `open`, `createBranchedSession`.

## 9. Models and auth — `core/model-runtime.ts`, `core/auth-storage.ts`, `core/auth-guidance.ts`

`ModelRuntime` is the coding-agent's view of "what models I have, what auth I have":
- `models.json` (per-agent-dir) for the local catalog (defaults + custom registrations).
- `auth.json` (per-agent-dir) for credentials (API keys, OAuth tokens, headers, base URLs, custom env).
- `ModelRegistry` (`core/model-registry.ts`) merges built-in catalog + extension-registered providers + custom virtual models.
- `streamSimple(model, context, options)` resolves auth + headers and calls into Layer 1 (`pi-ai`).

OAuth is supported natively for some providers (`packages/ai/src/bun-oauth.ts`, `packages/ai/src/oauth.ts`). CLI command `pi auth login <provider>` performs the OAuth flow; runtime stores the resulting credential in `auth.json` and the `ModelRuntime` refreshes it before expiry.

## 10. Where to read next

If you only read four files, read these in order:

1. `packages/coding-agent/src/main.ts` — process startup, mode dispatch.
2. `packages/coding-agent/src/core/sdk.ts` (`createAgentSession`) — assembles `Agent` + `AgentSession`.
3. `packages/coding-agent/src/core/agent-session.ts` (`prompt`, `_handleAgentEvent`) — session-level orchestration.
4. `packages/agent/src/agent-loop.ts` (`runLoop`) — the inner LLM/tool loop.

After that:
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts` for the TUI integration.
- `packages/coding-agent/src/core/extensions/runner.ts` for the extension event bus.
- `packages/coding-agent/src/core/session-manager.ts` for the JSONL persistence and branching model.