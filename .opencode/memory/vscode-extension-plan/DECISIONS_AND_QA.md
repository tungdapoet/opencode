# Questions & Technical Decisions

## Q1: How to handle server discovery/connection?

### Answer: Three-tier fallback strategy

**Tier 1: VS Code Configuration**
```json
{
  "opencode.serverUrl": "http://localhost:42600"
}
```
Users can set this in their workspace settings.

**Tier 2: Auto-discovery**
- Try `localhost:42600` (port 0 returns a fixed port)
- Or check for `OPENCODE_PORT` environment variable
- Perform health check with GET to `/project` endpoint

**Tier 3: Manual Input**
- If auto-discovery fails, show "Connect to Server" dialog
- User pastes URL or port number
- Validate connection before saving

**Implementation in `src/server/discovery.ts`:**
```typescript
async function discoverServer(): Promise<string> {
  // 1. Check config
  const configUrl = vscode.workspace.getConfiguration('opencode').get('serverUrl')
  if (configUrl && await healthCheck(configUrl)) return configUrl
  
  // 2. Try localhost defaults
  for (const port of [42600, 42601, 42602]) {
    const url = `http://localhost:${port}`
    if (await healthCheck(url)) return url
  }
  
  // 3. Ask user
  const url = await vscode.window.showInputBox({
    prompt: 'OpenCode server URL',
    value: 'http://localhost:42600',
  })
  return url!
}

async function healthCheck(url: string): Promise<boolean> {
  try {
    const res = await fetch(`${url}/project`, { timeout: 2000 })
    return res.ok
  } catch {
    return false
  }
}
```

**User Experience:**
1. On first activation, extension auto-discovers
2. If found, connects silently
3. If not found, shows "OpenCode not detected" → user can configure
4. Status bar shows: 🟢 Connected / 🔴 Disconnected / 🟡 Connecting

---

## Q2: WebSocket vs SSE for streaming?

### Answer: **SSE (Server-Sent Events)**

**Decision Criteria:**

| Aspect | SSE | WebSocket |
|--------|-----|-----------|
| **Overhead** | HTTP headers only | Full handshake + frame headers |
| **Reconnection** | Automatic (HTTP) | Manual implementation needed |
| **Proxy support** | Better (standard HTTP) | Often blocked in enterprise |
| **Code complexity** | Simpler (Event API) | More complex (binary frames) |
| **Existing use** | ✓ Already in `/global/event` | ✗ Not used in codebase |
| **One-way stream** | Perfect fit | Overkill |
| **Network efficiency** | Good for text events | Better for binary data |

**OpenCode already uses SSE for `/global/event`, so we maintain consistency.**

### SSE Implementation

**Server side (already exists):**
```typescript
// In server.ts
streamSSE(c, async (stream) => {
  async function handler(event: any) {
    await stream.writeSSE({ data: JSON.stringify(event) })
  }
  Bus.subscribe('*', handler)
  // ... cleanup on disconnect
})
```

**Client side (extension host):**
```typescript
// src/streaming/sse-consumer.ts
export function subscribeToMessageStream(
  sessionId: string,
  onPart: (part: MessageV2.Part) => void,
  onError: (error: Error) => void,
) {
  const eventSource = new EventSource(
    `/project/${projectId}/session/${sessionId}/message/stream`
  )
  
  eventSource.addEventListener('part-updated', (e) => {
    const part = JSON.parse(e.data) as MessageV2.Part
    onPart(part)
  })
  
  eventSource.addEventListener('error', (e) => {
    onError(new Error(`SSE error: ${e}`))
    eventSource.close()
  })
  
  return () => eventSource.close()
}
```

**Webview integration (via Zustand store):**
```typescript
// State mutation when parts arrive
store.getState().addMessagePart(sessionId, messageId, part)
// Triggers re-render of MessageItem
```

---

## Q3: State management approach for VS Code extension?

### Answer: **Zustand with split stores**

**Why Zustand?**
- Lightweight (~2KB)
- No boilerplate (vs Redux)
- Hooks-based (familiar to React devs)
- Middleware support for persistence
- Works in both extension host + webview

**Store Architecture:**

```typescript
// src/state/store.ts
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'

