# Phase 2 Implementation Checklist & Getting Started

## ⛔ BLOCKING: Critical Prerequisites (Must Complete First)

**These items were identified during architecture review and MUST be completed before Phase 2 begins.**

> **Estimated Time:** 2-3 days  
> **Status:** NOT STARTED  
> **Owner:** TBD

---

### Prerequisite 1: SSE Runtime Validation ⚠️ CRITICAL

**Why:** The plan assumes `EventSource` works in VSCode's Node18 extension host, but Node.js doesn't have native EventSource. Without this, streaming (Phase 4) will fail.

**Tasks:**

- [ ] Add `eventsource` package: `bun add eventsource @types/eventsource`
- [ ] Create `src/streaming/sse-polyfill.ts`:

  ```typescript
  import EventSource from "eventsource"

  export function createSSEConnection(url: string): EventSource {
    const es = new EventSource(url, {
      // Required for some proxies
      headers: { Accept: "text/event-stream" },
    })
    return es
  }
  ```

- [ ] Write integration test that connects to running `opencode serve`:

  ```typescript
  // test/sse-validation.test.ts
  import { createSSEConnection } from "../src/streaming/sse-polyfill"

  test("SSE connects to opencode server", async () => {
    const es = createSSEConnection("http://localhost:42600/global/event")
    const connected = await new Promise<boolean>((resolve) => {
      es.onopen = () => resolve(true)
      es.onerror = () => resolve(false)
      setTimeout(() => resolve(false), 5000)
    })
    es.close()
    expect(connected).toBe(true)
  })
  ```

- [ ] Document reconnection behavior (auto-reconnect interval, max retries)
- [ ] Test SSE message parsing matches server format

**Acceptance Criteria:**

- SSE connection established from extension host to `opencode serve`
- Messages received and parsed correctly
- Reconnection works after server restart

---

### Prerequisite 2: State Sync Architecture ⚠️ CRITICAL

**Why:** Broadcasting full Zustand state on every mutation will thrash the webview and cause data races. Need clear ownership model.

**Decision Required:** Choose ONE approach:

**Option A: Extension Host Owns State (RECOMMENDED)**

```
Extension Host                    Webview
┌─────────────────┐              ┌─────────────────┐
│ Zustand Store   │──projection──▶│ Read-only copy  │
│ (canonical)     │◀───actions───│ (UI state only) │
└─────────────────┘              └─────────────────┘
```

**Option B: Webview Owns UI State**

```
Extension Host                    Webview
┌─────────────────┐              ┌─────────────────┐
│ Server proxy    │───events────▶│ Zustand Store   │
│ SSE consumer    │◀───requests──│ (canonical)     │
└─────────────────┘              └─────────────────┘
```

**Tasks:**

- [ ] Document chosen approach in DECISIONS_AND_QA.md
- [ ] Define state projection schema (what goes to webview):
  ```typescript
  interface WebviewState {
    connection: { status: "connected" | "disconnected" | "connecting" }
    project: { id: string; path: string } | null
    session: { id: string; title: string } | null
    messages: MessageSummary[] // Not full MessageV2
    streaming: { active: boolean; pendingParts: number }
    ui: { sidebarCollapsed: boolean; theme: "light" | "dark" }
  }
  ```
- [ ] Implement throttled sync (max 10 updates/sec):

  ```typescript
  import { throttle } from "./utils/throttle"

  const syncToWebview = throttle((state: WebviewState) => {
    webviewPanel.webview.postMessage({ type: "state-sync", state })
  }, 100) // 100ms = max 10/sec
  ```

- [ ] Add message deduplication by ID
- [ ] Test with 1000+ messages to validate performance

**Acceptance Criteria:**

- Webview updates smoothly during streaming
- No data races between host and webview
- Performance acceptable with large message lists

---

### Prerequisite 3: Security Hardening ⚠️ HIGH

**Why:** Untrusted content (markdown, tool output) can enable XSS if not properly sandboxed.

