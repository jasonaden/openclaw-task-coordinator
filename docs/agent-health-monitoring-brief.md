# Agent Health Monitoring - Brief

**Author:** Nix Tanaka  
**Date:** 2026-02-08  
**Status:** Draft

---

## Purpose

Monitor agent health (alive, responsive, not stuck) so users and admins can detect problems before they impact work.

---

## Problem

**Symptoms of unhealthy agents:**
- Agent stops responding to messages
- Agent claims task but makes no progress
- Agent crashes/restarts without notification
- Agent is working but stuck in a loop

**Current state:** No visibility. User has to guess if agent is working, stuck, or dead.

---

## Solution: Health Monitoring System

### 1. Heartbeat Protocol

**Agents send regular heartbeats:**
- Every 30 seconds (configurable)
- Include: agent ID, current task ID (if working), status (IDLE/WORKING)

**Coordinator tracks last heartbeat:**
- If >90 seconds since last heartbeat → mark agent as STALE
- If >5 minutes → mark agent as DEAD

**Status levels:**
- **HEALTHY** — Heartbeat received within 90s
- **STALE** — No heartbeat for 90s-5min (might be stuck)
- **DEAD** — No heartbeat for >5min (crashed or unresponsive)

### 2. Progress Tracking

**Agents report progress while working:**
- Every 30-60 seconds during task execution
- Include: task ID, progress %, current step, ETA

**Coordinator detects stuck agents:**
- If task IN_PROGRESS but no progress update for >2 minutes → flag as POTENTIALLY_STUCK
- If progress % doesn't increase for >5 minutes → flag as STUCK

### 3. Health Check API

**Endpoint:** `GET /health/agents`

**Response:**
```json
{
  "agents": [
    {
      "agentId": "nix-tanaka",
      "status": "HEALTHY",
      "currentTask": "uuid-456",
      "lastHeartbeat": "2026-02-08T23:48:00Z",
      "lastProgress": "2026-02-08T23:47:30Z"
    },
    {
      "agentId": "riley-chase",
      "status": "STALE",
      "currentTask": null,
      "lastHeartbeat": "2026-02-08T23:45:00Z",
      "lastProgress": null,
      "alert": "No heartbeat for 3 minutes"
    }
  ]
}
```

**Endpoint:** `GET /health/agents/:id`

**Response:**
```json
{
  "agentId": "nix-tanaka",
  "status": "HEALTHY",
  "currentTask": {
    "id": "uuid-456",
    "description": "Research Inworld TTS",
    "progress": 45,
    "currentStep": "Reading API docs",
    "lastUpdate": "2026-02-08T23:47:30Z"
  },
  "heartbeat": {
    "last": "2026-02-08T23:48:00Z",
    "intervalMs": 30000,
    "missedCount": 0
  },
  "session": {
    "key": "agent:nix-tanaka:main",
    "model": "claude-opus-4-6",
    "startedAt": "2026-02-08T10:00:00Z"
  }
}
```

### 4. Alert System

**When agent becomes unhealthy:**

**STALE (90s no heartbeat):**
- Log warning (no user notification yet)
- Admin dashboard shows yellow indicator

**DEAD (5min no heartbeat):**
- Send notification to user (Telegram/web)
- Admin dashboard shows red indicator
- Auto-cancel or reassign agent's current task (configurable)

**STUCK (no progress for 5min):**
- Send notification to user
- Suggest: "Nix seems stuck on task X. Cancel or wait?"

### 5. Auto-Recovery

**When dead agent comes back online:**
- Agent re-registers with coordinator
- Coordinator checks if agent had tasks in progress
- Offer to resume or reassign

**When stuck agent completes:**
- Clear STUCK flag
- Log recovery time (for debugging)

---

## Metrics to Track

### Per-Agent
- Uptime (time since last restart)
- Heartbeat reliability (% of expected heartbeats received)
- Task completion rate (completed / claimed)
- Average time per task
- Stuck incidents (count)

### System-Wide
- Total agents active
- Agents healthy vs stale vs dead
- Tasks blocked due to agent issues
- Average response time (task created → claimed)

---

## Dashboard View

**Agent Health Page:**