interface AppState {
  // Connection
  connection: {
    url: string
    status: 'disconnected' | 'connecting' | 'connected'
    error?: string
  }
  setConnectionUrl: (url: string) => void
  setConnectionStatus: (status: AppState['connection']['status']) => void
  
  // Project
  projects: Project[]
  currentProjectId?: string
  setProjects: (projects: Project[]) => void
  setCurrentProject: (id: string) => void
  
  // Session
  sessions: Session[]
  currentSessionId?: string
  setCurrentSession: (id: string) => void
  setSessions: (sessions: Session[]) => void
  
  // Messages
  messages: Record<string, MessageV2.Assistant | MessageV2.User>
  addMessage: (msg: MessageV2.User) => void
  addMessagePart: (sessionId: string, msgId: string, part: MessageV2.Part) => void
  updateMessagePart: (sessionId: string, msgId: string, partId: string, part: MessageV2.Part) => void
  
  // UI
  ui: {
    sidebarCollapsed: boolean
    selectedMessage?: string
    theme: 'light' | 'dark'
  }
  setSidebarCollapsed: (collapsed: boolean) => void
  setSelectedMessage: (id?: string) => void
  
  // Permissions
  pendingPermissions: PermissionRequest[]
  addPermissionRequest: (req: PermissionRequest) => void
  removePermissionRequest: (id: string) => void
}

export const useStore = create<AppState>()(
  subscribeWithSelector((set) => ({
    connection: { url: 'http://localhost:42600', status: 'disconnected' },
    setConnectionUrl: (url) => set((state) => ({
      connection: { ...state.connection, url }
    })),
    // ... all other reducers
  }))
)
```

**Persistence:**
```typescript
// In extension.ts (main process)
useStore.subscribe(
  (state) => state.connection.url,
  (url) => {
    context.globalState.update('serverUrl', url)
  }
)

// On activation, restore
const savedUrl = context.globalState.get('serverUrl')
if (savedUrl) useStore.setState({ connection: { ...useStore.getState().connection, url: savedUrl } })
```

**Extension Host ↔ Webview Sync:**
```typescript
// In extension.ts
const unsubscribe = useStore.subscribe(
  (state) => state.messages,
  (messages) => {
    // Send to webview
    webviewPanel.webview.postMessage({
      type: 'state-update',
      payload: { messages }
    })
  }
)

// In webview (src/App.tsx)
window.addEventListener('message', (event) => {
  if (event.data.type === 'state-update') {
    useStore.setState(event.data.payload)
  }
})
```

**Benefits:**
- Single source of truth
- Easy to debug (Redux DevTools compatible with Zustand)
- Minimal re-renders (subscribeWithSelector)
- Shared logic between host and webview

---

## Q4: How to render markdown/code in webview efficiently?

### Answer: **Shiki + React components**

**Choice: Markdown-it for parsing, Shiki for code blocks**

Why not MDX/Remark/etc?
- **Markdown-it** is battle-tested, fast, minimal setup
- **Shiki** has theme support matching VS Code colors
- Lightweight bundle (~100KB total)
- One-pass rendering (no AST traversal overhead)

**Implementation:**

```typescript
// src/utils/markdown.ts
import MarkdownIt from 'markdown-it'
import { getHighlighter, type Highlighter } from 'shiki'

let highlighter: Highlighter

export async function initMarkdown() {
  highlighter = await getHighlighter({
    themes: ['github-light', 'github-dark'],
    langs: ['javascript', 'typescript', 'python', 'bash', 'json', 'html', 'css'],
  })
}

const md = new MarkdownIt({
  highlight: (code, lang) => {
    if (!lang) return code
    try {
      return highlighter.codeToHtml(code, {
        lang,
        theme: isDarkTheme() ? 'github-dark' : 'github-light',
      })
    } catch {
      return code
    }
  }
})