**Tasks:**

- [ ] Define CSP policy in `src/webview/index.html`:
  ```html
  <meta
    http-equiv="Content-Security-Policy"
    content="
    default-src 'none';
    style-src ${cspSource} 'unsafe-inline';
    script-src ${cspSource};
    img-src ${cspSource} https: data:;
    font-src ${cspSource};
    connect-src ${cspSource} http://localhost:* ws://localhost:*;
  "
  />
  ```
- [ ] Configure DOMPurify for markdown sanitization:

  ```typescript
  import DOMPurify from "dompurify"

  const ALLOWED_TAGS = ["p", "br", "strong", "em", "code", "pre", "ul", "ol", "li", "a", "h1", "h2", "h3", "blockquote"]
  const ALLOWED_ATTR = ["href", "class", "data-language"]

  export function sanitizeHtml(dirty: string): string {
    return DOMPurify.sanitize(dirty, { ALLOWED_TAGS, ALLOWED_ATTR })
  }
  ```

- [ ] Audit Shiki output for inline scripts (should be safe, but verify)
- [ ] Add `sandbox` attribute to any iframes (if used)
- [ ] Test XSS payloads in markdown don't execute

**Acceptance Criteria:**

- CSP blocks unauthorized script execution
- Markdown XSS payloads are sanitized
- No console errors for legitimate content

---

### Prerequisite 4: Message Part Coverage ⚠️ MEDIUM

**Why:** Only ~6 of 16 MessageV2 part types have renderer specs. Missing parts will break UI.

**Tasks:**

- [ ] Add renderer specs to DECISIONS_AND_QA.md for each missing type:

| Part Type     | Renderer Component | Behavior                                  |
| ------------- | ------------------ | ----------------------------------------- |
| `subtask`     | `<SubtaskPart>`    | Nested collapsible with child parts       |
| `retry`       | `<RetryPart>`      | Warning banner with retry count           |
| `step-start`  | `<StepStartPart>`  | Step header with name, timing             |
| `step-finish` | `<StepFinishPart>` | Step completion with duration             |
| `compaction`  | `<CompactionPart>` | Info notice about compacted messages      |
| `patch`       | `<PatchPart>`      | Unified diff viewer with +/- highlighting |
| `summary`     | `<SummaryPart>`    | Collapsible summary text                  |

- [ ] Create placeholder components that show part type + raw data:
  ```typescript
  export function UnknownPart({ part }: { part: MessagePart }) {
    return (
      <div className="unknown-part">
        <span className="label">Unknown: {part.type}</span>
        <pre>{JSON.stringify(part, null, 2)}</pre>
      </div>
    )
  }
  ```
- [ ] Add part type switch with exhaustive check:
  ```typescript
  function renderPart(part: MessagePart) {
    switch (part.type) {
      case 'text': return <TextPart part={part} />
      case 'tool-call': return <ToolCallPart part={part} />
      // ... all 16 types
      default: {
        const _exhaustive: never = part.type
        return <UnknownPart part={part} />
      }
    }
  }
  ```

**Acceptance Criteria:**

- All 16 part types have at least placeholder rendering
- Unknown types display gracefully (no crashes)
- Spec document covers all types

---

## ✅ Prerequisites Complete Checklist

Before starting Phase 2, verify:

- [ ] SSE polyfill works in extension host context
- [ ] State sync architecture documented and validated
- [ ] CSP policy defined and tested
- [ ] DOMPurify configured for markdown
- [ ] All 16 message part types have renderer specs
- [ ] Placeholder components exist for unimplemented parts

**Sign-off:** ********\_******** Date: ****\_****

---

## Phase 2: Core Infrastructure (3-4 days)

### Pre-Implementation Setup

#### 1. Dependencies to Add

```json
{
  "devDependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "zustand": "^4.4.0",
    "zod": "^3.22.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0"
  }
}
```

**Action items:**

