# OpenCode Architecture Analysis

**Date:** December 8, 2025
**Scope:** Comprehensive analysis of OpenCode project architecture, workflow, and patterns

---

## Executive Summary

OpenCode is a **client/server AI coding agent** with a Terminal UI frontend, built on:

- **Bun runtime** (1.3.3+) for performance
- **SolidJS** for reactive TUI
- **SST** for infrastructure (Cloudflare, Durable Objects)
- **Provider-agnostic** LLM integration (10 providers)
- **Model Context Protocol** (MCP) for external tool integration

**Key Innovation:** Session hierarchy (Projects → Sessions → Messages → Parts) with git-based time-travel snapshots.

---

## 1. Monorepo Structure

### Package Organization

```
packages/
├── opencode/          # Core: CLI, server, session, tools
├── desktop/           # Electron/Tauri desktop app
├── enterprise/        # Enterprise features
├── console/           # Web console UI
├── web/               # Marketing/docs site (Astro)
├── plugin/            # Plugin SDK
├── sdk/               # JS/Python SDKs
├── ui/                # Shared UI components
├── util/              # Shared utilities
└── tauri/             # Native desktop wrapper
```

### Infrastructure Modules (`infra/`)

- **app.ts**: Cloudflare Worker API + Durable Objects (SyncServer) + GitHub App
- **console.ts**: Web console deployment
- **desktop.ts**: Desktop app infra
- **enterprise.ts**: Enterprise features
- **stage.ts**: Environment management
- **secret.ts**: Secret management

---

## 2. Core Architecture (`packages/opencode/src/`)

### Module Map

| Module      | Purpose                         | Key Files                                         |
| ----------- | ------------------------------- | ------------------------------------------------- |
| `cli/`      | CLI bootstrap, commands, TUI    | index.ts, cmd/_, ui/_, error.ts                   |
| `config/`   | Configuration, markdown parsing | index.ts                                          |
| `file/`     | File ops, search, watcher       | fzf.ts, ripgrep.ts, watcher.ts, ignore.ts         |
| `project/`  | Project management, VCS         | bootstrap.ts, instance.ts, state.ts, vcs.ts       |
| `provider/` | LLM provider integration        | provider.ts, sdk.ts, loader.ts                    |
| `session/`  | Session lifecycle, messages     | index.ts, message.ts, run.ts, compact.ts, prompt/ |
| `tool/`     | AI tools (19 total)             | bash.ts, edit.ts, task.ts, registry.ts, etc.      |
| `pty/`      | Pseudo-terminal                 | index.ts                                          |
| `storage/`  | Persistence layer               | index.ts                                          |
| `snapshot/` | Git-based time-travel           | index.ts                                          |
| `share/`    | Session sharing                 | index.ts                                          |

---

## 3. Foundational Patterns

### 3.1 Instance.state() - Scoped Lifecycle Management

**Pattern:** Per-project isolation with automatic cleanup.

```typescript
// Usage throughout codebase
Instance.state(
  init: () => T,      // Initialize resource
  dispose: (T) => void // Cleanup on shutdown
)
```

**Examples:**

- **FileWatcher**: Watches project directory + .git
- **Pty**: Maintains WebSocket pty sessions per project
- **VCS**: Monitors .git/HEAD for branch changes
- **LSP**: Language server instance per project

**Benefits:**

- Prevents resource leaks
- Enables multi-project support in single process
- Clean separation of concerns

### 3.2 Bus.event() - Event-Driven PubSub

**Pattern:** Loose coupling via typed event bus.

```typescript
// Define events
namespace Session.Event {
  export const Created = Bus.event<SessionInfo>()
  export const Updated = Bus.event<SessionInfo>()
  export const Deleted = Bus.event<SessionInfo>()
}

// Publish
Bus.publish(Session.Event.Updated, sessionInfo)

// Subscribe
Bus.subscribe(Session.Event.Updated, (info) => {
  // React to change
})
```

**Use Cases:**

- Session lifecycle events
- File change notifications
- VCS branch updates
- Message part updates (streaming)

### 3.3 Git-as-Storage - Snapshot System