// Sanitization
import DOMPurify from 'dompurify'
export function renderMarkdown(text: string): string {
  const html = md.render(text)
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: [
      'p', 'br', 'strong', 'em', 'u', 'code', 'pre', 'h1', 'h2', 'h3',
      'ul', 'ol', 'li', 'blockquote', 'table', 'thead', 'tbody', 'tr', 'td', 'th'
    ],
    ALLOWED_ATTR: ['class', 'style'],
  })
}
```

**React Component:**

```typescript
// src/webview/components/parts/TextPartRenderer.tsx
import { useMemo } from 'react'
import { renderMarkdown } from '../../utils/markdown'

export function TextPartRenderer({ text }: { text: string }) {
  const html = useMemo(() => renderMarkdown(text), [text])
  
  return (
    <div
      className="prose prose-dark max-w-none"
      dangerouslySetInnerHTML={{ __html: html }}
    />
  )
}
```

**CSS for code blocks (Tailwind + custom):**
```css
/* src/styles/markdown.css */
.prose code {
  background: var(--vscode-editor-background);
  color: var(--vscode-editor-foreground);
  padding: 2px 4px;
  border-radius: 3px;
}

.prose pre {
  background: var(--vscode-editor-background);
  border: 1px solid var(--vscode-editorGroupHeader-border);
  padding: 1rem;
  overflow-x: auto;
}

.prose pre code {
  background: none;
  color: inherit;
  padding: 0;
}
```

**Performance optimization:**
- Memoize rendered HTML (renderMarkdown is pure)
- Lazy-load Shiki highlighter on first use
- Cache highlighted themes
- Virtualize for large message lists (react-window)

---

## Q5: Permission dialog UX in VS Code context?

### Answer: **Modal dialog + status notification**

**UX Flow:**

```
1. Tool execution pending permission
   ↓
2. Toast notification appears:
   "🔒 Requires permission: bash read file.txt"
   [View] button
   ↓
3. Click [View] → Modal opens:
   
   ┌─────────────────────────────────────┐
   │ Permission Required                  │
   │─────────────────────────────────────│
   │                                      │
   │ Tool: bash                          │
   │ Action: execute                      │
   │                                      │
   │ The agent wants to run:             │
   │ $ cat /Users/me/file.txt            │
   │                                      │
   │ Description:                         │
   │ Reading project configuration       │
   │                                      │
   │ Context:                             │
   │ Current task: Analyze codebase      │
   │                                      │
   │ ┌────────────┐ ┌───────┐ ┌──────────┐│
   │ │   Deny     │ │ Ask me│ │  Allow   ││
   │ │            │ │ again │ │ (always) ││
   │ └────────────┘ └───────┘ └──────────┘│
   │                                      │
   │ ☐ Remember this choice             │
   └─────────────────────────────────────┘
   ↓
4. User action:
   - Deny: Tool fails, message shows error
   - Ask me again: Single permission only
   - Allow (always): Update permission settings for this tool
   ↓
5. Session resumes, tool executes
```

**Implementation:**

```typescript
// src/webview/components/modals/PermissionDialog.tsx
export function PermissionDialog({
  request,
  onAllow,
  onDeny,
  onAskAgain,
}: {
  request: PermissionRequest
  onAllow: () => void
  onDeny: () => void
  onAskAgain: () => void
}) {
  const [remember, setRemember] = useState(false)
  
  return (
    <Modal isOpen={!!request}>
      <ModalHeader>
        <LockIcon /> Permission Required
      </ModalHeader>
      
      <ModalBody>
        <ToolBadge tool={request.tool} action={request.action} />
        
        <CodeBlock>
          {request.rawCommand || `${request.tool} ${request.action}`}
        </CodeBlock>
        
        {request.description && (
          <Section label="Description">{request.description}</Section>
        )}
        
        {request.context && (
          <Section label="Context">{request.context}</Section>
        )}
      </ModalBody>
      
      <ModalFooter>
        <button onClick={onDeny}>Deny</button>
        <button onClick={onAskAgain}>Ask me again</button>
        <button
          onClick={() => {
            onAllow()
            if (remember) savePermissionPreference(request.tool, 'allow')
          }}
          variant="primary"
        >
          Allow {remember && '(always)'}
        </button>
        <label>
          <input
            type="checkbox"
            checked={remember}
            onChange={(e) => setRemember(e.target.checked)}
          />
          Remember this choice
        </label>
      </ModalFooter>
    </Modal>
  )
}
```

**State management:**

```typescript
// In store
interface PermissionRequest {
  id: string
  tool: string
  action: string
  rawCommand?: string
  description?: string
  context?: string
  time: number
}