- [ ] Run `bun add react react-dom zustand zod @types/react @types/react-dom`
- [ ] Verify TypeScript types work without errors

#### 2. Project Structure

- [ ] Keep existing `sdks/vscode/` location (don't reorganize yet)
- [ ] Create `src/webview/` directory for React app
- [ ] Create `src/` subdirectories per architecture doc

#### 3. Build Config

- [ ] Update esbuild.js to bundle webview React app
- [ ] Ensure extension host code and webview are separate bundles
- [ ] Test build: `bun run compile`

---

### Task 1: Server Connection (1 day)

**Owner:** 1 dev

#### 1.1 Create Server Discovery Module

**File:** `src/server/discovery.ts`

```typescript
export async function discoverServer(): Promise<string> {
  // 1. Check workspace config
  const config = vscode.workspace.getConfiguration("opencode")
  const configUrl = config.get<string>("serverUrl")
  if (configUrl && (await healthCheck(configUrl))) return configUrl

  // 2. Try defaults
  for (const port of [42600, 42601, 42602]) {
    const url = `http://localhost:${port}`
    if (await healthCheck(url)) return url
  }

  // 3. Ask user
  const input = await vscode.window.showInputBox({
    prompt: "OpenCode server URL",
    value: "http://localhost:42600",
    validateInput: (v) => (!v ? "URL required" : undefined),
  })
  return input!
}

async function healthCheck(url: string): Promise<boolean> {
  const timeout = (ms: number) => new Promise((_, r) => setTimeout(() => r("timeout"), ms))
  try {
    const race = Promise.race([fetch(`${url}/project`, { method: "GET" }), timeout(2000)])
    const res = (await race) as Response
    return res && res.ok
  } catch {
    return false
  }
}
```

**Checklist:**

- [ ] Implement healthCheck() with timeout
- [ ] Test manual URL input
- [ ] Test auto-discovery on localhost ports
- [ ] Verify error handling (show user-friendly messages)

#### 1.2 Create Connection Manager

**File:** `src/server/connection.ts`

```typescript
export class ConnectionManager {
  private url: string = ""
  private retries = 0
  private maxRetries = 5

  async connect(): Promise<void> {
    this.url = await discoverServer()
    await this.healthCheck() // Verify before returning
  }

  async healthCheck(): Promise<void> {
    try {
      const res = await fetch(`${this.url}/project`)
      if (!res.ok) throw new Error(`Health check failed: ${res.status}`)
    } catch (e) {
      throw new Error(`Cannot reach OpenCode server at ${this.url}`)
    }
  }

  async reconnect(): Promise<void> {
    const delay = Math.min(2 ** this.retries * 100, 5000)
    await new Promise((r) => setTimeout(r, delay))
    this.retries++

    if (this.retries > this.maxRetries) {
      throw new Error("Max reconnection attempts reached")
    }

    try {
      await this.healthCheck()
      this.retries = 0
    } catch (e) {
      return this.reconnect()
    }
  }

