# Claude Code Orchestrator

## Design Document

**Version:** 1.0  
**Date:** January 11, 2026  
**Author:** Brendan  
**Status:** Draft

---

## Executive Summary

Claude Code Orchestrator is a self-hosted platform that enables remote orchestration of Claude Code sessions from Claude.ai (or any MCP client), while also providing a web-based management interface for direct human interaction. It builds on the existing `claude-code-container-mcp` project, adding HTTP transport, streaming output, session lifecycle management, and a React-based UI.

The system allows Claude.ai to dispatch coding tasks to containerized Claude Code instances running on a user's local machine, monitor their progress, and continue conversations across multiple prompts—all while giving the user full visibility and control through a web dashboard.

---

## Goals

1. **Remote MCP Access**: Expose the MCP server over HTTP so Claude.ai can connect via custom connector
2. **Streaming Output**: Real-time visibility into Claude Code's work via SSE
3. **Unified Interface**: Both MCP clients and humans interact with the same session pool
4. **Session Persistence**: Sessions remain active between prompts, supporting `--resume` workflows
5. **Self-Service Management**: Web UI for creating API keys, viewing sessions, and interacting directly
6. **Automatic Cleanup**: TTL-based garbage collection of stale containers

---

## Non-Goals (v1)

- Multi-user/multi-tenant support (single user, multiple API keys)
- Git worktree isolation (use container isolation; worktrees can be added later)
- Kubernetes deployment (Docker Compose only)
- `report_progress` callback MCP for Code agent (nice-to-have for v2)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Internet                                        │
│                                                                             │
│    ┌──────────────┐         ┌──────────────────┐                           │
│    │  Claude.ai   │         │  User's Browser  │                           │
│    │  (MCP Client)│         │  (React App)     │                           │
│    └──────┬───────┘         └────────┬─────────┘                           │
│           │                          │                                      │
└───────────┼──────────────────────────┼──────────────────────────────────────┘
            │                          │
            │ HTTPS (MCP-over-HTTP)    │ HTTPS (tRPC + SSE)
            │                          │