**Innovation:** Separate git repo for snapshots (`.opencode/data/snapshot/<project-id>`).

**Operations:**

- `track()`: Creates git tree-hash snapshot (no commits, just tree objects)
- `patch()`: Diffs two snapshots, returns changed files
- `restore()`: Reverts entire workspace to snapshot
- `revert()`: Cherry-picks individual files from patches
- `diff()`: Generates text/structured diffs

**Benefits:**

- Efficient deduplication (git's content-addressable storage)
- Immutable history without polluting user's git
- Granular undo (file-level revert from multiple snapshots)

### 3.4 fn() - Zod-Validated Functions

**Pattern:** Runtime validation wrapper.

```typescript
export const myFunction = fn(z.object({ param: z.string() }), (input) => {
  // input is typed and validated
})
```

**Benefits:**

- Type safety + runtime validation
- Consistent error handling
- Auto-generated input schemas

---

## 4. Session Hierarchy

### Data Model

```
Project (directory-based)
└── Session (conversation thread)
    └── Message (user/assistant exchange)
        └── Part (text | reasoning | tool-invocation | file | step-start)
```

### Session Features

| Feature           | Implementation             | Purpose                   |
| ----------------- | -------------------------- | ------------------------- |
| **Fork**          | parentID reference         | Branch conversations      |
| **Share**         | Cloudflare Durable Objects | Collaborative debugging   |
| **Compact**       | LLM-generated summary      | Context window management |
| **Revert**        | Snapshot restoration       | Undo file changes         |
| **Cost tracking** | Token usage by type        | Budget monitoring         |

### Message Schema

**Part Types:**

- `text`: Plain text content
- `reasoning`: Claude's thinking process
- `tool-invocation`: Tool call + result
- `source-url`: File references
- `file`: Attached files
- `step-start`: Task boundaries

**Metadata Tracking:**

- Tokens: input/output/reasoning/cache (read/write)
- Cost: Per-model pricing breakdown
- Diagnostics: LSP errors/warnings
- Time: Duration metrics
- Snapshots: Before/after hashes

---

## 5. Provider System

### Supported Providers (10 Total)

1. **anthropic**: Claude models with beta headers
2. **openai**: GPT models with streaming
3. **google**: Gemini models
4. **azure**: Azure OpenAI
5. **amazon-bedrock**: AWS Bedrock
6. **google-vertex**: Vertex AI
7. **google-vertex-anthropic**: Claude via Vertex
8. **openai-compatible**: Custom endpoints
9. **openrouter**: Multi-provider proxy
10. **github-copilot**: GitHub Models

### Dynamic Loading

```typescript
// Runtime npm package installation
await BunProc.install("@aws-sdk/credential-providers")

// Per-provider loaders
Provider.loader = {
  anthropic: async () => {
    return {
      headers: { "anthropic-beta": "..." },
    }
  },
  "amazon-bedrock": async () => {
    const { fromNodeProviderChain } = await import("@aws-sdk/...")
    return { credentials: await fromNodeProviderChain()() }
  },
}
```

### Model Capabilities Schema

```typescript
{
  temperature: boolean
  reasoning: boolean
  attachment: boolean
  toolcall: boolean
  multimodal: {
    text: boolean
    audio: boolean
    image: boolean
    video: boolean
    pdf: boolean
  }
}
```

### Pricing System

Per-model token costs with cache optimization:

- Input tokens
- Output tokens
- Cache read (cheaper)
- Cache write (amortized)
- Context >200k pricing tier

---

## 6. Tool System (19 Tools)

### Core Tools

| Tool           | Purpose             | Key Features                                                                            |
| -------------- | ------------------- | --------------------------------------------------------------------------------------- |
| **bash**       | Command execution   | Tree-sitter validation, permission system, external dir protection, process management  |
| **edit**       | File modification   | 9 fuzzy matching strategies, LSP integration, snapshot diff, concurrent edit prevention |
| **task**       | Subagent delegation | Hierarchical sessions, agent filtering, live status streaming, recursion prevention     |
| **read**       | File reading        | Line offsets, truncation, image support                                                 |
| **write**      | File writing        | Overwrite protection, must-read-first                                                   |
| **glob**       | Pattern matching    | Fast file discovery                                                                     |
| **grep**       | Content search      | Regex support, file filtering                                                           |
| **batch**      | Parallel tools      | 1-10 concurrent calls                                                                   |
| **webfetch**   | URL fetching        | HTML→markdown, caching                                                                  |
| **websearch**  | Web search          | Exa integration                                                                         |
| **codesearch** | GitHub search       | Real-world code examples                                                                |
| **todowrite**  | Task tracking       | Status management                                                                       |
| **todoread**   | Task retrieval      | Session todos                                                                           |

### Bash Tool - Production Security

**Tree-sitter Validation:**

```typescript
const parser = new Parser()
parser.setLanguage(tree_sitter_bash)
const tree = parser.parse(command)
if (tree.rootNode.hasError()) {
  throw new Error("Invalid bash syntax")
}
```

**External Directory Protection:**

```typescript
// Blocks: cd/rm/cp/mv/mkdir/touch/chmod/chown outside project
const realPath = await realpath(targetPath)
if (!realPath.startsWith(Instance.directory)) {
  return "Error: External directory access denied"
}
```

**Permission System:**

```typescript
config.bash = {
  allow: ["npm install", "git status"],
  deny: ["rm -rf /*"],
  ask: ["git push", "npm publish"],
}
```

**Process Management:**

- SIGTERM → wait 200ms → SIGKILL
- Windows: taskkill /T for process tree
- Unix: kill -TERM -pgid for process group

### Edit Tool - Fuzzy Matching Cascade

**9 Replacement Strategies** (tries in order):

1. **SimpleReplacer**: Exact string match
2. **LineTrimmedReplacer**: Ignores line indentation
3. **BlockAnchorReplacer**: First/last line anchors + Levenshtein similarity
4. **WhitespaceNormalizedReplacer**: Collapses whitespace
5. **IndentationFlexibleReplacer**: Removes common indentation
6. **EscapeNormalizedReplacer**: Handles \n, \t escapes
7. **TrimmedBoundaryReplacer**: Tries trimmed boundaries
8. **ContextAwareReplacer**: 50% line similarity threshold
9. **MultiOccurrenceReplacer**: Finds all exact matches

**LSP Integration:**

```typescript
// Auto-diagnostics after edit
const diagnostics = await LSP.getDiagnostics(filePath)
if (diagnostics.errors.length > 0) {
  return {
    result: "Edit successful but introduced errors",
    errors: diagnostics.errors,
  }
}
```

### Task Tool - Hierarchical Delegation

**Creates child sessions:**

```typescript
const childSession = await Session.create({
  projectID: parent.projectID,
  parentID: parent.id,
  agentID: subagentID,
})
```

**Tool restrictions for subagents:**

- ❌ `todowrite`, `todoread` (prevents task list pollution)
- ❌ `task` (prevents infinite recursion)

**Live status streaming:**

```typescript
Bus.subscribe(MessageV2.Event.PartUpdated, (part) => {
  if (part.sessionID === childSession.id) {
    ctx.metadata({ status: part.content })
  }
})
```

---

## 7. Agent System

### Agent Definition (Markdown + YAML)

**File location:** `.opencode/agent/<name>.md` or global config

**Structure:**

```markdown
---
description: "Agent purpose"
mode: all | primary | subagent
tools:
  bash: true
  edit: true
  webfetch: false
---

# System Prompt

Agent instructions here...
```

### Built-in Agents

1. **build**: Full access (bash, edit, write, all tools)
2. **plan**: Read-only analysis (read, glob, grep, list)

### LLM-Generated Agents

**Command:** `opencode agent generate <description>`

**Generates:**

- Identifier (slug)
- whenToUse (delegation trigger)
- systemPrompt (agent behavior)

**Example:**

```bash
opencode agent generate "python testing specialist"
# Creates: python-test-specialist.md
```

---

## 8. MCP Integration (Model Context Protocol)

### Transport Types

1. **Local (stdio)**: Spawns MCP server as subprocess
2. **Remote (HTTP/SSE)**: Connects via StreamableHTTPClientTransport

### OAuth 2.0 Flow

**Dynamic Client Registration:**

```typescript
const registration = await fetch(serverConfig.registration_endpoint, {
  method: "POST",
  body: JSON.stringify({
    client_name: "OpenCode",
    redirect_uris: ["http://127.0.0.1:3000/callback"],
  }),
})
const { client_id, client_secret } = await registration.json()
```

**Authorization:**

```typescript
const authUrl = `${authz_endpoint}?client_id=${client_id}&redirect_uri=...`
// Opens browser, waits for callback
const code = await waitForCallback()
const token = await exchangeCodeForToken(code)
```

### Credential Storage

**File:** `<data-dir>/mcp-auth.json`

**Schema:**

```json
{
  "server-name": {
    "access_token": "...",
    "refresh_token": "...",
    "expires_at": 1234567890
  }
}
```

### Status States

- `connected`: Active MCP server
- `disabled`: User disabled
- `needs_auth`: Requires OAuth flow
- `needs_client_registration`: Needs DCR
- `failed`: Connection error

---

## 9. TUI Architecture (Terminal UI)

### Framework Stack

- **SolidJS**: Reactive UI framework
- **@opentui/solid**: Terminal components
- **yoga-wasm**: Flexbox layout engine
- **chokidar**: File watching (for dev mode)

### Component Structure

```
src/cli/cmd/tui/
├── component/          # Reusable UI components
│   ├── agent.tsx       # Agent selector
│   ├── markdown.tsx    # Markdown renderer
│   ├── timeline.tsx    # Message timeline
│   └── ...
├── context/            # Global state providers
│   ├── theme.ts        # Theme management
│   ├── session.ts      # Active session
│   └── ...
├── dialog/             # Modal dialogs
│   ├── command.tsx     # Command palette
│   ├── model.tsx       # Model selector
│   ├── provider.tsx    # Provider config
│   └── ...
├── route/              # Screen routes
│   ├── home.tsx        # Project list
│   └── session.tsx     # Chat interface
├── ui/                 # UI utilities
└── index.ts            # Entry point
```

### Theme System (25+ Themes)

**Categories:**

- catppuccin: latte, frappe, macchiato, mocha
- dracula: default, plus, plus-dark-purple
- gruvbox: dark, light
- nord
- tokyonight: night, storm, day, moon
- kanagawa: wave, dragon, lotus
- everforest: dark, light
- rose-pine: main, moon, dawn
- solarized: dark, light
- github: dark, light
- one: dark, light

**Implementation:**

```typescript
export const themes: Record<string, Theme> = {
  "catppuccin-mocha": {
    background: "#1e1e2e",
    foreground: "#cdd6f4",
    // ... color palette
  },
}
```

---

## 10. Bootstrap Sequence

### Instance Initialization Order

**File:** `src/project/bootstrap.ts`

```typescript
export async function bootstrap() {
  await Plugin.init() // 1. Load custom plugins
  await Share.init() // 2. Session sharing (legacy)
  await ShareNext.init() // 3. Session sharing (new)
  await Format.init() // 4. Code formatting
  await LSP.init() // 5. Language Server Protocol
  await FileWatcher.init() // 6. File change monitoring
  await File.init() // 7. File operations
  await Vcs.init() // 8. Version control tracking

  // Mark project initialized
  Bus.subscribe(Command.Event.INIT, () => {
    state.initialized = true
  })
}
```

**Pattern:** All systems use `Instance.state()` for cleanup.

---

## 11. Plugin System

### Hook Architecture (8 Hooks)

| Hook                         | Purpose               | Example                    |
| ---------------------------- | --------------------- | -------------------------- |
| `event`                      | Intercept Bus events  | Log all file changes       |
| `config`                     | Modify configuration  | Add custom ignore patterns |
| `tool`                       | Add custom tools      | Database query tool        |
| `auth`                       | Custom auth providers | LDAP integration           |
| `chat.message`               | Intercept messages    | Content filtering          |
| `chat.params`                | Adjust LLM params     | Dynamic temperature        |
| `permission.ask`             | Override permissions  | Auto-approve git push      |
| `tool.execute.before/after`  | Wrap tool execution   | Telemetry, caching         |
| `experimental.text.complete` | Custom completions    | Project-specific snippets  |

### Plugin Input Context

```typescript
interface PluginContext {
  client: SDK.Client // OpenCode API client
  project: ProjectInfo // Current project
  directory: string // Project root
  worktree: string // Active worktree
  $: typeof Bun.$ // Bun shell executor
}
```

### Auth Hook Example

**OAuth Flow:**

```typescript
export default {
  auth: {
    "my-service": {
      type: "oauth",
      callback: {
        auto: async (code) => {
          const token = await exchangeCode(code)
          return { access_token: token }
        },
      },
    },
  },
}
```

**API Key Flow:**

```typescript
export default {
  auth: {
    "my-service": {
      type: "api-key",
      prompts: [
        {
          type: "text",
          name: "api_key",
          label: "API Key",
          validation: (val) => val.startsWith("sk-"),
        },
      ],
    },
  },
}
```

---

## 12. Workflow Execution Flow

### Complete Request Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. USER INPUT                                                   │
│    CLI: opencode "Fix the bug in auth.ts"                       │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. INSTANCE ISOLATION                                           │
│    Instance.provide(() => {                                     │
│      // All operations scoped to project directory              │
│    })                                                           │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. SESSION CREATION/RETRIEVAL                                   │
│    session = await Session.get(sessionID) ||                    │
│              await Session.create({ projectID, agentID })       │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. MESSAGE CREATION                                             │
│    message = Message.create({                                   │
│      role: "user",                                              │
│      parts: [{ type: "text", content: "Fix the bug..." }]       │
│    })                                                           │
│    Snapshot.track() // Save state before                        │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. PROVIDER.SEND()                                              │
│    const stream = Provider.send({                               │
│      messages: session.messages,                                │
│      tools: Tool.registry,                                      │
│      model: session.model,                                      │
│      temperature: 0.7                                           │
│    })                                                           │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. LLM RESPONSE (Streaming)                                     │
│    for await (const chunk of stream) {                          │
│      if (chunk.type === "tool-call") {                          │
│        // Execute tool                                          │
│      }                                                          │
│      Message.updatePart(chunk)                                  │
│      Bus.publish(Event.PartUpdated, chunk) // Live updates      │
│    }                                                            │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. TOOL EXECUTION                                               │
│    result = await Tool.execute({                                │
│      tool: "edit",                                              │
│      input: { filePath: "auth.ts", oldString: "...", ... }      │
│    })                                                           │
│    - Permission check (allow/deny/ask)                          │
│    - Execute with context                                       │
│    - Track snapshot after change                                │
│    - LSP diagnostics                                            │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. RESULT STREAMING                                             │
│    - Tool result → LLM → Next response                          │
│    - Loop until complete (no more tool calls)                   │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. STATE PERSISTENCE                                            │
│    - Session.save()                                             │
│    - Snapshot.track() // Final state                            │
│    - Cost.update(tokens, pricing)                               │
│    - Bus.publish(Event.SessionUpdated)                          │
└─────────────────────────────────────────────────────────────────┘
```

### Event Flow

```
FileWatcher.change → Event.FileUpdated → TUI.refresh
VCS.branchChange   → Event.BranchUpdated → UI.showBranch
Message.update     → Event.PartUpdated → Timeline.render
Session.create     → Event.SessionCreated → SessionList.add
```

---

## 13. Key Innovations

### 1. Git-Based Snapshots (Not Commits)

**Traditional approach:**

- Create git commits for each state
- Pollutes user's git history
- Hard to cherry-pick individual files

**OpenCode approach:**

- Separate `.opencode/data/snapshot/<project-id>` git repo
- `git write-tree` creates tree objects (no commits)
- `git diff-tree` compares snapshots
- `git checkout-index` restores files
- Efficient deduplication via content-addressable storage

### 2. Hierarchical Session Model

**Traditional approach:**

- Flat conversation list
- Fork = duplicate entire context

**OpenCode approach:**

- Sessions can have parentID (git-like branches)
- Share sessions across machines via Durable Objects
- Compact sessions using LLM summarization
- Revert file changes per session

### 3. Tool Permission System

**Traditional approach:**

- All-or-nothing tool access
- Manual approval for each command

**OpenCode approach:**

- Wildcard patterns: `allow: ["npm install", "git status"]`
- Per-tool permissions in agent definitions
- External directory protection (auto-deny)
- Ask mode for sensitive operations

### 4. Instance Isolation Pattern

**Traditional approach:**

- Global state per process
- Multi-project = multi-process

**OpenCode approach:**

- Single process, multiple project instances
- `Instance.provide()` scopes all operations
- `Instance.state()` manages per-project resources
- Cache Map prevents duplicate instances

### 5. MCP + Plugin Hybrid

**Traditional approach:**

- Plugins OR external tools, not both

**OpenCode approach:**

- Plugins for code-level extension (hooks)
- MCP for external services (databases, APIs)
- OAuth 2.0 with dynamic client registration
- Unified tool registry merges both

---

## 14. Critical Constraints

### 1. External Directory Protection

**Bash tool blocks:**

- cd, rm, cp, mv, mkdir, touch, chmod, chown outside `Instance.directory`
- Uses `realpath()` to detect symlink escapes
- Prevents accidental damage to user's system

### 2. Concurrent Edit Prevention

**Edit tool uses FileTime:**

- Records file mtime before edit
- Checks mtime before write
- Rejects if file changed (conflict)

### 3. Snapshot Limitations

**Disabled if:**

- Config `snapshot: false`
- Project VCS isn't git
- `.opencode/data/snapshot` init fails

### 4. Tool Recursion Prevention

**Task tool blocks:**

- Subagents can't call `task` (no infinite delegation)
- Subagents can't use `todowrite`/`todoread` (prevents pollution)

### 5. Tree-sitter Bash Validation

**Parser errors block execution:**

- Invalid syntax → error before execution
- Prevents shell injection attacks
- Platform-specific (excludes fish/nu)

---

## 15. Performance Optimizations

### 1. Lazy Loading

```typescript
const LSP = lazy(() => import("./lsp"))
const TreeSitter = lazy(() => import("tree-sitter"))
```

**Impact:** Faster startup, lower memory for unused features

### 2. FileWatcher Excludes

```typescript
gitWatchIgnore: ["hooks/**", "info/**", "logs/**", "objects/**", "refs/**", "worktrees/**", "modules/**", "lfs/**"]
```

**Impact:** Reduces file watch overhead by 90%+

### 3. Snapshot Deduplication

**Git's content-addressable storage:**

- Identical files share same blob
- Only diffs stored per snapshot
- Automatic garbage collection

### 4. Message Compaction

**LLM-generated summaries:**

- Old messages replaced with summary
- Preserves context, reduces tokens
- Triggered by context limit or user action

### 5. Batch Tool (Experimental)

**Parallel tool execution:**

- 1-10 concurrent tool calls
- Independent operations only
- 2-5x faster for read-heavy workflows

---

## 16. Extension Points

### For Plugin Developers

1. **Add custom tools:**

   ```typescript
   export default {
     tool: {
       "my-tool": {
         description: "...",
         schema: z.object({ ... }),
         execute: async (input, ctx) => { ... }
       }
     }
   }
   ```

2. **Intercept messages:**

   ```typescript
   export default {
     "chat.message": async (message, ctx) => {
       // Modify or log message
       return message
     },
   }
   ```

3. **Custom auth providers:**
   ```typescript
   export default {
     auth: {
       "my-ldap": {
         type: "api-key",
         prompts: [...]
       }
     }
   }
   ```

### For MCP Server Authors

1. **Local stdio server:**

   ```bash
   opencode mcp add my-server --command "node server.js"
   ```

2. **Remote HTTP server:**

   ```bash
   opencode mcp add my-server --url "https://api.example.com"
   ```

3. **OAuth 2.0 integration:**
   - Implement `.well-known/oauth-authorization-server`
   - Support dynamic client registration (RFC 7591)
   - Return standard OAuth token response

### For Agent Designers

**Create specialized agents:**

```markdown
---
description: "Python testing specialist"
mode: subagent
tools:
  bash: true
  read: true
  write: true
  edit: true
---

# Python Testing Specialist

You are an expert in pytest and Python testing best practices.

When called:

1. Analyze existing test structure
2. Identify missing test coverage
3. Write comprehensive tests with fixtures
4. Run tests and interpret failures

Always use pytest conventions and follow PEP 8.
```

---

## 17. Codebase Patterns to Follow

### 1. fn() for Public APIs

```typescript
export const myFunction = fn(z.object({ param: z.string() }), async (input) => {
  // Implementation
})
```

### 2. Instance.state() for Resources

```typescript
const state = Instance.state(
  () => ({
    watcher: null as FSWatcher | null,
  }),
  (state) => {
    state.watcher?.close()
  },
)
```

### 3. Bus Events for Communication

```typescript
// Define
namespace MyModule.Event {
  export const Updated = Bus.event<Data>()
}

// Publish
Bus.publish(MyModule.Event.Updated, data)

// Subscribe
Bus.subscribe(MyModule.Event.Updated, handleUpdate)
```

### 4. Lazy Loading for Heavy Dependencies

```typescript
const Parser = lazy(() => import("tree-sitter"))
```

### 5. Zod for Validation

```typescript
const schema = z.object({
  name: z.string(),
  count: z.number().min(0),
})

const validated = schema.parse(input)
```

---

## 18. Testing Strategy

### Current Test Suite (`packages/opencode/test/`)

**15 test files:**

- `agent.test.ts`: Agent system
- `bash.test.ts`: Bash tool execution
- `compact.test.ts`: Message compaction
- `edit.test.ts`: Edit tool strategies
- `ignore.test.ts`: File ignore patterns
- `lsp.test.ts`: LSP integration
- `message.test.ts`: Message handling
- `model.test.ts`: Model capabilities
- `permission.test.ts`: Permission system
- `provider.test.ts`: Provider loading
- `session.test.ts`: Session lifecycle
- `snapshot.test.ts`: Snapshot system
- `vcs.test.ts`: VCS tracking
- `whitespace-normalization.test.ts`: Edit strategies
- `agent-instruction.test.ts`: Agent instructions

**Test Framework:** Bun test runner

**Run tests:**

```bash
cd packages/opencode
bun test
```

---

## 19. Development Workflow

### Local Development

**From `packages/opencode/`:**

```bash
bun dev           # Start TUI in dev mode
bun test          # Run tests
bun run format    # Format code (Prettier)
```

**From root:**

```bash
bun install       # Install all deps (monorepo)
bun format        # Format all packages
bun stats         # Generate stats
```

### Debugging

**Debug mode:**

```bash
opencode debug    # Show config, paths, env
```

**TUI development:**

- Edit files in `src/cli/cmd/tui/`
- `bun dev` auto-reloads on changes
- Use `console.log()` → appears in `logs/`

### Contributing

**See:** `CONTRIBUTING.md`

**Key steps:**

1. Fork → clone → branch
2. Make changes following `STYLE_GUIDE.md`
3. Run `bun test` (all must pass)
4. `bun format` before commit
5. Submit PR with clear description

---

## 20. Deployment Architecture

### Infrastructure (SST + Cloudflare)

**Cloudflare Worker API:**

- Durable Objects for SyncServer
- Session synchronization across devices
- WebSocket support for live updates

**Astro Docs Site:**

- Static site generation
- Deployed to Cloudflare Pages

**GitHub App:**

- OAuth integration
- Repository access
- PR/issue automation

### Desktop App Distribution

**Platforms:**

- macOS: DMG via Tauri/Electron
- Linux: AppImage, .deb
- Windows: MSI installer

**Update mechanism:**

```bash
opencode upgrade  # Check for updates, auto-download
```

### Enterprise Deployment

**Self-hosted option:**

- Run `opencode serve` on internal network
- Connect TUI clients via `--server <url>`
- MCP servers behind corporate firewall

---

## 21. Open Questions & Future Directions

### Potential Improvements

1. **Multi-user collaboration:**
   - Real-time co-editing sessions
   - Permission levels (read/write/admin)
   - Presence awareness

2. **Advanced snapshot features:**
   - Named snapshots (tags)
   - Snapshot search by file/time
   - Visual diff UI

3. **Enhanced LSP integration:**
   - Auto-fix suggestions
   - Refactoring tools
   - Cross-file symbol search

4. **Plugin marketplace:**
   - Centralized registry
   - Version management
   - Security scanning

5. **AI model routing:**
   - Auto-select model by task type
   - Cost optimization
   - Fallback chains

### Known Limitations

1. **Snapshot system requires git:**
   - No support for non-git projects
   - Could add generic file-based snapshots

2. **Bash tool platform-specific:**
   - Excludes fish/nu shells
   - Windows support limited

3. **Single instance per directory:**
   - Can't run multiple sessions concurrently in same project
   - Could add worktree support

4. **Tool recursion depth:**
   - Task tool limited to 1 level (subagent can't delegate)
   - Could add configurable depth limit

---

## 22. Key Takeaways

### Architecture Strengths

1. **Clean separation of concerns:**
   - Instance isolation prevents cross-project leaks
   - Bus events enable loose coupling
   - Tool registry provides unified interface

2. **Extensibility:**
   - Plugins for code-level customization
   - MCP for external integrations
   - Custom agents for domain expertise

3. **Developer experience:**
   - SolidJS TUI with 25+ themes
   - Live updates via WebSocket
   - Git-based time-travel

4. **Production-ready:**
   - Permission system prevents accidents
   - External directory protection
   - LSP integration for code quality
   - Comprehensive error handling

### Design Philosophy

- **Execute-first, ask permission later:** Agents autonomous by default
- **Provider-agnostic:** 10 LLM providers, easy to add more
- **Git-native:** Snapshots, VCS tracking, branch awareness
- **Terminal-first:** TUI as primary interface, web as secondary
- **Type-safe:** TypeScript + Zod everywhere
- **Event-driven:** Bus pub/sub for all inter-module communication

---

## Appendix: Command Reference

### CLI Commands (20+)

```bash
opencode                    # Start TUI
opencode agent              # Manage agents
opencode agent generate     # Generate agent from description
opencode auth               # Authenticate with providers
opencode debug              # Show debug info
opencode export             # Export session
opencode generate           # Generate code
opencode github             # GitHub integration
opencode import             # Import session
opencode mcp                # Manage MCP servers
opencode models             # List available models
opencode pr                 # Create pull request
opencode run                # Run agent on prompt
opencode serve              # Start headless server
opencode session            # Manage sessions
opencode stats              # Show usage statistics
opencode upgrade            # Check for updates
opencode web                # Start web interface
```

### Config Locations

- **Global:** `~/.config/opencode/`
- **Project:** `.opencode/` (gitignored)
- **Data:** `~/.local/share/opencode/`

### Environment Variables

- `OPENCODE=1` - Set by CLI
- `AGENT=1` - Set by CLI
- `OPENCODE_SERVER` - Remote server URL
- `OPENCODE_LOG_LEVEL` - Debug logging

---

## Conclusion

OpenCode is a **well-architected AI coding agent** with:

- **Solid foundations:** Instance isolation, event-driven, git-native
- **Production security:** Permission system, external dir protection, tree-sitter validation
- **Rich extensibility:** Plugins, MCP, custom agents
- **Developer experience:** SolidJS TUI, 25+ themes, live updates
- **Provider flexibility:** 10 LLM providers, easy to add more

The codebase follows **consistent patterns** (fn, Instance.state, Bus.event) and prioritizes **type safety** (TypeScript + Zod). The **git-based snapshot system** is a unique innovation enabling granular undo without polluting user's history.

**Ready for:** Production use, plugin development, MCP integration, custom agent creation.

**Best suited for:** Terminal-first developers, teams needing AI-assisted coding, enterprises requiring self-hosted solutions.