  getUrl(): string {
    return this.url
  }
}
```

**Checklist:**

- [ ] Implement reconnection logic with backoff
- [ ] Test max retries behavior
- [ ] Verify URL is stored for later use

#### 1.3 Create Status Bar Indicator

**File:** `src/extension.ts` (update)

```typescript
export function activate(context: vscode.ExtensionContext) {
  const statusBar = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Right, 100)
  statusBar.command = "opencode.showConnectionStatus"
  statusBar.text = "$(sync~spin) OpenCode: Connecting..."
  statusBar.show()

  const connectionManager = new ConnectionManager()

  connectionManager
    .connect()
    .then(() => {
      statusBar.text = "$(circle-filled~green) OpenCode: Connected"
      useStore.getState().setConnectionStatus("connected")
    })
    .catch((e) => {
      statusBar.text = "$(circle-outline~red) OpenCode: Disconnected"
      vscode.window.showErrorMessage(`OpenCode: ${e.message}`)
      useStore.getState().setConnectionStatus("disconnected")
    })

  // Show connection info on click
  vscode.commands.registerCommand("opencode.showConnectionStatus", () => {
    vscode.window.showInformationMessage(`OpenCode: ${useStore.getState().connection.status}`)
  })
}
```

**Checklist:**

- [ ] Status bar shows correct state
- [ ] Color indicators work (green/red)
- [ ] Click opens connection info dialog
- [ ] Reconnect attempts happen automatically

---

### Task 2: SDK Client Wrapper (0.5 day)

**Owner:** 1 dev

#### 2.1 Create SDK Client Wrapper

**File:** `src/sdk/client.ts`

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

export interface ClientConfig {
  serverUrl: string
  timeout?: number
}

export class OpencodeClient {
  private client: ReturnType<typeof createOpencodeClient>
  private config: ClientConfig

  constructor(config: ClientConfig) {
    this.config = config
    this.client = createOpencodeClient({
      baseURL: config.serverUrl,
      httpClient: {
        async request(config) {
          const controller = new AbortController()
          const timeout = setTimeout(() => controller.abort(), config.timeout || 30000)
          try {
            const response = await fetch(config.url, {
              method: config.method,
              headers: config.headers,
              body: config.body,
              signal: controller.signal,
            })
            return response
          } finally {
            clearTimeout(timeout)
          }
        },
      },
    })
  }

  // Expose SDK methods with error normalization
  async getProjects() {
    try {
      return await this.client.project.list()
    } catch (e) {
      throw this.normalizeError(e)
    }
  }

  async getSession(projectId: string, sessionId: string) {
    return await this.client.project.session.retrieve({
      project_id: projectId,
      session_id: sessionId,
    })
  }

  async sendMessage(projectId: string, sessionId: string, message: string) {
    return await this.client.project.session.message.create({
      project_id: projectId,
      session_id: sessionId,
      text: message,
    })
  }

  private normalizeError(e: any) {
    if (e instanceof Error) {
      return {
        message: e.message,
        code: "UNKNOWN",
        status: 500,
      }
    }
    return e
  }
}

let globalClient: OpencodeClient | null = null

export function getClient(): OpencodeClient {
  if (!globalClient) throw new Error("Client not initialized")
  return globalClient
}

export function initClient(config: ClientConfig) {
  globalClient = new OpencodeClient(config)
}
```

**Checklist:**

- [ ] Client initializes correctly
- [ ] Error handling works
- [ ] All SDK methods are wrapped and testable
- [ ] Timeout handling works

---

### Task 3: State Management Setup (1 day)

**Owner:** 1 dev

#### 3.1 Create Main Zustand Store

**File:** `src/state/store.ts`