┌───────────┼──────────────────────────┼──────────────────────────────────────┐
│           │    Cloudflare Tunnel     │                                      │
│           └──────────┬───────────────┘                                      │
│                      │                                                      │
│    ┌─────────────────▼─────────────────┐                                   │
│    │     Claude Code Orchestrator       │                                   │
│    │     (Next.js Application)          │                                   │
│    │                                    │                                   │
│    │  ┌────────────┐  ┌──────────────┐ │                                   │
│    │  │ MCP Server │  │  tRPC Router │ │                                   │
│    │  │ /api/mcp   │  │  /api/trpc/* │ │                                   │
│    │  └─────┬──────┘  └──────┬───────┘ │                                   │
│    │        │                │         │                                   │
│    │        └────────┬───────┘         │                                   │
│    │                 │                 │                                   │
│    │        ┌────────▼────────┐        │                                   │
│    │        │ SessionManager  │        │                                   │
│    │        └────────┬────────┘        │                                   │
│    │                 │                 │                                   │
│    │        ┌────────▼────────┐        │                                   │
│    │        │  OutputBuffer   │        │  ◄── SSE: /api/sessions/:id/stream│
│    │        │  (per session)  │        │                                   │
│    │        └────────┬────────┘        │                                   │
│    │                 │                 │                                   │
│    └─────────────────┼─────────────────┘                                   │
│                      │                                                      │
│    ┌─────────────────▼─────────────────┐                                   │
│    │           Docker                   │                                   │
│    │  ┌─────────┐ ┌─────────┐          │                                   │
│    │  │Session 1│ │Session 2│  ...     │                                   │
│    │  │Claude   │ │Claude   │          │                                   │
│    │  │Code     │ │Code     │          │                                   │
│    │  └─────────┘ └─────────┘          │                                   │
│    └───────────────────────────────────┘                                   │
│                                                                             │
│                    User's Desktop                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Next.js Application

The main application server, handling all HTTP traffic.

**Routes:**

| Route | Purpose |
|-------|---------|
| `/api/mcp` | MCP-over-HTTP endpoint for Claude.ai connector |
| `/api/trpc/*` | tRPC router for web UI |
| `/api/sessions/:id/stream` | SSE endpoint for live output streaming |
| `/api/auth/*` | NextAuth.js authentication routes |
| `/*` | React web application |

### 2. SessionManager

Core service managing the lifecycle of Claude Code containers. Forked and adapted from `claude-code-container-mcp`.

**Responsibilities:**

- Create/destroy Docker containers
- Execute prompts via `claude -p --output-format stream-json`
- Resume sessions via `claude -p --resume <session_id>`
- Track session metadata (status, Claude session ID, timestamps)
- Coordinate with OutputBuffer for streaming

**Key Changes from Upstream:**

| Upstream Behavior | Our Changes |
|-------------------|-------------|
| Runs `claude -p` and waits for completion | Streams output in real-time via `--output-format stream-json` |
| Returns final result only | Stores full event history in OutputBuffer |
| No session resume support | Tracks Claude session ID, supports `--resume` |
| Stdio MCP transport only | HTTP transport with bearer auth |

### 3. OutputBuffer

Per-session buffer storing parsed `stream-json` events.

**Structure:**

```typescript
interface OutputBuffer {
  sessionId: string;
  events: StreamEvent[];          // Full history
  status: 'idle' | 'running' | 'completed' | 'error';
  claudeSessionId?: string;       // For --resume
  lastActivity: Date;
  subscribers: Set<SSEClient>;    // For live streaming
}

type StreamEvent = 
  | { type: 'init'; session_id: string; timestamp: string }
  | { type: 'message'; role: 'assistant' | 'user'; content: ContentBlock[] }
  | { type: 'tool_use'; name: string; input: unknown }
  | { type: 'tool_result'; output: string }
  | { type: 'result'; status: string; duration_ms: number };
```

**Operations:**

- `append(event)`: Add event, broadcast to SSE subscribers
- `getRecent(maxMessages?, maxTokens?)`: Retrieve recent output with limits
- `subscribe(client)` / `unsubscribe(client)`: SSE management

### 4. CleanupWorker

Background job that garbage-collects stale sessions.

**Logic:**

```typescript
// Run every hour
for (const session of sessions) {
  const idleTime = Date.now() - session.lastActivity;
  if (idleTime > TTL_MS) {  // Default: 48 hours
    await sessionManager.destroy(session.id);
  }
}
```

### 5. Database (SQLite)

Lightweight persistence for session metadata and API keys.

**Schema:**

```sql
-- API Keys for MCP authentication
CREATE TABLE api_keys (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  key_hash TEXT NOT NULL,      -- bcrypt hash
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_used_at DATETIME
);

-- Session metadata (containers are source of truth for running state)
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  container_id TEXT NOT NULL,
  claude_session_id TEXT,       -- For --resume
  project_path TEXT NOT NULL,
  status TEXT NOT NULL,         -- 'running', 'idle', 'destroyed'
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_activity DATETIME
);

-- Output events (optional, for persistence across restarts)
CREATE TABLE session_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL REFERENCES sessions(id),
  event_type TEXT NOT NULL,
  event_data TEXT NOT NULL,     -- JSON
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## API Design

### MCP Tools

Exposed via `/api/mcp` for Claude.ai connector.

| Tool | Description | Parameters |
|------|-------------|------------|
| `create_session` | Create a new Claude Code container | `projectPath`, `sessionName?`, `model?` |
| `execute_prompt` | Run a prompt in a session | `sessionId`, `prompt` |
| `get_output` | Retrieve session output | `sessionId`, `maxMessages?`, `maxTokens?` |
| `list_sessions` | List all active sessions | — |
| `destroy_session` | Stop and remove a session | `sessionId` |

### tRPC Router

For the web UI, mirroring MCP tools plus additional management endpoints.

```typescript
const appRouter = router({
  // Session management (mirrors MCP)
  sessions: router({
    create: procedure.input(CreateSessionInput).mutation(/* ... */),
    execute: procedure.input(ExecuteInput).mutation(/* ... */),
    getOutput: procedure.input(GetOutputInput).query(/* ... */),
    list: procedure.query(/* ... */),
    destroy: procedure.input(DestroyInput).mutation(/* ... */),
  }),
  
  // API key management (web UI only)
  apiKeys: router({
    create: procedure.input(CreateKeyInput).mutation(/* ... */),
    list: procedure.query(/* ... */),
    revoke: procedure.input(RevokeKeyInput).mutation(/* ... */),
  }),
  
  // User info
  auth: router({
    me: procedure.query(/* ... */),
  }),
});
```

### SSE Streaming

`GET /api/sessions/:id/stream`

Returns a Server-Sent Events stream of session output.

```
event: init
data: {"session_id":"abc123","timestamp":"2026-01-11T..."}

