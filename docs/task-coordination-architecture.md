# Task Coordination System - Architecture

**Author:** Nix Tanaka  
**Date:** 2026-02-08  
**Status:** Draft

---

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         User Layer                          │
│  (Telegram, CLI, Web UI)                                    │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────────┐
│                    OpenClaw Gateway                         │
│  - Routes messages to agents                                │
│  - Integrates with Task Coordinator                         │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────────┐
│                   Task Coordinator                          │
│  - Task Queue (priority queue)                              │
│  - Agent Registry (who's working on what)                   │
│  - Progress Tracker (heartbeat monitor)                     │
│  - API Server (REST endpoints)                              │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────────┐
│                      Task Storage                           │
│  (SQLite or Postgres)                                       │
│  - tasks table                                              │
│  - progress_updates table                                   │
│  - agent_heartbeats table                                   │
└─────────────────────────────────────────────────────────────┘
                 ↑
                 │
        ┌────────┴────────┐
        │                 │
    ┌───▼───┐         ┌───▼───┐
    │ Agent │         │ Agent │  ... (N agents)
    │  Nix  │         │  Riley│
    └───────┘         └───────┘
```

---

## Components

### 1. Task Coordinator (Core Service)

**Responsibilities:**
- Maintain task queue (priority-sorted)
- Track agent status (active task, last heartbeat)
- Route tasks to agents (assignment or claiming)
- Monitor progress (heartbeat tracking)
- Handle interruptions (priority escalation, cancellation)

**Implementation:**
- Node.js service (runs alongside OpenClaw Gateway)
- REST API (agents and users interact via HTTP)
- In-memory queue + persistent storage (SQLite/Postgres)
- Background job: Check for stale heartbeats every 30s

**Configuration:**
```json5
{
  taskCoordinator: {
    enabled: true,
    storage: "sqlite",           // or "postgres"
    dbPath: "~/.openclaw/tasks.db",
    heartbeatIntervalMs: 30000,  // 30s
    staleThresholdMs: 90000,     // 90s
    maxTasksPerAgent: 3,         // soft limit
    api: {
      port: 18790,               // Gateway on 18789
      auth: "gateway-token"      // Share token with gateway
    }
  }
}
```

### 2. Agent Integration

**How agents interact with Task Coordinator:**

**A. Polling Mode (Simple)**
- Agent polls `/api/tasks/next` every 10-30 seconds
- Receives next task for its queue (if any)
- Reports progress via POST `/api/tasks/:id/progress`

**B. Push Mode (Efficient)**
- Agent connects to WebSocket (`/api/agents/connect`)
- Coordinator pushes tasks to agent when available
- Agent pushes progress updates over same socket

**Recommendation: Start with polling (simpler), add push later if needed.**

**Agent workflow:**
```
1. Agent starts → Register with coordinator
2. Poll for tasks (or wait for push)
3. Claim task → POST /api/tasks/:id/claim
4. Work on task → Report progress every 30-60s
5. Complete → POST /api/tasks/:id/complete
6. Repeat
```

### 3. Gateway Integration

**OpenClaw Gateway modifications:**

**Option A: Plugin (Recommended)**
- Task coordinator runs as OpenClaw plugin
- Loaded at gateway startup
- Shares session context with agents

**Option B: Sidecar Service**
- Task coordinator runs as separate process
- Gateway communicates via HTTP
- More decoupled, easier to test independently

**Recommendation: Plugin for V1 (easier integration), sidecar for open-source later.**

**Gateway hooks:**
```javascript
// When user sends message to agent
gateway.on('message:toAgent', async (msg) => {
  // Check message priority
  const priority = detectPriority(msg.text);  // "urgent!" → URGENT
  
  // Create task
  await taskCoordinator.createTask({
    description: msg.text,
    priority,
    assignee: msg.targetAgent,
    source: msg.userId
  });
});

// When agent sends progress update
agent.on('progress', async (update) => {
  await taskCoordinator.reportProgress({
    taskId: update.taskId,
    status: update.status,
    progress: update.progress,
    eta: update.eta
  });
});
```

### 4. Storage Layer

**Database: SQLite or Postgres**

**SQLite for MVP:**
- Simple setup (single file)
- Good enough for 5 agents, 100 tasks/hour
- Easy to bundle with OpenClaw

**Postgres for scale:**
- Better concurrency (if open-sourced)
- Robust querying (complex task filters)
- Industry standard

**Recommendation: SQLite for V1, Postgres migration path documented.**

---

## Data Model

### Tasks Table

```sql
CREATE TABLE tasks (
  id TEXT PRIMARY KEY,              -- UUID
  description TEXT NOT NULL,
  priority TEXT NOT NULL,           -- URGENT, HIGH, NORMAL, LOW
  status TEXT NOT NULL,             -- PENDING, IN_PROGRESS, BLOCKED, COMPLETE, CANCELLED
  
  -- Assignment
  assignee TEXT,                    -- Agent ID (nullable = unclaimed)
  claimed_at TIMESTAMP,
  claimed_by TEXT,                  -- Agent ID who claimed it
  
  -- Hierarchy
  parent_task_id TEXT,              -- For subtasks
  
  -- Timestamps
  created_at TIMESTAMP NOT NULL,
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  cancelled_at TIMESTAMP,
  
  -- Progress
  progress_pct INTEGER DEFAULT 0,   -- 0-100
  current_step TEXT,                -- "Writing code for X"
  eta_seconds INTEGER,              -- Estimated time remaining
  
  -- Metadata
  source_user TEXT,                 -- User who created task
  result TEXT,                      -- Summary of result (when complete)
  blocking_reason TEXT,             -- Why task is blocked (if BLOCKED)
  
  FOREIGN KEY (parent_task_id) REFERENCES tasks(id)
);

CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_priority ON tasks(priority);
CREATE INDEX idx_tasks_assignee ON tasks(assignee);
CREATE INDEX idx_tasks_parent ON tasks(parent_task_id);
```

### Progress Updates Table

```sql
CREATE TABLE progress_updates (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  task_id TEXT NOT NULL,
  agent_id TEXT NOT NULL,
  
  status TEXT NOT NULL,             -- IN_PROGRESS, BLOCKED, COMPLETE
  progress_pct INTEGER,
  current_step TEXT,
  eta_seconds INTEGER,
  
  created_at TIMESTAMP NOT NULL,
  
  FOREIGN KEY (task_id) REFERENCES tasks(id)
);

CREATE INDEX idx_progress_task ON progress_updates(task_id);
```

### Agent Heartbeats Table

```sql
CREATE TABLE agent_heartbeats (
  agent_id TEXT PRIMARY KEY,
  current_task_id TEXT,             -- What agent is working on (nullable)
  status TEXT NOT NULL,             -- IDLE, WORKING, STUCK
  
  last_heartbeat TIMESTAMP NOT NULL,
  last_progress_update TIMESTAMP,
  
  -- Metadata
  session_key TEXT,
  model TEXT,
  
  FOREIGN KEY (current_task_id) REFERENCES tasks(id)
);
```

---

## API Specification

**Base URL:** `http://localhost:18790/api`

**Authentication:** Gateway token in `Authorization: Bearer <token>`

---

### Task Endpoints

#### `POST /tasks`
**Create a new task**

**Request:**
```json
{
  "description": "Research Inworld TTS API and create integration plan",
  "priority": "HIGH",
  "assignee": "nix-tanaka",  // optional
  "parentTaskId": "uuid-123"  // optional, for subtasks
}
```

**Response:**
```json
{
  "id": "uuid-456",
  "status": "PENDING",
  "createdAt": "2026-02-08T23:45:00Z"
}
```

---

#### `GET /tasks`
**List tasks (with filters)**

**Query params:**
- `status` — Filter by status (PENDING, IN_PROGRESS, etc.)
- `priority` — Filter by priority
- `assignee` — Filter by agent ID
- `limit` — Max results (default 50)

**Response:**
```json
{
  "tasks": [
    {
      "id": "uuid-456",
      "description": "Research Inworld TTS",
      "priority": "HIGH",
      "status": "IN_PROGRESS",
      "assignee": "nix-tanaka",
      "progress": 45,
      "currentStep": "Reading API docs",
      "etaSeconds": 180,
      "createdAt": "2026-02-08T23:45:00Z",
      "startedAt": "2026-02-08T23:46:00Z"
    }
  ],
  "total": 12
}
```

---

#### `GET /tasks/:id`
**Get task details (includes progress history)**

**Response:**
```json
{
  "id": "uuid-456",
  "description": "Research Inworld TTS",
  "priority": "HIGH",
  "status": "IN_PROGRESS",
  "assignee": "nix-tanaka",
  "progress": 45,
  "currentStep": "Reading API docs",
  "etaSeconds": 180,
  "createdAt": "2026-02-08T23:45:00Z",
  "startedAt": "2026-02-08T23:46:00Z",
  "progressUpdates": [
    {
      "timestamp": "2026-02-08T23:46:30Z",
      "progress": 10,
      "step": "Found API docs"
    },
    {
      "timestamp": "2026-02-08T23:47:30Z",
      "progress": 30,
      "step": "Reading authentication section"
    },
    {
      "timestamp": "2026-02-08T23:48:30Z",
      "progress": 45,
      "step": "Reading API docs"
    }
  ]
}
```

---

#### `POST /tasks/:id/claim`
**Agent claims an unclaimed task**

**Request:**
```json
{
  "agentId": "nix-tanaka"
}
```

**Response:**
```json
{
  "id": "uuid-456",
  "status": "IN_PROGRESS",
  "claimedAt": "2026-02-08T23:46:00Z"
}
```

---

#### `POST /tasks/:id/progress`
**Agent reports progress on task**

**Request:**
```json
{
  "agentId": "nix-tanaka",
  "progress": 45,
  "currentStep": "Reading API docs",
  "etaSeconds": 180
}
```

**Response:**
```json
{
  "ok": true
}
```

---

#### `POST /tasks/:id/complete`
**Mark task as complete**

**Request:**
```json
{
  "agentId": "nix-tanaka",
  "result": "Integration plan created at research/inworld-tts-integration.md"
}
```

**Response:**
```json
{
  "id": "uuid-456",
  "status": "COMPLETE",
  "completedAt": "2026-02-08T23:50:00Z"
}
```

---

#### `POST /tasks/:id/block`
**Mark task as blocked (waiting on something)**

**Request:**
```json
{
  "agentId": "nix-tanaka",
  "reason": "Waiting for Riley's product feedback on approach"
}
```

**Response:**
```json
{
  "id": "uuid-456",
  "status": "BLOCKED",
  "blockingReason": "Waiting for Riley's product feedback"
}
```

---

#### `POST /tasks/:id/cancel`
**Cancel a task**

**Request:**
```json
{
  "userId": "jason",
  "reason": "No longer needed"
}
```

**Response:**
```json
{
  "id": "uuid-456",
  "status": "CANCELLED",
  "cancelledAt": "2026-02-08T23:52:00Z"
}
```

---

#### `PATCH /tasks/:id/priority`
**Change task priority (for escalation/de-escalation)**

**Request:**
```json
{
  "priority": "URGENT"
}
```

**Response:**
```json
{
  "id": "uuid-456",
  "priority": "URGENT"
}
```

---

### Agent Endpoints

#### `POST /agents/register`
**Agent registers with coordinator**

**Request:**
```json
{
  "agentId": "nix-tanaka",
  "sessionKey": "agent:nix-tanaka:main",
  "model": "claude-opus-4-6"
}
```

**Response:**
```json
{
  "ok": true,
  "registeredAt": "2026-02-08T23:45:00Z"
}
```

---

#### `POST /agents/:id/heartbeat`
**Agent sends heartbeat (I'm alive)**

**Request:**
```json
{
  "currentTaskId": "uuid-456",
  "status": "WORKING"
}
```

**Response:**
```json
{
  "ok": true
}
```

---

#### `GET /agents`
**List all registered agents and their status**

**Response:**
```json
{
  "agents": [
    {
      "agentId": "nix-tanaka",
      "status": "WORKING",
      "currentTaskId": "uuid-456",
      "lastHeartbeat": "2026-02-08T23:48:00Z",
      "sessionKey": "agent:nix-tanaka:main"
    },
    {
      "agentId": "riley-chase",
      "status": "IDLE",
      "currentTaskId": null,
      "lastHeartbeat": "2026-02-08T23:47:30Z",
      "sessionKey": "agent:riley-chase:main"
    }
  ]
}
```

---

#### `GET /agents/:id`
**Get specific agent status**

**Response:**
```json
{
  "agentId": "nix-tanaka",
  "status": "WORKING",
  "currentTaskId": "uuid-456",
  "lastHeartbeat": "2026-02-08T23:48:00Z",
  "sessionKey": "agent:nix-tanaka:main",
  "taskHistory": [
    {
      "taskId": "uuid-789",
      "description": "Fix GitHub auth",
      "startedAt": "2026-02-08T22:00:00Z",
      "completedAt": "2026-02-08T22:15:00Z"
    }
  ]
}
```

---

### Queue Endpoints

#### `GET /queue/next`
**Get next task for an agent (polling mode)**

**Query params:**
- `agentId` — Agent requesting task

**Response:**
```json
{
  "task": {
    "id": "uuid-456",
    "description": "Research Inworld TTS",
    "priority": "HIGH"
  }
}
```

OR (if no tasks available):
```json
{
  "task": null
}
```

---

#### `GET /queue/status`
**Get queue overview (how many tasks at each priority)**

**Response:**
```json
{
  "pending": {
    "URGENT": 1,
    "HIGH": 3,
    "NORMAL": 8,
    "LOW": 2
  },
  "inProgress": 4,
  "blocked": 1,
  "total": 19
}
```

---

## Agent Workflow (Detailed)

### Startup

```javascript
// Agent starts up (during OpenClaw session init)
await taskCoordinator.registerAgent({
  agentId: 'nix-tanaka',
  sessionKey: context.sessionKey,
  model: context.model
});

// Start heartbeat loop (every 30s)
setInterval(async () => {
  await taskCoordinator.sendHeartbeat({
    agentId: 'nix-tanaka',
    currentTaskId: currentTask?.id || null,
    status: currentTask ? 'WORKING' : 'IDLE'
  });
}, 30000);
```

### Task Execution

```javascript
// Main agent loop
while (true) {
  // Check for tasks (polling mode)
  const task = await taskCoordinator.getNextTask({ agentId: 'nix-tanaka' });
  
  if (!task) {
    // No tasks, wait 10s
    await sleep(10000);
    continue;
  }
  
  // Claim task
  await taskCoordinator.claimTask({ taskId: task.id, agentId: 'nix-tanaka' });
  
  // Work on task (with progress reporting)
  await executeTask(task, {
    onProgress: async (update) => {
      await taskCoordinator.reportProgress({
        taskId: task.id,
        agentId: 'nix-tanaka',
        progress: update.progress,
        currentStep: update.step,
        etaSeconds: update.eta
      });
    }
  });
  
  // Complete task
  await taskCoordinator.completeTask({
    taskId: task.id,
    agentId: 'nix-tanaka',
    result: 'Task done. Files: X, Y, Z'
  });
}
```

### Handling Interruptions

```javascript
// When URGENT task arrives while working on NORMAL task
if (newTask.priority === 'URGENT' && currentTask.priority === 'NORMAL') {
  // Pause current task
  await taskCoordinator.blockTask({
    taskId: currentTask.id,
    agentId: 'nix-tanaka',
    reason: 'Paused for urgent request'
  });
  
  // Start urgent task
  currentTask = newTask;
  await executeTask(newTask);
  
  // Resume paused task
  const pausedTask = await taskCoordinator.unblockTask(currentTask.id);
  currentTask = pausedTask;
  await executeTask(pausedTask);
}
```

---

## Monitoring & Debugging

### Dashboard (Web UI)

**V1 features:**
- Task queue view (pending, in-progress, blocked)
- Agent status view (who's working on what)
- Task detail view (progress history)
- Filters (by priority, agent, status)

**Tech stack:**
- Next.js (already used for family hub)
- tRPC (for type-safe API calls)
- Real-time updates via polling (upgrade to WebSocket later)

**Routes:**
- `/tasks` — Task queue list
- `/tasks/:id` — Task detail
- `/agents` — Agent status list
- `/agents/:id` — Agent detail + task history

### CLI

**Commands:**

```bash
# List tasks
openclaw tasks list --status=PENDING --priority=HIGH

# View task details
openclaw tasks show uuid-456

# Cancel task
openclaw tasks cancel uuid-456

# List agents
openclaw agents list

# View agent status
openclaw agents show nix-tanaka
```

**Implementation:**
- Add `tasks` and `agents` subcommands to OpenClaw CLI
- Call Task Coordinator API under the hood

---

## Rollout Plan

### Phase 1: Core MVP (1 week)

**Deliverables:**
- Task Coordinator service (SQLite storage)
- REST API (task CRUD, claiming, progress)
- Agent integration (polling mode)
- Basic CLI (`openclaw tasks list/show`)

**Testing:**
- Nix creates task, claims it, reports progress, completes
- Jason creates URGENT task, Nix interrupts current work

### Phase 2: Monitoring (1 week)

**Deliverables:**
- Web dashboard (task queue + agent status)
- Heartbeat monitoring (detect stuck agents)
- Progress history view

### Phase 3: Collaboration (1-2 weeks)

**Deliverables:**
- Subtask creation (agent delegates to another agent)
- Blocking/handoff workflow
- Cross-agent coordination APIs

### Phase 4: Polish & Open Source (1-2 weeks)

**Deliverables:**
- Push mode (WebSocket for efficiency)
- Automatic task breakdown (planner agent)
- Documentation for external users
- GitHub repo public + README

---

## Open Source Considerations

**If open-sourced:**

1. **Standalone or Plugin?**
   - Standalone: Separate repo, works with any OpenClaw install
   - Plugin: Part of OpenClaw core, enabled via config

2. **Storage flexibility:**
   - Support SQLite (for single-user) AND Postgres (for teams)
   - Config-driven (user picks backend)

3. **Multi-user:**
   - Current design is single-user (Jason) with multiple agents
   - For open source, support multiple users (each with their own agents)
   - Add `userId` to tasks table, filter by user

4. **Security:**
   - API authentication (token-based)
   - Rate limiting (prevent abuse)
   - Input validation (prevent SQL injection, XSS)

5. **Documentation:**
   - README with quick start guide
   - API reference (OpenAPI/Swagger)
   - Architecture diagram (this doc)
   - Examples (common workflows)

---

## Alternatives Considered

### Alternative 1: No Coordinator (Agent-to-Agent Direct)

**Idea:** Agents coordinate via direct messages (sessions_send), no central queue.

**Pros:**
- Simpler (no new service)
- More "agent-native"

**Cons:**
- No visibility into queue (user can't see pending work)
- Hard to prioritize (no URGENT vs LOW)
- Hard to interrupt (agent might not check messages while working)

**Verdict:** Rejected. Central coordinator is worth the complexity.

### Alternative 2: Use Existing Task Tracker (Linear, GitHub Issues)

**Idea:** Agents create/update issues in external tracker, no custom system.

**Pros:**
- Reuse existing tools
- User already familiar with UI

**Cons:**
- External API latency (slower)
- Not designed for real-time coordination (30-60s updates)
- Hard to integrate interruption logic

**Verdict:** Rejected. Custom system gives us the control we need.

### Alternative 3: LangGraph / LangChain Orchestration

**Idea:** Use LangGraph's multi-agent orchestration.

**Pros:**
- Battle-tested framework
- Built-in state management

**Cons:**
- Heavy dependency (adds complexity)
- Opinionated workflow (may not fit our needs)
- Harder to customize

**Verdict:** Rejected for V1. Could revisit if we need more complex orchestration later.

---

## Next Steps

1. **Get feedback** from Jason (architecture review)
2. **Create GitHub repo** (`openclaw-task-coordinator`)
3. **Scaffold project:**
   - Node.js service
   - SQLite setup
   - REST API skeleton
4. **Implement core endpoints:**
   - POST /tasks
   - GET /tasks
   - POST /tasks/:id/claim
   - POST /tasks/:id/progress
   - POST /tasks/:id/complete
5. **Integrate with one agent** (Nix) and test end-to-end
6. **Iterate** based on real-world usage

---

*Architecture complete. Ready for implementation.*