```typescript
import { create } from "zustand"
import { subscribeWithSelector } from "zustand/middleware"

export interface AppState {
  // Connection state
  connection: {
    url: string
    status: "disconnected" | "connecting" | "connected" | "error"
    error?: string
  }

  // Project state
  projects: any[]
  currentProjectId?: string

  // Session state
  currentSessionId?: string
  sessions: any[]
  messages: Record<string, any>

  // UI state
  ui: {
    sidebarCollapsed: boolean
    theme: "light" | "dark"
    selectedMessage?: string
  }

  // Permissions
  pendingPermissions: any[]

  // Actions
  setConnectionUrl: (url: string) => void
  setConnectionStatus: (status: AppState["connection"]["status"]) => void
  setProjects: (projects: any[]) => void
  setCurrentProject: (id: string) => void
  setCurrentSession: (id: string) => void
  setSessions: (sessions: any[]) => void
  addMessage: (msg: any) => void
  addMessagePart: (sessionId: string, msgId: string, part: any) => void
  setSidebarCollapsed: (collapsed: boolean) => void
}

export const useStore = create<AppState>()(
  subscribeWithSelector((set) => ({
    connection: { url: "http://localhost:42600", status: "disconnected" },
    projects: [],
    sessions: [],
    messages: {},
    ui: { sidebarCollapsed: false, theme: "dark" },
    pendingPermissions: [],

    // Connection actions
    setConnectionUrl: (url) =>
      set((state) => ({
        connection: { ...state.connection, url },
      })),
    setConnectionStatus: (status) =>
      set((state) => ({
        connection: { ...state.connection, status },
      })),

    // Project actions
    setProjects: (projects) => set({ projects }),
    setCurrentProject: (id) => set({ currentProjectId: id }),

    // Session actions
    setCurrentSession: (id) => set({ currentSessionId: id }),
    setSessions: (sessions) => set({ sessions }),

    // Message actions
    addMessage: (msg) =>
      set((state) => ({
        messages: { ...state.messages, [msg.id]: msg },
      })),
    addMessagePart: (sessionId, msgId, part) =>
      set((state) => ({
        messages: {
          ...state.messages,
          [msgId]: {
            ...state.messages[msgId],
            parts: [...(state.messages[msgId]?.parts || []), part],
          },
        },
      })),

    // UI actions
    setSidebarCollapsed: (collapsed) =>
      set((state) => ({
        ui: { ...state.ui, sidebarCollapsed: collapsed },
      })),
  })),
)
```

**Checklist:**

- [ ] Store initializes without errors
- [ ] All getters/setters work
- [ ] Persist connection URL to VS Code storage
- [ ] Restore state on extension reload

#### 3.2 Add Persistence Middleware

**File:** `src/state/persistence.ts`

```typescript
export function setupPersistence(context: vscode.ExtensionContext) {
  // Save connection URL
  useStore.subscribe(
    (state) => state.connection.url,
    (url) => {
      context.globalState.update("opencode.serverUrl", url)
    },
  )

  // Restore on activate
  const savedUrl = context.globalState.get<string>("opencode.serverUrl")
  if (savedUrl) {
    useStore.setState((state) => ({
      connection: { ...state.connection, url: savedUrl },
    }))
  }
}
```

**Checklist:**

- [ ] URL persists across reload
- [ ] Other state is reset appropriately
- [ ] No storage errors

---

### Task 4: Webview IPC Bridge (0.5 day)

**Owner:** 1 dev

#### 4.1 Create Message Protocol

**File:** `src/messaging/protocol.ts`

```typescript
import z from "zod"

export const WebviewMessage = z.discriminatedUnion("type", [
  z.object({ type: z.literal("store-update"), payload: z.record(z.any()) }),
  z.object({ type: z.literal("send-message"), text: z.string() }),
  z.object({ type: z.literal("state-request") }),
])

export const HostMessage = z.discriminatedUnion("type", [
  z.object({ type: z.literal("state-sync"), state: z.record(z.any()) }),
  z.object({ type: z.literal("message-sent"), messageId: z.string() }),
  z.object({ type: z.literal("error"), error: z.string() }),
])

export type WebviewMessage = z.infer<typeof WebviewMessage>
export type HostMessage = z.infer<typeof HostMessage>
```

**Checklist:**

- [ ] Types are properly validated with Zod
- [ ] Discriminated union works
- [ ] All messages are documented

#### 4.2 Create IPC Bridge

**File:** `src/messaging/bridge.ts`

```typescript
export class MessageBridge {
  constructor(private webview: vscode.Webview) {}

  sendToWebview(msg: HostMessage) {
    this.webview.postMessage(msg)
  }

  onWebviewMessage(handler: (msg: WebviewMessage) => void) {
    return this.webview.onDidReceiveMessage(async (msg) => {
      try {
        WebviewMessage.parse(msg)
        handler(msg)
      } catch (e) {
        console.error("Invalid webview message", e)
      }
    })
  }

  syncState() {
    const state = useStore.getState()
    this.sendToWebview({
      type: "state-sync",
      state: {
        connection: state.connection,
        projects: state.projects,
        currentProjectId: state.currentProjectId,
        sessions: state.sessions,
        currentSessionId: state.currentSessionId,
      },
    })
  }
}
```