event: message
data: {"role":"assistant","content":[{"type":"text","text":"I'll start by..."}]}

event: tool_use
data: {"name":"Bash","input":{"command":"ls -la"}}

event: tool_result
data: {"output":"total 64\ndrwxr-xr-x..."}

event: message
data: {"role":"assistant","content":[{"type":"text","text":"I found..."}]}

event: result
data: {"status":"success","duration_ms":12345}
```

---

## Authentication

### Web UI

- **NextAuth.js** with Google OAuth provider
- Session stored in HTTP-only cookie
- Single authorized user (configured via environment variable)

### MCP Endpoint

- **Bearer token** authentication
- Tokens created via web UI, stored as bcrypt hashes
- Header: `Authorization: Bearer <api_key>`

### Configuration

```env
# Auth
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
AUTHORIZED_EMAIL=brendan@example.com

# MCP
MCP_API_KEYS_ENABLED=true
```

---

## Web UI

### Pages

1. **Dashboard** (`/`)
   - List of active sessions with status indicators
   - Quick actions: view, execute prompt, destroy
   - System stats (container count, resource usage)

2. **Session Detail** (`/sessions/:id`)
   - Live streaming output (terminal-style view)
   - Prompt input field
   - Session metadata (created, last activity, Claude session ID)
   - Destroy button

3. **New Session** (`/sessions/new`)
   - Project path input (with suggestions from recent)
   - Optional session name
   - Model selection

4. **API Keys** (`/settings/api-keys`)
   - Create new key (shows key once, never again)
   - List existing keys (name, created, last used)
   - Revoke keys

### Components

```
src/
├── app/
│   ├── page.tsx                    # Dashboard
│   ├── sessions/
│   │   ├── page.tsx                # Session list
│   │   ├── new/page.tsx            # Create session
│   │   └── [id]/page.tsx           # Session detail
│   └── settings/
│       └── api-keys/page.tsx       # API key management
├── components/
│   ├── SessionList.tsx
│   ├── SessionCard.tsx
│   ├── TerminalOutput.tsx          # Streaming output display
│   ├── PromptInput.tsx
│   └── ApiKeyManager.tsx
└── lib/
    ├── trpc.ts                     # tRPC client
    └── sse.ts                      # SSE hook
```

---

## Deployment

### Local Development

```bash
# Clone and setup
git clone https://github.com/user/claude-code-orchestrator.git
cd claude-code-orchestrator
pnpm install

# Configure
cp .env.example .env.local
# Edit .env.local with credentials

# Run
pnpm dev
```

### Production (Self-Hosted)

```bash
# Build
pnpm build

# Run with PM2 or similar
pm2 start pnpm --name orchestrator -- start

# Expose via Cloudflare Tunnel
cloudflared tunnel --url http://localhost:3000
```

### Docker Compose (Alternative)

```yaml
version: '3.8'
services:
  orchestrator:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
    environment:
      - DATABASE_URL=file:/app/data/db.sqlite
      - GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID}
      - GOOGLE_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET}
      - AUTHORIZED_EMAIL=${AUTHORIZED_EMAIL}
