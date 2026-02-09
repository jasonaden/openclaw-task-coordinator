# OpenClaw Task Coordinator

**Task coordination and health monitoring system for OpenClaw agents**

---

## Overview

OpenClaw Task Coordinator solves a critical problem: agents go deep on tasks without checking in, making it impossible to see progress, interrupt work, or collaborate across agents.

**This system provides:**

1. **Task queue with priorities** (URGENT, HIGH, NORMAL, LOW)
2. **Progress tracking** (agents report status every 30-60s)
3. **Agent health monitoring** (heartbeats, stuck detection, auto-recovery)
4. **Interruption support** (reprioritize or cancel tasks)
5. **Agent collaboration** (subtasks, handoffs, blocking)

---

## Key Features

### 🎯 Task Management
- Create tasks with priority levels
- Assign to specific agents or leave unclaimed (first-come)
- Break large tasks into subtasks
- Track progress with regular updates
- Complete, cancel, or block tasks

### 📊 Progress Transparency
- Agents report progress every 30-60 seconds
- See current step, % complete, ETA
- View full progress history per task
- Dashboard shows queue status at a glance

### 🏥 Health Monitoring
- Agents send heartbeats every 30s
- Detect STALE (no heartbeat 90s) or DEAD (no heartbeat 5min)
- Detect STUCK (no progress for 5min)
- Auto-recovery when agents come back online
- Metrics: uptime, completion rate, stuck incidents

### 🔄 Collaboration
- Agents can delegate subtasks to other agents
- Block tasks while waiting for dependencies
- Hand off work between agents
- Cross-agent coordination APIs

### ⚡ Interruption
- User can create URGENT tasks that interrupt current work
- Escalate task priority mid-execution
- Cancel tasks and see partial progress
- Agent resumes prior work after interruption

---

## Documentation

- **[Requirements](docs/task-coordination-requirements.md)** — Problem statement, user stories, core requirements
- **[Architecture](docs/task-coordination-architecture.md)** — System design, data model, API specification
- **[Health Monitoring](docs/agent-health-monitoring-brief.md)** — Heartbeat protocol, stuck detection, alerts

---

## Quick Start

### Installation

```bash
# Clone the repo
git clone https://github.com/jasonaden/openclaw-task-coordinator.git
cd openclaw-task-coordinator

# Install dependencies
npm install

# Set up database
npm run db:setup

# Start the coordinator
npm start
```

### Configuration

Add to your OpenClaw config:

```json5
{
  taskCoordinator: {
    enabled: true,
    storage: "sqlite",
    dbPath: "~/.openclaw/tasks.db",
    heartbeatIntervalMs: 30000,
    staleThresholdMs: 90000,
    api: {
      port: 18790,
      auth: "your-gateway-token"
    }
  }
}
```

### Agent Integration

Agents automatically integrate with the coordinator. No code changes needed.

**How it works:**
1. Agent registers on startup
2. Polls for tasks every 10-30 seconds
3. Reports progress every 30-60 seconds while working
4. Sends heartbeat every 30 seconds

---

## API Examples

### Create a Task

```bash
curl -X POST http://localhost:18790/api/tasks \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Research Inworld TTS API",
    "priority": "HIGH",
    "assignee": "nix-tanaka"
  }'
```

### List Tasks

```bash
curl http://localhost:18790/api/tasks?status=IN_PROGRESS \
  -H "Authorization: Bearer <token>"
```

### Check Agent Health

```bash
curl http://localhost:18790/api/health/agents \
  -H "Authorization: Bearer <token>"
```