**Checklist:**

- [ ] Messages validate correctly
- [ ] State syncs to webview
- [ ] Error handling doesn't crash

---

### Task 5: React Webview Scaffold (1 day)

**Owner:** 1-2 devs

#### 5.1 Create Webview Entry Point

**File:** `src/webview/index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>OpenCode</title>
    <script>
      const vscode = acquireVsCodeApi()
    </script>
  </head>
  <body>
    <div id="root"></div>
    <script src="./index.tsx"></script>
  </body>
</html>
```

**Checklist:**

- [ ] HTML loads without CSP errors
- [ ] VS Code API is available (vscode global)

#### 5.2 Create React App Root

**File:** `src/webview/App.tsx`

```typescript
import React, { useEffect } from 'react'
import { useStore } from '../state/store'
import Sidebar from './components/sidebar/Sidebar'
import ChatView from './components/chat/ChatView'

export function App() {
  const connection = useStore((s) => s.connection)
  const currentSessionId = useStore((s) => s.currentSessionId)

  useEffect(() => {
    // Request initial state from host
    window.vscode.postMessage({ type: 'state-request' })
  }, [])

  if (connection.status === 'disconnected') {
    return (
      <div className="flex items-center justify-center h-screen">
        <p>Waiting for OpenCode server...</p>
      </div>
    )
  }

  return (
    <div className="flex h-screen">
      <Sidebar />
      {currentSessionId ? <ChatView /> : <EmptyState />}
    </div>
  )
}

function EmptyState() {
  return (
    <div className="flex-1 flex items-center justify-center">
      <p>Select a session to start</p>
    </div>
  )
}
```

**Checklist:**

- [ ] App renders without errors
- [ ] TypeScript compilation works
- [ ] Basic layout appears

#### 5.3 Create Webview Entry

**File:** `src/webview/index.tsx`

```typescript
import React from 'react'
import ReactDOM from 'react-dom/client'
import { App } from './App'
import './styles/globals.css'

const root = ReactDOM.createRoot(document.getElementById('root')!)
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)

// Listen for messages from host
window.addEventListener('message', (event) => {
  const { type, state } = event.data
  if (type === 'state-sync') {
    // Will implement proper state sync later
    console.log('Received state:', state)
  }
})
```

**Checklist:**

- [ ] React renders
- [ ] Message handler logs messages
- [ ] CSS loads

#### 5.4 Create Basic Styling

**File:** `src/webview/styles/globals.css`

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: var(--vscode-editor-background);
  color: var(--vscode-editor-foreground);
  font-size: 13px;
  line-height: 1.5;
}

.flex {
  display: flex;
}
.flex-1 {
  flex: 1;
}
.h-screen {
  height: 100vh;
}
.items-center {
  align-items: center;
}
.justify-center {
  justify-content: center;
}
```

**Checklist:**

- [ ] VS Code theme colors apply
- [ ] Layout classes work

---

### Task 6: Update Extension Entry Point (1 day)

**Owner:** 1 dev (can be done in parallel)

#### 6.1 Update `src/extension.ts`

```typescript
import * as vscode from "vscode"
import { ConnectionManager } from "./server/connection"
import { initClient } from "./sdk/client"
import { useStore } from "./state/store"
import { setupPersistence } from "./state/persistence"
import { MessageBridge } from "./messaging/bridge"

let webviewPanel: vscode.WebviewPanel | undefined

