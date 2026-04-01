# Claude Code — Learning Guide & Knowledge Extraction

> In-depth analysis based on decompiled Claude Code v2.1.88 source  
> This document systematically covers the software's requirements, features, and highlights for learning purposes

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Software Requirements & Design Goals](#2-software-requirements--design-goals)
3. [Architecture Overview](#3-architecture-overview)
4. [Core Feature Modules](#4-core-feature-modules)
   - [4.1 Agent Loop](#41-agent-loop)
   - [4.2 Tool System](#42-tool-system)
   - [4.3 Permission System](#43-permission-system)
   - [4.4 Context Compaction](#44-context-compaction)
   - [4.5 Multi-Agent Collaboration](#45-multi-agent-collaboration)
   - [4.6 MCP Protocol Integration](#46-mcp-protocol-integration)
   - [4.7 Session Persistence](#47-session-persistence)
   - [4.8 Slash Command System](#48-slash-command-system)
   - [4.9 Memory System (CLAUDE.md)](#49-memory-system-claudemd)
5. [Technical Highlights](#5-technical-highlights)
   - [5.1 The 12 Progressive Harness Mechanisms](#51-the-12-progressive-harness-mechanisms)
   - [5.2 Key Design Patterns](#52-key-design-patterns)
   - [5.3 State Management Architecture](#53-state-management-architecture)
   - [5.4 Terminal UI with React/Ink](#54-terminal-ui-with-reactink)
6. [Hidden Features & Deep Discoveries](#6-hidden-features--deep-discoveries)
7. [Future Roadmap](#7-future-roadmap)
8. [Code Quality Observations](#8-code-quality-observations)
9. [Key Learning Takeaways](#9-key-learning-takeaways)

---

## 1. Project Overview

**Claude Code** is Anthropic's AI-powered terminal coding agent, published as the npm package `@anthropic-ai/claude-code`.

| Property | Value |
|---------|-------|
| Version | 2.1.88 |
| Tech stack | TypeScript + React (Ink terminal UI) + Bun (compiler) |
| Runtime | Node.js ≥ 18 (published as a single 12MB bundle) |
| Source files | ~1,884 .ts/.tsx files |
| Lines of code | ~512,664 |
| Built-in tools | 40+ |
| Slash commands | ~80 |
| Dependencies | ~192 npm packages |
| Largest file | `query.ts` (~785KB, main agent loop) |

**Core value proposition**: Deep integration of Claude LLM capabilities with the local development environment, enabling AI to read/write files, execute commands, manage Git, search codebases, and complete complex programming tasks through multi-agent collaboration — just like a human developer.

---

## 2. Software Requirements & Design Goals

### 2.1 Functional Requirements

| Category | Requirements |
|---------|-------------|
| **File operations** | Read, edit, create files; support PDF, images, Jupyter Notebooks |
| **Command execution** | Safely execute Bash commands; PowerShell support on Windows |
| **Code search** | ripgrep-based content search; glob pattern file search |
| **Network access** | HTTP fetch; optional web search |
| **Version control** | Git operations; GitHub API; PR management |
| **Multi-session** | Session persistence, resume, fork; cross-session memory |
| **Multi-agent** | Sub-agent fork; parallel collaboration teams; remote agents |
| **Extensibility** | MCP protocol services; plugin system; Skill system |

### 2.2 Non-Functional Requirements

| Category | Requirements |
|---------|-------------|
| **Security** | Tool call permission auditing; sandbox execution; path boundary checks |
| **Cost control** | Precise token usage and USD cost tracking |
| **Context management** | Auto-detect and compress long conversations, avoid prompt-too-long errors |
| **Observability** | Telemetry (OpenTelemetry + Datadog); detailed logging |
| **Responsiveness** | Full-chain streaming output; parallel tool execution |
| **Portability** | macOS / Linux / Windows (WSL); Docker containers; CI/CD environments |
| **Configurability** | Global config + project-level config; enterprise managed settings |

### 2.3 Design Constraints

- **Zero runtime dependencies**: Compiled to a single bundle, only requires Node.js standard library
- **Offline-first**: Core features don't require network (except API calls)
- **Non-destructive**: All file modifications have undo mechanisms (FileHistory)

---

## 3. Architecture Overview

### 3.1 Layered Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                         ENTRY LAYER                           │
│  cli.tsx ──> main.tsx ──> REPL.tsx (interactive terminal)    │
│                      └──> QueryEngine.ts (SDK/headless)       │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                       QUERY ENGINE                            │
│  submitMessage(prompt) ──> AsyncGenerator<SDKMessage>         │
│    ├── fetchSystemPromptParts()  → dynamic system prompt      │
│    ├── processUserInput()        → handle /commands           │
│    ├── query()                   → main agent loop (785KB)    │
│    │     ├── StreamingToolExecutor → parallel execution        │
│    │     ├── autoCompact()         → context compression       │
│    │     └── runTools()            → tool orchestration        │
│    └── yield SDKMessage            → stream to consumer        │
└───────────────────────────────┬──────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   TOOL SYSTEM   │  │  SERVICE LAYER   │  │   STATE LAYER    │
│  Tool Interface │  │  api/claude.ts   │  │  AppStateStore   │
│  40+ tools      │  │  compact/        │  │  React Context   │
│  MCPTool        │  │  mcp/            │  │  FileHistory     │
│  SkillTool      │  │  analytics/      │  │  ToolPermission  │
└─────────────────┘  └──────────────────┘  └──────────────────┘
              │                 │
              ▼                 ▼
┌─────────────────┐  ┌──────────────────┐
│   TASK SYSTEM   │  │   BRIDGE LAYER   │
│  local_bash     │  │  Claude Desktop  │
│  local_agent    │  │  bridgeMain.ts   │
│  remote_agent   │  │  JWT + work sec. │
│  dream / ...    │  │  capacityWake    │
└─────────────────┘  └──────────────────┘
```

### 3.2 Data Flow: Single Query Lifecycle

```
USER INPUT
    │
    ▼
processUserInput()       ← parse /commands, build UserMessage
    │
    ▼
fetchSystemPromptParts() ← tools + CLAUDE.md memory + permissions
    │
    ▼
recordTranscript()        ← persist user message to disk (JSONL)
    │
    ▼
┌─→ normalizeMessagesForAPI()  ← strip UI fields, compact if needed
│   │
│   ▼
│   Claude API (streaming)     ← POST /v1/messages
│   │
│   ├─ text block ───────────→ yield to consumer (SDK / REPL)
│   │
│   └─ tool_use block?
│       │
│       ▼
│   StreamingToolExecutor      ← partition: concurrent-safe vs serial
│       │
│       ▼
│   canUseTool()               ← permission check
│       │
│       ├─ DENY ─────────────→ append tool_result(error), continue loop
│       │
│       └─ ALLOW
│           │
│           ▼
│       tool.call()             ← execute tool
│           │
│           ▼
│       append tool_result      ← push to messages[], recordTranscript()
│           │
└─────────┘                    ← loop back to API call
    │
    ▼ (stop_reason != "tool_use")
yield result message            ← final text, usage, cost, session_id
```

---

## 4. Core Feature Modules

### 4.1 Agent Loop

The **minimal agent loop** is the essence of every AI agent:

```
User → messages[] → Claude API → response
                                  |
                        stop_reason == "tool_use"?
                       /                          \
                     yes                           no
                      |                             |
                 execute tools                  return text
                 append tool_result
                 loop back ────────────→ messages[]
```

Claude Code wraps this minimal loop with 12 layers of production mechanisms (see section 5.1).

**Core file**: `src/query.ts` (785KB, largest file, contains full agent loop logic)

### 4.2 Tool System

#### Tool Interface Definition

Every tool is an object implementing `Tool<Input, Output, Progress>`:

```typescript
export type Tool<Input, Output, Progress> = {
  // Core (required)
  name: string
  inputSchema: ZodSchema           // Zod-defined input structure
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<Output>>
  checkPermissions(input, context): Promise<PermissionResult>
  description(input, options): Promise<string>
  prompt(...): Promise<string>     // Description for Claude
  
  // Capability declarations
  isConcurrencySafe(input): boolean   // Runs in parallel?
  isReadOnly(input): boolean          // No side effects?
  isDestructive?(input): boolean      // Irreversible?
  isEnabled(): boolean                // Feature gate check
  
  // UI rendering (React/Ink)
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage(content, progress, options): React.ReactNode
  renderToolUseProgressMessage(progress, options): React.ReactNode
}
```

**Factory function**: `buildTool(definition)` provides safe defaults for optional methods.

#### Complete Tool Inventory

```
FILE OPERATIONS          SEARCH & DISCOVERY        EXECUTION
════════════════         ══════════════════════     ══════════
FileReadTool             GlobTool                  BashTool
FileEditTool             GrepTool                  PowerShellTool
FileWriteTool            ToolSearchTool
NotebookEditTool
                                                   INTERACTION
WEB & NETWORK            AGENT / TASK              ═══════════
════════════════         ══════════════════         AskUserQuestionTool
WebFetchTool             AgentTool                 BriefTool
WebSearchTool            SendMessageTool
                         TeamCreateTool            PLANNING & WORKFLOW
MCP PROTOCOL             TeamDeleteTool            ════════════════════
══════════════           TaskCreateTool            EnterPlanModeTool
MCPTool                  TaskGetTool               ExitPlanModeTool
ListMcpResourcesTool     TaskUpdateTool            EnterWorktreeTool
ReadMcpResourceTool      TaskListTool              ExitWorktreeTool
                         TaskStopTool              TodoWriteTool
SKILLS                   TaskOutputTool
══════════                                         SYSTEM
SkillTool                                          ════════
LSPTool                                            ConfigTool
                                                   ScheduleCronTool
```

### 4.3 Permission System

The permission system is the core safety mechanism, with multi-level checks before every tool call:

```
TOOL CALL REQUEST
      │
      ▼
validateInput()           ← reject invalid inputs first
      │
      ▼
PreToolUse Hooks          ← user-defined shell command hooks (settings.json)
      │                     can: approve, deny, or modify input
      ▼
Permission Rules          ← rule engine
  alwaysAllow: match → auto-approve
  alwaysDeny:  match → auto-deny
  alwaysAsk:   match → always prompt
  Sources: settings.json, CLI args, session decisions
      │
      ▼ (no rule match)
Auto Classifier           ← (YOLO/auto mode) AI classifier judges safety
      │
      ▼ (classifier uncertain)
Interactive Dialog        ← user sees tool name + input
  Options: Allow Once / Allow Always / Deny
      │
      ▼
checkPermissions()        ← tool-specific security logic (path sandboxing, etc.)
      │
      ▼
APPROVED → tool.call()
```

**Permission modes**:
- **Default**: Dangerous operations require confirmation every time
- **Plan Mode**: Only read operations, modifications blocked
- **Auto Mode (YOLO)**: AI auto-judges, no human confirmation

### 4.4 Context Compaction

When conversation history exceeds the model's context window, compaction is triggered automatically:

**Three compaction strategies**:

| Strategy | Trigger | Mechanism |
|---------|---------|-----------|
| `autoCompact` | Token count exceeds threshold | Calls Claude API to summarize older messages |
| `snipCompact` | `HISTORY_SNIP` feature flag | Removes zombie messages and stale markers |
| `contextCollapse` | `CONTEXT_COLLAPSE` feature flag | Restructures context for efficiency |

**Post-compaction message structure**:

```
messages[]
  ├── [compacted summary of older messages]
  ├── [compact_boundary marker]
  ├── [recent messages — full fidelity]
  │     user → assistant → tool_use → tool_result
  └── [current turn]
```

**Key constants**:
- `AUTOCOMPACT_BUFFER_TOKENS = 13,000` (compaction trigger buffer)
- `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` (circuit breaker)
- `POST_COMPACT_MAX_FILES_TO_RESTORE = 5` (files to re-inject after compaction)

### 4.5 Multi-Agent Collaboration

Claude Code supports multiple sub-agent modes:

#### Sub-Agent Types

| Type | Creation | Characteristics |
|------|---------|----------------|
| `local_agent` | AgentTool (default) | Runs in process, fresh messages[] |
| `fork` | AgentTool (fork mode) | Child process, shared file cache, new message queue |
| `worktree` | EnterWorktreeTool | Isolated git worktree + fork subprocess |
| `remote_agent` | Bridge layer | Connected to remote container/session via bridge |
| `in_process_teammate` | TeamCreateTool | Same process, user-visible execution |

#### Inter-Agent Communication

```
Agent communication tools:
├── SendMessageTool     → agent-to-agent messaging (request-response)
├── TaskCreate/Update   → shared task board
└── TeamCreate/Delete   → team lifecycle management
```

#### Coordinator Mode

Special execution mode where one coordinator agent orchestrates multiple workers:

- Coordinator handles: planning, task delegation, result synthesis
- Workers handle: research, implementation, verification
- Enabled via `CLAUDE_CODE_COORDINATOR_MODE` env var

#### Swarm Mode (feature-gated)

```
Lead Agent
  ├── Worker A ──> claims Task 1
  ├── Worker B ──> claims Task 2
  └── Worker C ──> claims Task 3
  
Shared: task board, message inbox
Isolated: messages[], file cache, working directory
```

### 4.6 MCP Protocol Integration

MCP (Model Context Protocol) is Anthropic's tool extension protocol, allowing external services to provide additional tools to Claude Code:

**Supported transports**:

| Transport | Description |
|---------|-------------|
| `stdio` | Spawn child process, communicate via stdin/stdout |
| `sse` | HTTP Server-Sent Events (EventSource) |
| `http` | Streamable HTTP |
| `ws` | WebSocket |
| `sdk` | In-process transport |

**Tool naming convention**: `mcp__<server>__<tool>`

**Authentication**:
- OAuth 2.0 (McpOAuthConfig)
- Cross-App Access (XAA / SEP-990)
- Header API Key

### 4.7 Session Persistence

```
~/.claude/projects/<path-hash>/sessions/
└── <session-id>.jsonl          ← append-only log
    ├── {"type":"user",...}
    ├── {"type":"assistant",...}
    ├── {"type":"progress",...}  ← tool execution progress
    └── {"type":"system","subtype":"compact_boundary",...}
```

**Resume modes**:
- `--continue`: Resume last session in cwd
- `--resume <id>`: Resume specific session
- `--fork-session`: New session ID, copy history

**Write strategy (performance optimization)**:

| Message type | Strategy |
|-------------|---------|
| User messages | `await` (blocking, for crash recovery) |
| Assistant messages | Fire-and-forget (order-preserving queue) |
| Progress messages | Inline write (deduped on next query) |

### 4.8 Slash Command System

~80 slash commands covering:

| Category | Example commands |
|---------|----------------|
| Authentication | `/login`, `/logout` |
| Configuration | `/config`, `/model` |
| Agent management | `/agents`, `/fork` |
| Context | `/compact`, `/clear` |
| Memory | `/memory` |
| MCP services | `/mcp` |
| Code review | `/review` |
| Plan mode | `/plan` |
| Session resume | `/resume` |
| Help | `/help`, `/doctor` |
| Hidden commands | `/btw`, `/stickers`, `/thinkback` |

### 4.9 Memory System (CLAUDE.md)

Claude Code provides project context to the model through layered `CLAUDE.md` files:

**Memory file search order**:

```
~/.claude/CLAUDE.md              ← global memory
<project-root>/CLAUDE.md         ← project-level memory
<cwd>/CLAUDE.md                  ← current directory memory
```

**Memory loading strategy**:
- **Lazy loading**: Injected via tool_result, not in system prompt
- **Directory-aware**: Traverses upward from current working directory
- **Skills**: `SkillTool` can load domain-specific knowledge documents

---

## 5. Technical Highlights

### 5.1 The 12 Progressive Harness Mechanisms

The most important technical insight from this codebase: **building a production-grade AI agent requires 12 layered mechanisms on top of the basic loop**.

```
s01  THE LOOP
     "One loop & Bash is all you need"
     query.ts: while-true → Claude API → check stop_reason → execute tools → append results

s02  TOOL DISPATCH
     "Adding a tool = adding one handler"
     Tool.ts + tools.ts: tools register into dispatch map. Loop stays identical.
     buildTool() factory provides safe defaults.

s03  PLANNING
     "An agent without a plan drifts"
     EnterPlanModeTool/ExitPlanModeTool + TodoWriteTool:
     list steps first, then execute. Reportedly doubles completion rate.

s04  SUB-AGENTS
     "Break big tasks; clean context per subtask"
     AgentTool + forkSubagent.ts: each child gets fresh messages[],
     keeping the main conversation clean.

s05  KNOWLEDGE ON DEMAND
     "Load knowledge when you need it"
     SkillTool + memdir/: inject via tool_result, not system prompt.
     CLAUDE.md files loaded lazily per directory.

s06  CONTEXT COMPRESSION
     "Context fills up; make room"
     services/compact/: autoCompact (summarize) + snipCompact (trim) + contextCollapse

s07  PERSISTENT TASKS
     "Big goals → small tasks → disk"
     TaskCreate/Update/Get/List: file-based task graph with
     status tracking, dependencies, and persistence.

s08  BACKGROUND TASKS
     "Slow ops in background; agent keeps thinking"
     DreamTask + LocalShellTask: daemon threads run commands,
     inject notifications on completion.

s09  AGENT TEAMS
     "Too big for one → delegate to teammates"
     TeamCreate/Delete + InProcessTeammateTask: persistent
     teammates with async mailboxes.

s10  TEAM PROTOCOLS
     "Shared communication rules"
     SendMessageTool: one request-response pattern drives
     all negotiation between agents.

s11  AUTONOMOUS AGENTS
     "Teammates scan and claim tasks themselves"
     coordinator/coordinatorMode.ts: idle cycle + auto-claim,
     no need for lead to assign each task.

s12  WORKTREE ISOLATION
     "Each works in its own directory"
     EnterWorktreeTool/ExitWorktreeTool: tasks manage goals,
     worktrees manage directories, bound by ID.
```

### 5.2 Key Design Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| **AsyncGenerator streaming** | `QueryEngine`, `query()` | Full-chain streaming from API to consumer |
| **Builder + Factory** | `buildTool()` | Safe defaults for tool definitions |
| **Branded Types** | `SystemPrompt`, `asSystemPrompt()` | Prevent string/array confusion |
| **Feature Flags + DCE** | `feature()` from `bun:bundle` | Compile-time dead code elimination |
| **Discriminated Unions** | `Message` types | Type-safe message handling |
| **Observer + State Machine** | `StreamingToolExecutor` | Tool execution lifecycle tracking |
| **Snapshot State** | `FileHistoryState` | Undo/redo for file operations |
| **Ring Buffer** | Error log | Bounded memory for long sessions |
| **Fire-and-Forget Write** | `recordTranscript()` | Non-blocking persistence with ordering |
| **Lazy Schema** | `lazySchema()` | Defer Zod schema evaluation for performance |
| **Context Isolation** | `AsyncLocalStorage` | Per-agent context in shared process |
| **Circuit Breaker** | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | Auto-compaction failure protection |

### 5.3 State Management Architecture

```typescript
// AppState is the single source of truth
export type AppState = DeepImmutable<{
  settings: SettingsJson         // config file contents
  verbose: boolean
  mainLoopModel: ModelSetting    // active model
  toolPermissionContext: {       // permission context
    mode: PermissionMode
    alwaysAllowRules: Rule[]
    alwaysDenyRules: Rule[]
    alwaysAskRules: Rule[]
    isBypassPermissionsModeAvailable: boolean
  }
  thinkingEnabled: boolean       // extended thinking mode
  // ... more fields
}> & {
  // Mutable section (contains function references, cannot be deep-frozen)
  tasks: { [taskId: string]: TaskState }
  fileHistory: FileHistoryState
  mcp: { clients, tools, commands, resources }
  plugins: { enabled, disabled, commands, errors }
}

// React integration
const AppStateProvider    // creates store via createContext
const useAppState(sel)    // selector-based subscriptions
const useSetAppState()    // immer-style updater function
```

**Design highlights**: `DeepImmutable` enforces immutability, selector subscriptions avoid unnecessary re-renders, `immer`-style updates ensure convenient immutable updates.

### 5.4 Terminal UI with React/Ink

Claude Code uses **React + Ink** to render interactive UI in the terminal — a rare architectural choice:

- **Component tree**: Same component model as regular React, but renders to stdout
- **Every tool** has its own React rendering methods:
  - `renderToolUseMessage()` — display tool call input
  - `renderToolResultMessage()` — display output
  - `renderToolUseProgressMessage()` — progress animation
  - `renderGroupedToolUse()` — parallel tool group display

**UI components** (40+ component groups):

```
src/components/
├── design-system/         # Reusable UI primitives
├── messages/              # Message rendering
├── permissions/           # Permission confirmation dialogs
├── PromptInput/           # Input field + autocomplete
├── LogoV2/                # Branding + welcome screen
├── Settings/              # Settings panels
└── Spinner.tsx            # Loading animations
```

---

## 6. Hidden Features & Deep Discoveries

### 6.1 Telemetry & Privacy

- **Dual analytics pipeline**: First-party (Anthropic) + third-party (Datadog)
- **Collected data**: Environment fingerprint, process metrics, session/user IDs, repo URL hash
- **No opt-out**: Direct Anthropic API users cannot disable first-party logging
- **Full tool input capture**: Set `OTEL_LOG_TOOL_DETAILS=1` to enable
- **GrowthBook A/B testing**: Users assigned to experiment groups without awareness

### 6.2 Undercover Mode

Anthropic employees automatically enter undercover mode when working in public/open-source repos:

```
## UNDERCOVER MODE — CRITICAL
Do not blow your cover.
NEVER include: internal codenames, unreleased version numbers,
"Claude Code" wording, Co-Authored-By attribution.
Write commit messages like a human developer.
```

- No force-OFF option ("There is NO force-OFF")
- Dead-code-eliminated in external builds, never executes

### 6.3 Internal vs External User Differences

| Dimension | External users | Internal (ant) |
|---------|---------------|---------------|
| Output style | "Be concise" | "Lean toward more explanation" |
| Hallucination mitigation | None | Capybara v8 dedicated patches |
| Verification agents | None | Required for non-trivial changes |
| Hidden commands | None | `/btw`, `/stickers`, `/thinkback`, etc. |

### 6.4 Remote Control Mechanisms

- **Hourly polling** of `/api/claude_code/settings`
- **"Accept or exit" dialog**: Rejecting dangerous settings = program exits
- **6+ kill switches**: Can remotely disable permission bypass, fast mode, voice mode, analytics
- **GrowthBook feature flags**: Can change any user's behavior without consent

### 6.5 Buddy System (Virtual Pet)

Fully implemented but unreleased virtual companion system:
- **18 species**: duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, bunny, mushroom, chonk
- **5 rarity tiers**: Common (60%) / Uncommon (25%) / Rare (10%) / Epic (4%) / Legendary (1%)
- **1% shiny probability**: Shiny variant of any species
- **Deterministic generation based on user ID hash** (same user always gets same companion)
- **7 hats**: crown, top hat, propeller hat, halo, wizard hat, beanie, ducky hat

---

## 7. Future Roadmap

### 7.1 Next-Generation Models

```
Codename evolution:
Fennec → Opus 4.6 → [Numbat?]
Capybara → Sonnet v8 → [?]

Confirmed in development:
- Opus 4.7
- Sonnet 4.8
- Numbat (next-generation flagship model codename)
```

### 7.2 KAIROS — Fully Autonomous Agent Mode

The biggest unreleased feature, transforming Claude Code from passive assistant to active autonomous agent:

```
System prompt excerpt:
"You are operating autonomously."
"You will receive <tick> prompts to stay active."
"If there's nothing useful to do, call SleepTool."
"Bias toward action — read files, make changes, commit, without asking."

Terminal focus awareness:
- Unfocused: User has stepped away. Heavily bias toward autonomous action.
- Focused: User is watching. Be more collaborative.
```

Associated tools:
- **SleepTool**: Control pacing between autonomous operations
- **PushNotificationTool**: Proactively push notifications to user's device
- **SubscribePRTool**: Subscribe to GitHub PR webhook events
- **BriefTool**: Proactive status updates

### 7.3 Voice Mode

Push-to-talk voice input fully implemented, gated by `VOICE_MODE` feature flag:
- Connects to Anthropic's `voice_stream` WebSocket endpoint
- OAuth users only (not API Key / Bedrock / Vertex)
- mTLS secure connection

### 7.4 17 Unreleased Tools

| Tool | Description |
|------|-------------|
| `WebBrowserTool` | Built-in browser automation (codename: bagel) |
| `TerminalCaptureTool` | Terminal panel capture and monitoring |
| `WorkflowTool` | Execute predefined workflow scripts |
| `MonitorTool` | MCP monitoring |
| `SnipTool` | Conversation history snipping |
| `ListPeersTool` | Unix domain socket peer discovery |
| `SubscribePRTool` | GitHub PR webhook subscription |
| `REPLTool` | Interactive REPL (VM sandbox) |
| `VerifyPlanExecutionTool` | Plan verification |
| ... | |

### 7.5 Three Strategic Directions

1. **New models**: Numbat (next-gen), Opus 4.7, Sonnet 4.8 in development
2. **Autonomous agents**: KAIROS mode — unattended operation, proactive actions, push notifications
3. **Multimodal**: Voice input ready, browser tool pending, workflow automation coming soon

> Claude Code is evolving from a **coding assistant** into a **24/7 autonomous development agent**.

---

## 8. Code Quality Observations

### 8.1 Engineering Practices

- **Zod** for runtime schema validation of all tool inputs
- **TypeScript** `strict: false` (historical debt, primarily inferred types)
- **Branded Types** prevent accidental type confusion (e.g., `SystemPrompt` vs `string`)
- **AsyncGenerator** for full-chain streaming, no intermediate buffering
- **AsyncLocalStorage** for per-agent isolated context (analogous to thread-local storage in Node.js)

### 8.2 Performance Design

- **Parallel tool execution**: `StreamingToolExecutor` automatically parallelizes concurrency-safe tools
- **LRU file cache**: `readFileCache` avoids repeated file reads
- **Lazy Schema**: `lazySchema()` defers Zod schema evaluation
- **Fire-and-Forget writes**: Async session transcript persistence
- **Speculative execution**: Pre-execute likely tool calls

### 8.3 Reliability Design

- **Circuit Breaker**: Stop retrying after 3 consecutive compaction failures
- **Exponential backoff**: API retries and bridge reconnection both have backoff
- **Crash recovery**: User messages are blocking writes to ensure recoverability
- **Ring Buffer error log**: Prevents memory overflow in long sessions

### 8.4 Compile-Time Optimization (Bun)

```
feature('FLAG_NAME')
  ├── true  → code kept in bundle (Anthropic internal build)
  └── false → code dead-code-eliminated (published npm package)
```

This ensures the 12MB published bundle only contains external-user-facing code; 108 internal modules are completely stripped.

---

## 9. Key Learning Takeaways

### Core Insights for Building AI Agents

**1. Start with the minimal loop**  
Agent = while-true + API call + tool execution. Master this minimal closed loop, then add production mechanisms.

**2. Tool system design**  
Use Builder pattern to define tools, decoupling tool registration from loop logic. Each tool declares: capabilities (read-only/destructive/concurrency-safe), permission requirements, UI rendering.

**3. Permission systems can't be simplified**  
Three-level permissions (config rules → auto classifier → interactive prompt) are necessary for production. "Allow/Deny" binary is insufficient.

**4. Context management is key for long sessions**  
Compression strategies (summarization + trimming) enable the agent to handle tasks exceeding the context window. Circuit breakers prevent infinite retries.

**5. Sub-agents need isolated context**  
Each sub-agent having independent messages[] is a core architectural decision: keeps main conversation clean, prevents context pollution.

**6. Load knowledge on demand**  
Injecting knowledge via tool_result rather than system prompt means context window is only consumed when needed.

**7. Persistence strategies should be differentiated**  
Different importance levels deserve different write strategies: critical data blocks, secondary data fires-and-forgets.

**8. Terminal UI can use React**  
Ink + React brings declarative UI to the terminal, dramatically improving development efficiency for complex interactive interfaces.

### Recommended Deep-Dive Files

| Topic | Recommended files |
|-------|-----------------|
| Agent loop | `src/query.ts` |
| Tool interface design | `src/Tool.ts` + `src/tools/BashTool/` |
| Permission system | `src/hooks/useCanUseTool.tsx` + `src/utils/permissions/` |
| Context compaction | `src/services/compact/` |
| Multi-agent collaboration | `src/tasks/` + `src/coordinator/` |
| MCP integration | `src/services/mcp/` |
| State management | `src/state/AppStateStore.ts` |
| Session persistence | `src/history.ts` + `src/QueryEngine.ts` |
| Telemetry system | `src/services/analytics/` |
| Build optimization | `stubs/bun-bundle.ts` + `scripts/build.mjs` |

---

> **Disclaimer**: This document is based on analysis of Claude Code v2.1.88 source code, for technical research and learning purposes only. All source code copyright belongs to Anthropic and Claude. Commercial use is strictly prohibited.