```

---

## Fork Strategy

### Files to Keep from Upstream

- `src/container-manager.ts` — Docker container lifecycle
- `src/tools/` — MCP tool implementations (adapt, don't rewrite)
- `Dockerfile`, `Dockerfile.custom` — Container images
- `scripts/build-custom-image.sh`

### Files to Add

- `src/app/` — Next.js app router pages
- `src/server/` — tRPC router, session manager adaptations
- `src/lib/` — Shared utilities
- `src/components/` — React components
- `prisma/` or `drizzle/` — Database schema (SQLite)

### Key Modifications

1. **Transport Layer**
   - Add HTTP server (Next.js API routes)
   - Implement MCP-over-HTTP protocol
   - Add SSE streaming endpoint

2. **Session Execution**
   - Change from blocking execution to streaming
   - Parse `stream-json` output in real-time
   - Store events in OutputBuffer
   - Track Claude session ID for `--resume`

3. **Authentication**
   - Add NextAuth.js for web UI
   - Add bearer token validation for MCP endpoint

4. **Persistence**
   - Add SQLite database
   - Store API keys, session metadata
   - Optional: persist output events

---

## Implementation Plan

### Phase 1: Core Infrastructure (Week 1)

- [ ] Fork repository, set up Next.js structure
- [ ] Implement HTTP transport for MCP
- [ ] Add bearer token authentication
- [ ] Basic session creation/destruction via MCP

### Phase 2: Streaming (Week 1-2)

- [ ] Modify execution to use `--output-format stream-json`
- [ ] Implement OutputBuffer with event parsing
- [ ] Add SSE endpoint for live streaming
- [ ] Implement `--resume` support

### Phase 3: Web UI (Week 2)

- [ ] NextAuth.js setup with Google OAuth
- [ ] Dashboard and session list pages
- [ ] Session detail with streaming terminal view
- [ ] Prompt input and execution

### Phase 4: Management (Week 3)

- [ ] API key management UI
- [ ] Cleanup worker for TTL-based garbage collection
- [ ] Session persistence across restarts
- [ ] Error handling and recovery

### Phase 5: Polish (Week 3-4)

- [ ] Cloudflare Tunnel documentation
- [ ] Claude.ai custom connector setup guide
- [ ] Testing and bug fixes
- [ ] README and deployment docs

---

## Open Questions

1. **Output persistence**: Should we persist all output events to SQLite, or just keep them in memory? Memory is simpler but loses history on restart.

2. **Multiple project paths**: Should a session be able to access multiple directories, or stick with single `projectPath`?

3. **Model selection**: Expose model choice (`sonnet`, `opus`) per-session, or use a global default?

4. **Resource limits**: Should we enforce CPU/memory limits on containers?

5. **Concurrent execution**: Can a session handle multiple `execute_prompt` calls simultaneously, or queue them?

---

## Security Considerations

1. **Docker socket access**: The orchestrator needs Docker socket access, which is effectively root. Run on a dedicated machine or VM.

2. **API key storage**: Keys are bcrypt-hashed; raw keys shown once at creation.

3. **Cloudflare Tunnel**: Provides HTTPS termination and DDoS protection. Ensure tunnel is configured with access policies if needed.

4. **Container isolation**: Each Claude Code instance is isolated. Consider adding `--security-opt=no-new-privileges` and `--cap-drop=ALL`.

5. **Project path validation**: Validate that `projectPath` is within allowed directories to prevent arbitrary filesystem access.

---

## Success Criteria

1. Claude.ai can create a session, execute multiple prompts with `--resume`, and retrieve output via MCP
2. User can view live streaming output in web UI while Claude.ai is executing
3. Sessions automatically clean up after 48 hours of inactivity
4. API keys can be created/revoked without restarting the server
5. System handles 5+ concurrent sessions without degradation