export async function activate(context: vscode.ExtensionContext) {
  console.log("OpenCode extension activating...")

  // Setup persistence
  setupPersistence(context)

  // Discover and connect to server
  const statusBar = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Right, 100)
  statusBar.text = "$(sync~spin) OpenCode: Connecting..."
  statusBar.command = "opencode.showStatus"
  statusBar.show()

  const connectionManager = new ConnectionManager()

  try {
    await connectionManager.connect()
    const serverUrl = connectionManager.getUrl()

    useStore.setState((state) => ({
      connection: { ...state.connection, url: serverUrl, status: "connected" },
    }))
    statusBar.text = "$(circle-filled) OpenCode: Connected"

    initClient({ serverUrl })
  } catch (e) {
    statusBar.text = "$(circle-outline) OpenCode: Disconnected"
    vscode.window.showWarningMessage(`OpenCode: ${e}`)
    useStore.setState((state) => ({
      connection: { ...state.connection, status: "error", error: String(e) },
    }))
  }

  // Register commands
  context.subscriptions.push(vscode.commands.registerCommand("opencode.openPanel", openWebviewPanel))

  // Open webview panel on command
  async function openWebviewPanel() {
    if (webviewPanel) {
      webviewPanel.reveal(vscode.ViewColumn.Beside)
      return
    }

    webviewPanel = vscode.window.createWebviewPanel("opencode", "OpenCode", vscode.ViewColumn.Beside, {
      enableScripts: true,
      retainContextWhenHidden: true,
    })

    const bridge = new MessageBridge(webviewPanel.webview)

    // Send HTML
    const html = getWebviewHtml(context, webviewPanel.webview)
    webviewPanel.webview.html = html

    // Sync state when store changes
    const unsubscribe = useStore.subscribe(
      (state) => state, // Subscribe to all state changes
      () => bridge.syncState(),
    )

    // Handle messages from webview
    bridge.onWebviewMessage(async (msg) => {
      if (msg.type === "state-request") {
        bridge.syncState()
      }
    })

    webviewPanel.onDidDispose(() => {
      webviewPanel = undefined
      unsubscribe()
    })
  }

  // Open panel on startup (optional)
  if (vscode.workspace.workspaceFolders) {
    // Defer to avoid blocking activation
    setTimeout(() => openWebviewPanel(), 500)
  }
}

function getWebviewHtml(context: vscode.ExtensionContext, webview: vscode.Webview): string {
  const scriptUri = webview.asWebviewUri(vscode.Uri.joinPath(context.extensionUri, "dist", "webview.js"))
  const styleUri = webview.asWebviewUri(vscode.Uri.joinPath(context.extensionUri, "dist", "webview.css"))

  return `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <link rel="stylesheet" href="${styleUri}">
    </head>
    <body>
      <div id="root"></div>
      <script>
        const vscode = acquireVsCodeApi();
      </script>
      <script src="${scriptUri}"></script>
    </body>
    </html>
  `
}

export function deactivate() {
  webviewPanel?.dispose()
}
```

**Checklist:**

- [ ] Extension activates without errors
- [ ] Status bar shows correct state
- [ ] Webview panel opens
- [ ] State syncs to webview

---

## Summary: Phase 2 Deliverables

By end of Phase 2, you should have:

- [x] Server discovery working (manual + auto-fallback)
- [x] Connection management with reconnect logic
- [x] Status bar indicator
- [x] SDK client wrapper initialized
- [x] Zustand store with basic state
- [x] State persistence to VS Code storage
- [x] Message bridge (host ↔ webview IPC)
- [x] React app scaffolded and rendering
- [x] Extension entry point configured

**Testing checklist:**

- [ ] `bun run compile` succeeds
- [ ] Extension loads in VS Code
- [ ] Server auto-discovery works
- [ ] Status bar shows connection state
- [ ] Webview panel opens
- [ ] No console errors
- [ ] State persists across reload

---

## Next Phase (Phase 3)

Once Phase 2 is complete:

1. Create Sidebar components (ProjectList, SessionList)
2. Create ChatView with MessageList
3. Build Part renderers (TextPart, ToolPart, FilePart)
4. Create input composer
5. Implement message sending (no streaming yet)

See MASTER_PLAN.md Phase 3 for details.