See [API Documentation](docs/task-coordination-architecture.md#api-specification) for full reference.

---

## CLI Usage

```bash
# List tasks
openclaw tasks list --status=PENDING --priority=HIGH

# View task details
openclaw tasks show <task-id>

# Cancel task
openclaw tasks cancel <task-id>

# Check agent health
openclaw agents health

# View specific agent
openclaw agents show nix-tanaka
```

---

## Architecture

```
User (Telegram/CLI/Web)
  ↓
OpenClaw Gateway
  ↓
Task Coordinator (Node.js service)
  ├── Task Queue (priority-sorted)
  ├── Agent Registry (status, current task)
  ├── Progress Tracker (heartbeat monitor)
  └── REST API
  ↓
SQLite/Postgres Database
  ├── tasks
  ├── progress_updates
  └── agent_heartbeats
  ↓
Agents (Nix, Riley, Leo, etc.)
```

**Key components:**

- **Task Coordinator** — Central service managing queue, agents, progress
- **Storage Layer** — SQLite (MVP) or Postgres (scale)
- **Agent Integration** — Polling mode (simple) or push mode (efficient)
- **Web Dashboard** — Real-time task queue and agent status view
- **CLI** — Command-line tools for task and agent management

See [Architecture Doc](docs/task-coordination-architecture.md) for details.

---

## Roadmap

### Phase 1: Core MVP (1 week)
- [x] Requirements doc
- [x] Architecture doc
- [x] Health monitoring spec
- [ ] Task Coordinator service (SQLite)
- [ ] REST API (task CRUD, claiming, progress)
- [ ] Agent integration (polling mode)
- [ ] Basic CLI

### Phase 2: Monitoring (1 week)
- [ ] Web dashboard (task queue + agent status)
- [ ] Heartbeat monitoring (detect stuck agents)
- [ ] Progress history view

### Phase 3: Collaboration (1-2 weeks)
- [ ] Subtask creation and delegation
- [ ] Blocking/handoff workflow
- [ ] Cross-agent coordination APIs

### Phase 4: Polish & Open Source (1-2 weeks)
- [ ] Push mode (WebSocket)
- [ ] Automatic task breakdown (planner agent)
- [ ] External documentation
- [ ] Public release

---

## Use Cases

### Use Case 1: Urgent Interruption
**Scenario:** User has urgent question while agent is working on background research.

**Flow:**
1. User sends "urgent: need this now!"
2. System creates URGENT task
3. Agent gets notification, pauses current work
4. Agent responds to urgent request within 30s
5. Agent resumes background research

### Use Case 2: Progress Tracking
**Scenario:** User asks agent to build a complex feature.

**Flow:**
1. Agent breaks work into subtasks (design, implement, test)
2. Agent reports progress every 60s: "Writing code for X component"
3. User sees updates in real-time, knows work is progressing
4. User can interrupt if direction seems wrong

### Use Case 3: Agent Collaboration
**Scenario:** Engineering agent needs product input from product agent.

**Flow:**
1. Nix (engineer) starts task: "Build feature X"
2. Nix creates subtask: "Riley, validate this approach"
3. Nix blocks main task, waits for Riley
4. Riley completes subtask with feedback
5. Nix receives notification, unblocks, continues work

### Use Case 4: Stuck Detection
**Scenario:** Agent gets stuck in a loop, stops making progress.

**Flow:**
1. Agent claims task, starts work
2. Reports initial progress: "Reading docs" (30s)
3. No progress for 5 minutes (stuck)
4. System detects stuck state, notifies user
5. User cancels or helps debug

---

## Contributing

This system is designed to be open-sourced for the OpenClaw community.

**To contribute:**
1. Fork the repo
2. Create a feature branch
3. Make your changes
4. Submit a pull request

**Areas needing help:**
- Agent integration libraries (Python, other languages)
- Web dashboard UI improvements
- Additional storage backends (Redis, DynamoDB)
- Testing and bug reports

---

## License

MIT (to be confirmed)

---

## Contact

**Author:** Nix Tanaka (OpenClaw Engineering Agent)  
**Project:** OpenClaw Task Coordinator  
**Repo:** https://github.com/jasonaden/openclaw-task-coordinator

**Questions?** Open an issue or ask in the OpenClaw Discord.

---

*Making agent coordination transparent, interruptible, and collaborative.*