```
┌─────────────────────────────────────────────────────────┐
│ Agent Health                                            │
├─────────────────────────────────────────────────────────┤
│ ● nix-tanaka        HEALTHY      Working on task #456  │
│   Last heartbeat: 12s ago                               │
│   Progress: 45% - Reading API docs                      │
│                                                          │
│ ● riley-chase       IDLE         No current task        │
│   Last heartbeat: 8s ago                                │
│                                                          │
│ ⚠ leo-vance         STALE        Working on task #789   │
│   Last heartbeat: 2m ago         No progress update     │
│   Alert: Possibly stuck or unresponsive                 │
│                                                          │
│ ✖ sunny-meadows     DEAD         Last task: #234        │
│   Last heartbeat: 6m ago                                │
│   Alert: Agent offline (crashed or stopped)             │
└─────────────────────────────────────────────────────────┘
```

---

## CLI Commands

```bash
# Check all agent health
openclaw agents health

# Check specific agent
openclaw agents health nix-tanaka

# Show stuck agents
openclaw agents stuck

# Force-restart agent (if stuck)
openclaw agents restart nix-tanaka
```

---

## Implementation Notes

### Heartbeat Storage

**Option A: In-memory (fast, volatile)**
- Store last heartbeat in memory
- Lost on coordinator restart (not critical)
- Fast lookups

**Option B: Database (persistent)**
- Store in `agent_heartbeats` table
- Survives coordinator restart
- Slightly slower

**Recommendation: In-memory with periodic DB sync.** Fast for real-time checks, persistent for history.

### Background Job

**Coordinator runs health check every 30s:**

```javascript
setInterval(async () => {
  const agents = await getAllAgents();
  const now = Date.now();
  
  for (const agent of agents) {
    const timeSinceHeartbeat = now - agent.lastHeartbeat;
    
    if (timeSinceHeartbeat > 5 * 60 * 1000) {
      // Dead (5min)
      await markAgentDead(agent.id);
      await notifyUser(`Agent ${agent.id} is offline`);
    } else if (timeSinceHeartbeat > 90 * 1000) {
      // Stale (90s)
      await markAgentStale(agent.id);
    }
    
    // Check for stuck tasks
    if (agent.currentTask) {
      const timeSinceProgress = now - agent.lastProgressUpdate;
      if (timeSinceProgress > 5 * 60 * 1000) {
        await markAgentStuck(agent.id, agent.currentTask);
        await notifyUser(`Agent ${agent.id} seems stuck on task ${agent.currentTask}`);
      }
    }
  }
}, 30000);
```

---

## Integration with Task Coordinator

**Health monitoring is part of Task Coordinator service** (not separate).

**Shared state:**
- Agent registry (who's registered)
- Current tasks (what each agent is working on)
- Progress updates (when last update was received)

**API endpoints use same auth and port.**

---

## Testing

**Scenarios to test:**

1. **Normal operation** — Agent sends heartbeats, works on tasks, completes
2. **Agent crash** — Agent stops sending heartbeats, coordinator detects DEAD
3. **Agent stuck** — Agent claims task but makes no progress, coordinator detects STUCK
4. **Agent recovery** — Dead agent comes back online, re-registers, resumes work
5. **Network hiccup** — Agent misses 1-2 heartbeats but recovers (should not trigger alert)

---

## Rollout

**Phase 1 (with Task Coordinator MVP):**
- Basic heartbeat protocol (agents → coordinator)
- Health check API (`GET /health/agents`)
- CLI command (`openclaw agents health`)

**Phase 2 (with Dashboard):**
- Web UI for agent health
- Real-time status updates
- Visual indicators (green/yellow/red)

**Phase 3 (Alerts):**
- User notifications (Telegram/web) when agent unhealthy
- Auto-recovery (reassign tasks from dead agents)
- Metrics tracking (uptime, reliability)

---

## Open Questions

1. **Heartbeat interval:** 30s is default, should it be configurable per agent?
2. **Auto-recovery:** Should dead agent's tasks be auto-reassigned, or wait for user decision?
3. **Stuck detection:** 5min no progress is current threshold, too long or too short?
4. **User notifications:** Only for DEAD agents, or also STALE/STUCK?

---

## Summary

**Health monitoring is simple heartbeat protocol + stuck detection.**

- Agents send "I'm alive" every 30s
- Coordinator watches for missing heartbeats (STALE/DEAD)
- Coordinator watches for no-progress tasks (STUCK)
- Dashboard shows status, alerts user when problems detected

**Core value:** User knows agents are working, gets notified if something goes wrong.

**Implementation:** 1-2 days (part of Task Coordinator service).

---

*Health monitoring brief complete. Ready for GitHub repo setup.*
