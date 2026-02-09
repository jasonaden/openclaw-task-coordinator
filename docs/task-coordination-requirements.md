# Task Coordination System - Requirements

**Author:** Nix Tanaka  
**Date:** 2026-02-08  
**Status:** Draft

---

## Problem Statement

**Current state:** Agents receive tasks and go deep without checking in. Users can't see progress, can't interrupt, can't collaborate across agents, and get slow responses when agents are working on other things.

**Symptoms:**
- Agent works for 5+ minutes without updates
- User can't tell if agent is stuck or making progress
- Can't prioritize urgent requests over background work
- Multiple agents can't collaborate on complex tasks
- No way to interrupt long-running work

**Goal:** Task queue + coordination system that enables:
- Transparent progress tracking
- Agent collaboration (break work into chunks, hand off)
- Interruption/reprioritization
- Fast response to urgent requests

---

## User Stories

### 1. User with Urgent Request
**As a user,** when I have an urgent question while my agent is working on something else,  
**I want** the agent to pause and respond quickly,  
**So that** I'm not blocked waiting for background work to finish.

### 2. User Watching Progress
**As a user,** when my agent is working on a complex task,  
**I want** to see progress updates every 30-60 seconds,  
**So that** I know it's working and can intervene if it's going off track.

### 3. Agent Working on Large Task
**As an agent,** when I receive a multi-step task,  
**I want** to break it into small chunks and report progress after each,  
**So that** the user sees I'm making progress and can interrupt if needed.

### 4. Agent Collaborating with Others
**As an agent,** when I need help from another agent (e.g., product input from Riley),  
**I want** to hand off a subtask and resume when they're done,  
**So that** we can work together without blocking each other.

### 5. System Admin Debugging
**As a system admin,** when agents seem slow or unresponsive,  
**I want** to see the task queue and each agent's current work,  
**So that** I can diagnose bottlenecks or stuck agents.

---

## Core Requirements

### Task Queue

**R1. Task Creation**
- Any agent or user can create a task
- Tasks have: description, priority, assignee (optional), status
- Priority levels: URGENT, HIGH, NORMAL, LOW
- Status: PENDING, IN_PROGRESS, BLOCKED, COMPLETE, CANCELLED

**R2. Task Assignment**
- Tasks can be assigned to specific agent OR unclaimed (any agent can pick up)
- Agents can claim unclaimed tasks (FIFO within priority level)
- Assigned agent can reject/reassign if not appropriate

**R3. Task Priority**
- URGENT: Interrupt current work, respond within 30s
- HIGH: Start after current chunk completes (~30-60s)
- NORMAL: Work through in order
- LOW: Background work, do when nothing else queued

**R4. Task Breakdown**
- Large tasks (>5 min estimate) MUST be broken into subtasks
- Each subtask should be achievable in ~30-60 seconds
- Parent task tracks progress via subtask completion

### Progress Reporting

**R5. Regular Heartbeats**
- Agents report progress every 30-60 seconds while working
- Progress updates include: task ID, current step, % complete (estimate), ETA
- If agent doesn't report for 90 seconds → flagged as potentially stuck

**R6. Completion Reporting**
- When task completes, agent posts summary + artifacts (files, links, etc.)
- User can accept completion or request changes (reopens task)

**R7. Blocking/Handoff**
- Agent can mark task as BLOCKED with reason (waiting on X from Y)
- Blocked tasks don't count against agent's active work
- When unblocked, task re-enters queue at original priority

### Collaboration

**R8. Subtask Delegation**
- Agent working on task can create subtask and assign to another agent
- Parent task remains IN_PROGRESS, subtask gets its own lifecycle
- When subtask completes, parent agent gets notification

**R9. Cross-Agent Coordination**
- Agents can query: "Who's working on X?" "What's Y's current task?"
- Agents can request: "Zak, need you to route this to Riley when she's free"

### Interruption & Reprioritization

**R10. User Interruption**
- User can send URGENT message → creates URGENT task, interrupts current work
- Agent acknowledges interruption within 30s, estimates when it can resume prior work

**R11. Priority Escalation**
- User or admin can escalate existing task priority
- Agent working on lower-priority task gets notification, decides whether to switch

**R12. Task Cancellation**
- User can cancel tasks (in-progress or pending)
- Agent stops work immediately, posts summary of partial progress

---

## System Constraints

### Technical

**C1. Stateless Agents**
- Agents can crash/restart — task queue must persist
- Task state stored in database (not agent memory)
- Agents poll queue or receive push notifications

**C2. Latency**
- Task creation → agent acknowledgment: <5 seconds (URGENT), <30 seconds (others)
- Progress updates: every 30-60 seconds
- Task completion → user notification: <5 seconds

**C3. Scalability**
- Support 5 agents initially (Jason's team)
- Design for 50+ agents (if open-sourced)
- Task queue can handle 100+ tasks/hour

**C4. Persistence**
- Task history retained for 30 days (configurable)
- Completed tasks archived (can be queried but not in active queue)

### Usability

**C5. Transparency**
- User can view task queue at any time (web UI or CLI)
- User can see each agent's current task and progress
- User can see task history (what was worked on, when)

**C6. Low Overhead**
- Progress reporting should take <5 seconds per update
- Agents shouldn't spend >10% of time on coordination overhead

---

## Success Metrics

**M1. Response Time**
- 90% of URGENT tasks acknowledged within 30s
- 90% of HIGH tasks started within 60s

**M2. Progress Transparency**
- 100% of tasks >2 minutes have at least one progress update
- 90% of tasks >5 minutes have updates every 60s

**M3. Collaboration**
- Agents can hand off subtasks without user intervention
- Task handoff time (blocked → assigned to other agent) <30s

**M4. Interruption**
- User can interrupt any task with URGENT request
- Agent responds to interruption within 30s, resumes prior work when clear

**M5. User Satisfaction**
- User survey: "I can see what my agents are working on" → 8/10 avg
- User survey: "I can interrupt and reprioritize easily" → 8/10 avg

---

## Out of Scope (for V1)

- **Automatic task planning** — V1 requires agents to manually break down tasks
- **Dependency graphs** — V1 doesn't model task dependencies (just blocking/handoff)
- **Multi-user coordination** — V1 is single-user (Jason) with multiple agents
- **Time tracking / billing** — V1 doesn't track time spent per task
- **SLA enforcement** — V1 doesn't automatically escalate if tasks miss deadlines

---

## Open Questions

1. **Task storage:** Postgres? SQLite? Redis? In-memory with periodic snapshots?
2. **Agent polling vs push:** Should agents poll the queue every N seconds, or receive push notifications?
3. **Planning agent:** Should there be a dedicated "planner" agent that breaks tasks down, or does each agent do its own planning?
4. **User interface:** Web UI? CLI? Telegram bot commands? All of the above?
5. **Task granularity:** Is 30-60 seconds the right chunk size, or should it be configurable?

---

## Next Steps

1. **Architecture design** — System components, data model, API endpoints
2. **API specification** — REST or GraphQL? Endpoints for task CRUD, claiming, progress updates
3. **Prototype** — Minimal implementation (task queue + one agent reporting progress)
4. **GitHub repo + PR** — Code + docs + README

---

*Requirements complete. Moving to architecture next.*