// Mutations
addPermissionRequest: (req) => set((state) => ({
  pendingPermissions: [...state.pendingPermissions, req]
}))

respondToPermission: (id, decision) => {
  // POST to server
  api.permission(id, { decision })
  // Remove from pending
  set((state) => ({
    pendingPermissions: state.pendingPermissions.filter(p => p.id !== id)
  }))
}
```

**Server integration (extension host):**

```typescript
// src/streaming/event-subscription.ts
bus.subscribe('permission.requested', (event) => {
  useStore.getState().addPermissionRequest(event.data)
})

// When user responds
useStore.subscribe(
  (state) => state.pendingPermissions,
  async (permissions) => {
    // POST response to server
    for (const perm of permissions.filter(p => p.decided)) {
      await client.permission(perm.id, { decision: perm.decision })
    }
  }
)
```

---

## Q6: How to integrate with VS Code's native file system?

### Answer: **Bridge between session and workspace**

**Use Cases:**
1. User mentions file in chat → Open in editor
2. Session modifies files → Sync to user's editor
3. User opens file → Auto-add to session context
4. Diffs are displayed → User can apply to workspace

**Implementation:**

```typescript
// src/utils/file-integration.ts
export async function openFileInEditor(filePath: string, range?: LSP.Range) {
  const uri = vscode.Uri.file(filePath)
  const doc = await vscode.workspace.openTextDocument(uri)
  const editor = await vscode.window.showTextDocument(doc)
  
  if (range) {
    const sel = new vscode.Selection(
      range.start.line,
      range.start.character,
      range.end.line,
      range.end.character
    )
    editor.selection = sel
    editor.revealRange(sel, vscode.TextEditorRevealType.InCenter)
  }
}

export async function getCurrentFileContext(): Promise<string> {
  const editor = vscode.window.activeTextEditor
  if (!editor) return ''
  
  const { document, selection } = editor
  const filePath = document.uri.fsPath
  const content = document.getText()
  
  // Include selection if available
  const selectedText = selection.isEmpty
    ? ''
    : document.getText(selection)
  
  return `File: ${filePath}\n${selectedText || content}`
}
```

**File sync from session:**

```typescript
// When a FilePart arrives with changes
useEffect(() => {
  for (const filePart of messageParts.filter(p => p.type === 'file')) {
    const { path, url } = filePart
    // Fetch file content from session
    fetch(url).then(async (res) => {
      const content = await res.text()
      // Write to workspace
      const uri = vscode.Uri.file(path)
      const edit = new vscode.WorkspaceEdit()
      edit.createFile(uri, { overwrite: true })
      edit.insert(uri, new vscode.Position(0, 0), content)
      await vscode.workspace.applyEdit(edit)
    })
  }
}, [messageParts])
```

**Diff visualization:**

```typescript
// src/webview/components/DiffViewer.tsx
export function DiffViewer({ patchPart }: { patchPart: MessageV2.PatchPart }) {
  return (
    <div className="diff-container">
      {patchPart.files.map((file) => (
        <DiffFile
          key={file}
          path={file}
          oldContent={...}
          newContent={...}
          onApply={() => {
            // Send to extension host to apply patch
            vscode.postMessage({
              type: 'apply-patch',
              file,
              patch: patchPart.hash,
            })
          }}
        />
      ))}
    </div>
  )
}
```

---

## Summary Table: All Decisions

| Question | Decision | Key Rationale |
|----------|----------|---------------|
| Server discovery | 3-tier: config → auto → manual | User control + sensible defaults |
| Streaming | SSE over WebSocket | Already used, simpler, better proxy support |
| State mgmt | Zustand | Lightweight, no boilerplate, persistent |
| Markdown rendering | Markdown-it + Shiki | Fast, theme-aware, single-pass |
| Permissions | Modal + toast notification | Clear UX, explicit control |
| File integration | Open in editor, fetch content, apply patches | Seamless workspace sync |
