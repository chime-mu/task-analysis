# Task System Comparison: Claude Code vs Beads

## Overview

This document compares two approaches to AI-agent task management:

- **Claude Code Tasks**: Claude Code v2.1.34 has **two mutually exclusive** task systems -- TodoWrite (solo/in-memory) and Tasks (team/swarm/persisted-to-disk) -- with a built-in team coordination layer, plan mode, completion verification hooks, inter-agent messaging, an idle notification system, and a 29-type context attachment system
- **Beads**: A persistent, git-native, multi-project issue tracker designed for multi-agent workflows

The gap between these systems is narrower than a surface-level inspection suggests. Claude Code's swarm-mode Tasks system provides file-based persistence, dependency tracking, multi-agent ownership, atomic claiming with file locking, agent busy checks, completion verification via sub-agents, structured inter-agent messaging, plan mode with team-lead approval, and automatic idle notifications. However, Beads still operates at a fundamentally different scale in several dimensions -- particularly multi-project management, git-native distribution, workflow templates, and long-term context management.

---

## Architecture Note: Claude Code's Two Systems

Claude Code does not have one task system. It has two, controlled by a feature gate (`CLAUDE_CODE_ENABLE_TASKS`):

| System | Tool Names | Storage | Scope | Enabled When |
|--------|-----------|---------|-------|-------------|
| **TodoWrite** | `TodoWrite` | In-memory (`AppState.todos`) + disk persistence at `~/.claude/todos/` | Per-agent, per-session | Tasks feature **disabled** (solo mode) |
| **Tasks** | `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList` | Filesystem JSON at `~/.claude/tasks/{team-name}/` | Shared across team agents | Tasks feature **enabled** (team/swarm mode) |

These are **mutually exclusive** -- `isEnabled()` returns opposite values for each. In interactive mode, the Tasks system is enabled by default, and TodoWrite is disabled.

Beyond the core task CRUD, the Tasks system includes:
- **Team management tools**: `TeamCreate`, `TeamDelete`, `SendMessage` (broadcast + direct messaging)
- **Background task infrastructure**: `TaskOutput`, `TaskStop` for local bash, local agents, remote agents, and in-process teammates
- **Plan mode**: `EnterPlanMode`, `ExitPlanMode` with team-lead approval for structured planning
- **Completion verification**: Agent stop hooks that spawn sub-agents to validate task completion
- **Reminder system**: Automatic nudges after 10+ idle turns without task tool usage
- **Idle notification system**: Automatic notifications when teammates finish their turns
- **Attachment system**: 29 distinct context injection types per turn (task_progress, task_reminder, task_status, team_context, teammate_mailbox, etc.)

Note: Even TodoWrite has a disk persistence layer at `~/.claude/todos/{sessionId}-agent-{agentId}.json`. In a live system, 6,062 such files were observed (98% empty/cleared, ~2% with active content). This is separate from the primary in-memory storage and appears to be a framework-level serialization mechanism.

This comparison focuses on the **Tasks system** (swarm mode) since it represents Claude Code's full capabilities.

---

## Side-by-Side Comparison

| Dimension | Claude Code Tasks (Swarm Mode) | Beads |
|-----------|-------------------------------|-------|
| **Scope** | Single machine, team of agents in one session | Multi-project, multi-agent, multi-org |
| **Persistence** | **Yes** -- JSON files on disk (`~/.claude/tasks/{team}/`) | Yes -- SQLite + JSONL in git |
| **Storage** | Individual JSON files per task + file locking | SQLite + JSONL + optional Dolt |
| **Data Model** | ~10 fields (id, subject, description, activeForm, status, owner, blocks, blockedBy, metadata) | ~40+ fields (full issue tracker) |
| **ID System** | Auto-incrementing integers ("1", "2", "3") with highwatermark | Hash-based (`bd-a1b2`) |
| **States** | 3 + delete: pending, in_progress, completed, (deleted via TaskUpdate) | 7+: open, in_progress, blocked, deferred, closed, hooked, tombstone |
| **Dependencies** | blocks/blockedBy (2 types, enforced at claim time) | 6 types: blocks, parent-child, conditional-blocks, waits-for, related, discovered-from |
| **Multi-project** | No | Yes (multi-repo hydration, cross-project deps) |
| **Multi-agent** | **Yes** -- team system with ownership, atomic claiming, file locking, agent busy checks, auto-unassign on shutdown | Yes -- hash IDs, atomic claims, agent-as-bead, hook pattern |
| **Inter-agent messaging** | **Yes** -- SendMessage tool (broadcast + direct), idle notifications, teammate mailbox | No built-in messaging (relies on git-shared state) |
| **Quality gates** | **Yes** -- completion verification hooks spawn sub-agents to validate before marking complete | Yes -- gate steps in molecules (wait for CI, human approval, etc.) |
| **Planning workflow** | **Yes** -- EnterPlanMode/ExitPlanMode with plan files, team-lead approval | No formal planning mode (agents plan organically) |
| **Workflow Engine** | **Partial** -- plan mode provides a plan-approve-execute cycle; no reusable templates or formulas | Yes (molecules, formulas, gates) |
| **Agent reminders** | **Yes** -- automatic nudges after 10+ idle turns | No (agents check `bd ready` on their own) |
| **Cross-session** | **Partial** -- task files persist on disk, but no explicit resume protocol or ready computation | Yes (git-persisted, handoff protocols, `bd ready`) |
| **Context injection** | **Yes** -- 29 attachment types per turn (task progress, reminders, team context, etc.) | Minimal (CLI output, `--json` for agents) |
| **Compaction** | None | Semantic memory decay (tier 1/2 compression) |
| **Federation** | None | Yes (peer-to-peer via Dolt) |
| **Git integration** | None -- filesystem only | Native (JSONL tracked in git, daemon syncs) |
| **UI** | Terminal status line with spinners, expanded tasks view, teammate status display | CLI output + `--json` for agents |
| **Integration** | Built into Claude Code agent loop | CLI, MCP server, Claude Code plugin |
| **Language** | TypeScript (Bun-compiled binary) | Go |
| **Setup** | Zero (built-in, default enabled in interactive mode) | `bd init` per project |

---

## Detailed Comparison by Category

### 1. Persistence and Lifecycle

**Claude Code Tasks** are persistent in swarm/team mode. Each task is written as an individual JSON file at `~/.claude/tasks/{sanitized-team-name}/{id}.json`. The system uses `proper-lockfile` for concurrent access, maintains a `.highwatermark` file to ensure monotonically increasing IDs even after deletions, and supports a full CRUD lifecycle (create, read, update, delete, list, clear-all). Task files survive across process restarts within the same team context.

However, this persistence is **team-scoped, not project-scoped**. Tasks are keyed by team name, not by working directory or git repository. There is no built-in mechanism to associate tasks with a specific project, resume tasks in a new session, or aggregate tasks across different teams/projects.

Even TodoWrite (solo mode) has a disk persistence layer at `~/.claude/todos/`, though its primary storage is in-memory. This persistence appears to be a framework-level serialization rather than a deliberate cross-session feature.

**Beads** is fundamentally persistent and project-scoped. Issues are stored in SQLite for fast queries and exported to JSONL tracked in git. They survive across sessions, across machines, across teams. The daemon handles background sync, and the "Landing the Plane" protocol ensures agents push state before ending sessions. Compaction manages the long tail of old issues.

**Key difference**: Claude Code persists to local filesystem within a team context. Beads persists to git, enabling distribution, version history, and cross-machine access.

### 2. Data Model Complexity

**Claude Code Tasks** have a focused model: id, subject, description, activeForm, status, owner, blocks, blockedBy, metadata. The metadata field is an arbitrary `Record<string, unknown>` with a special `_internal` key that hides tasks from listing. This is lean but sufficient for coordinating work among a team of agents.

**Beads** has 40+ fields covering issue content, assignment, timestamps, external integration, compaction metadata, tombstone fields, agent identity, molecule/workflow fields, gate fields, slot fields, and event fields. This richness supports the full range of project management scenarios, from simple task tracking to federated multi-org workflows.

**Key difference**: Claude Code's model is minimal but extensible via metadata. Beads' model is comprehensive and purpose-built for every scenario it supports.

### 3. Dependency System

**Claude Code Tasks** support `blocks`/`blockedBy` relationships with **enforcement at claim time**. When an agent attempts to claim a task via `oBA()`, the system checks for active blockers -- if any blockedBy task is still non-completed, the claim returns `{ success: false, reason: "blocked" }`. Dependencies are also cleaned up on deletion (references to the deleted task are removed from all other tasks' blocks/blockedBy arrays). TaskList filters completed tasks from blockedBy in its display output so agents see only active blockers.

This is more than advisory -- it is soft enforcement. The agent can still set status to in_progress via TaskUpdate without going through the claim path, but the claim mechanism provides a formal gate.

**Beads** has 6 dependency types with different semantics:
- `blocks` / `parent-child` / `conditional-blocks` / `waits-for` -- affect readiness computation
- `related` / `discovered-from` -- informational only

Beads materializes a `blocked_issues_cache` in SQLite for O(1) readiness checks and provides `bd ready` to compute available work. Dependencies form the backbone of the workflow engine.

**Key difference**: Claude Code has one dependency type with enforcement at claim time. Beads has six dependency types with rich semantic meaning and a dedicated ready-work computation engine.

### 4. Multi-Agent Coordination

**Claude Code Tasks** have a sophisticated multi-agent coordination system in swarm mode:

- **Team management**: `TeamCreate`, `TeamDelete` for team lifecycle; one team at a time per leader
- **Inter-agent messaging**: `SendMessage` tool with two message types -- `broadcast` (team-wide announcements) and `message` (direct to specific teammates). System prompt enforces: "Your team cannot hear you if you do not use the SendMessage tool."
- **Team lead role**: Special privileges including creating/deleting teams, approving/rejecting plans from teammates, receiving idle notifications. The team lead is always referenced by name ("team-lead"), never by UUID
- **Ownership**: Tasks have an `owner` field; auto-assigned to the claiming agent's name in swarm mode
- **Atomic claiming**: `oBA()` acquires a file lock on the individual task file before checking and updating ownership
- **Agent busy check**: `SyD()` acquires a global lock and verifies the agent does not already own another active task before allowing a claim
- **Claim failure reasons**: Structured responses for `task_not_found`, `already_claimed`, `already_resolved`, `blocked`, `agent_busy`
- **Auto-unassign on shutdown**: `qn()` resets all tasks owned by a departing agent back to `pending` with `owner: undefined`, and generates a notification message suggesting reassignment
- **Idle notification system**: Automatic notifications when teammates finish their turns. Peer DM summaries included for visibility into peer collaboration. Team leader is exempt (would be notifying itself)
- **In-process teammates**: Agents can run in the same Node.js process (not just as separate processes), using "t" prefix IDs. Lifecycle includes spawn, complete, fail, and transcript extraction
- **Delegate mode**: A delegation-specific team context injection (`delegate_mode` attachment) with team name and task list path
- **Agent types**: Background system supports local bash tasks ("b" prefix), local agents ("a"), remote agents ("r"), and in-process teammates ("t") -- each with typed ID prefixes and UUID-based IDs

This is a real coordination system with race-condition protection, structured communication, and graceful degradation on agent failure.

**Beads** supports true multi-agent coordination at a different level:
- Hash-based IDs prevent creation conflicts between independent agents
- `bd update --claim` provides atomic work assignment
- Agent-as-bead enables monitoring agent health (idle, spawning, running, working, stuck, done, stopped, dead -- 8 states)
- Content hashing ensures convergent state across clones
- The "hook" pattern tracks which agent is working on what
- Git-based distribution means agents on different machines share state
- Slot fields provide exclusive access primitives

**Key difference**: Claude Code's multi-agent coordination is more sophisticated than previously captured -- it includes structured messaging (broadcast + direct), idle notifications with peer DM visibility, delegate mode, in-process teammates, and a team lead role with plan approval authority. However, it is limited to agents on the same machine sharing a filesystem. Beads coordinates agents across machines, sessions, and organizations via git. Beads also has the agent-as-bead self-monitoring pattern with 8 agent states (vs Claude Code's implicit agent lifecycle) and slot-based exclusive access.

### 5. Workflow and Quality Gates

**Claude Code Tasks** do not have a general-purpose workflow engine, but have two significant workflow-adjacent capabilities:

1. **Plan Mode**: A structured plan-approve-execute cycle via `EnterPlanMode`/`ExitPlanMode`. Plans are stored as markdown files at `~/.claude/plans/{adjective-gerund-noun}.md`. In plan mode, the agent is explicitly prohibited from making edits or running non-readonly tools -- it can only explore, design, and write the plan. In team mode, plan approval flows through the team lead via `plan_approval_request` messages, with the InboxPoller handling approval/rejection. Plan mode includes its own reminder system (injected every 5 turns, alternating between full and sparse reminders).

2. **Completion Verification Hooks (Agent Stop Hooks)**: Before a task can be marked `completed`, the system runs verification hooks (`evT()`) that spawn a separate sub-agent to evaluate whether conditions are met. The hook agent runs in non-interactive mode with up to 50 turns, and must return `{ ok: true }` or `{ ok: false, reason: "..." }`. If the hook returns a blocking error, the task completion is denied. This is functionally a quality gate -- similar in concept to Beads' gate steps, though scoped specifically to task completion rather than arbitrary workflow points.

Neither of these constitutes a workflow engine. There are no reusable templates, no formula definitions, no multi-step orchestration beyond what the agent reasons about on its own.

**Beads** has a full molecular workflow system:
- Formulas (TOML templates) define reusable multi-step workflows
- Molecules instantiate formulas as active work
- Wisps provide ephemeral local execution without sync overhead
- Gates enable async coordination at arbitrary workflow points (wait for CI, wait for human approval, timeout)
- Bonding connects work graphs across molecules

**Key difference**: Claude Code now has plan mode (a formal plan-approve-execute cycle) and completion verification hooks (quality gates for task completion). These narrow the workflow gap but do not close it. Beads has reusable workflow templates, arbitrary gate steps, ephemeral execution via wisps, and cross-graph bonding. The distinction is between Claude Code's **point capabilities** (planning, completion gates) and Beads' **workflow engine** (templates, instantiation, orchestration).

### 6. Multi-Project Management

**Claude Code Tasks** are scoped to a team name, not a project directory. While the team name can theoretically be set to anything via `CLAUDE_CODE_TASK_LIST_ID`, there is no built-in concept of multiple projects, cross-project dependencies, aggregated views, or project-aware routing.

**Beads** handles multi-project natively:
- Per-project `.beads/` directories with isolated databases
- Multi-repo hydration aggregates issues into a unified view
- Auto-routing directs new issues to the appropriate repository
- Cross-project dependencies link issues across repo boundaries
- Stealth mode enables personal tracking without team buy-in

**Key difference**: Unchanged -- Beads is project-aware; Claude Code is team-aware but not project-aware.

### 7. Cross-Session Continuity

**Claude Code Tasks** persist to disk in swarm mode, so task files exist between sessions. However, there is no explicit resume mechanism. There is no `bd ready` equivalent, no session handoff protocol, and no compaction. A new session could theoretically read `~/.claude/tasks/{team}/` files, but the system does not expose this as a user-facing workflow.

**Beads** has full cross-session support: git-persisted state, `bd ready` for computing available work, the "Landing the Plane" protocol for clean session endings, and compaction for managing long-lived backlogs.

**Key difference**: Claude Code has the persistence foundation but not the workflow built on top of it. Beads has both.

### 8. Context Management and Agent Awareness

**Claude Code Tasks** have a sophisticated **attachment system** that injects up to 29 distinct types of contextual information into the model's context on each turn. Task-related attachments include:

- `task_progress`: Running background task progress updates, re-reported every 3 turns
- `task_status`: Final status with output delta summary for completed background tasks
- `task_reminder`: Automatic reminder after 10+ turns without task tool usage, formatted as task list
- `todo_reminder`: Same for TodoWrite system
- `unified_tasks`: Combined task_status + task_progress
- `team_context`: Team membership info (agent ID, name, team name, config path, task list path) -- injected on first turn only
- `teammate_mailbox`: Messages received from other agents via SendMessage
- `plan_mode` / `plan_mode_exit`: Plan mode context, alternating full/sparse reminders every 5 turns

The reminder system uses configurable thresholds: 10 turns since last task write, 10 turns between reminders. This ensures agents do not "forget" about task management during long stretches of other work.

Additionally, Claude Code injects `changed_files`, `lsp_diagnostics`, `diagnostics`, `budget_usd`, `token_usage`, and other context types. The attachment system is a rich, proactive context management mechanism -- it does not wait for the agent to ask; it pushes relevant information automatically.

However, there is **no long-term compaction**. Completed tasks remain as JSON files indefinitely. The auto-clear behavior in TodoWrite (all completed -> `[]`) is the only cleanup mechanism, and it applies to the simpler system.

**Beads** implements semantic memory decay:
- Tier 1: 70% compression after 30 days closed
- Tier 2: 95% compression after 90 days closed
- Tombstone lifecycle for safe deletion propagation (30-day TTL with 1-hour clock skew grace)
- Wisp TTL-based compaction for ephemeral operations (6 hours to 7 days depending on category)

**Key difference**: Claude Code has rich **real-time context management** via its 29-type attachment system -- it proactively injects task progress, reminders, team context, and diagnostic information into every turn. Beads has none of this (it relies on the agent explicitly running CLI commands). However, Beads has **long-term memory management** via compaction and semantic decay, which Claude Code entirely lacks. These are complementary concerns: Claude Code manages the agent's context within a session; Beads manages the task store across months and years.

### 9. Inter-Agent Communication

**Claude Code Tasks** have a built-in inter-agent communication system:

- **SendMessage tool**: Two message types -- `broadcast` for team-wide announcements (use sparingly) and `message` for direct communication to specific teammates
- **Idle notifications**: Automatic system-generated notifications when a teammate's turn ends. Includes brief summaries of peer DMs for visibility into collaboration happening between other agents
- **Teammate mailbox**: The `teammate_mailbox` attachment delivers received messages to the agent's context
- **Team lead notifications**: The team leader receives idle notifications from all teammates, enabling orchestration awareness
- **Notification queue**: Task notifications are prioritized over regular queued commands via a dedicated `task-notification` mode

The system prompt explicitly instructs: "Your team cannot hear you if you do not use the SendMessage tool" and "Do NOT send structured JSON status messages -- just communicate in plain text."

**Beads** has no built-in inter-agent messaging. Agents communicate implicitly through shared state: creating issues, updating statuses, and closing work. The `discovered-from` dependency type provides a form of contextual linking, but there is no direct or broadcast messaging primitive.

**Key difference**: Claude Code has explicit inter-agent messaging with broadcast, direct, and automatic idle notifications. Beads relies on implicit communication via shared git state. This is a genuine Claude Code advantage for real-time multi-agent coordination.

### 10. Setup and Adoption

**Claude Code Tasks** require zero setup. The Tasks system is enabled by default in interactive mode. The agent decides when to use it. No configuration, no initialization, no external binary.

**Beads** requires installation of the `bd` binary, `bd init` in each project, optional daemon setup, and optional git hook installation. It offers progressive adoption (`--stealth` -> contributor -> full team -> federation).

**Key difference**: Unchanged -- Claude Code is zero-friction; Beads has meaningful onboarding cost.

---

## Architectural Philosophy Comparison

### Claude Code: Coordination-Rich Agent Infrastructure

The Claude Code task system in swarm mode is more than a coordination layer -- it is a **context-aware agent operating environment**. It provides:
- Persistent shared state via filesystem JSON
- Race-condition-safe task claiming via file locks
- Graceful degradation via auto-unassign on shutdown
- Background task monitoring via typed IDs and output files
- Structured inter-agent messaging (broadcast + direct)
- Automatic idle notifications for awareness of teammate availability
- Completion verification via sub-agent hooks (quality gates)
- Plan mode for structured plan-approve-execute cycles
- 29-type attachment system for proactive context injection
- Automatic task reminders when agents drift from task management

It is deliberately **session-centric** and **machine-local**. It does not attempt to be a project management system, a version-controlled issue tracker, or a workflow engine. But within its scope, it provides remarkably rich agent support infrastructure. The intelligence is in the agents; the system provides both coordination infrastructure and active context management to keep agents effective.

### Beads: Infrastructure-as-Workflow

Beads treats tasks as **persistent infrastructure**. Issues are first-class entities with rich metadata, distributed storage, and formal dependency semantics. The system embodies workflow logic:
- Dependencies ARE the execution graph
- `bd ready` computes what to do next
- Formulas encode repeatable processes
- Compaction manages long-term memory
- Git distribution enables cross-machine, cross-org coordination

But Beads has no real-time context management, no inter-agent messaging, no automatic reminders, and no proactive information injection. It provides durable state and workflow primitives; agents must actively query it. The intelligence is shared between agents and infrastructure, but the infrastructure's role is **state management**, not **context management**.

---

## Where Claude Code Falls Short of Beads

Despite having more capabilities than initially apparent, Claude Code's task system still lacks:

1. **Multi-project awareness**: No concept of projects, no aggregated cross-project views
2. **Cross-session ready computation**: No `bd ready` equivalent -- no way to ask "what work is available?"
3. **Reusable workflow templates**: No formulas, no reusable multi-step patterns (plan mode is ad-hoc, not templated)
4. **Git-native storage**: Filesystem JSON, not version-controlled, no distribution
5. **Compaction / memory decay**: No management of long-lived task backlogs
6. **Rich dependency types**: Only blocks/blockedBy, no conditional-blocks, waits-for, related, discovered-from
7. **Agent self-monitoring across sessions**: No agent-as-bead pattern, no stuck/dead detection across sessions (though in-session idle notifications partially address intra-session awareness)
8. **Federation**: No cross-org synchronization
9. **Content hashing**: No deduplication or convergent state detection
10. **Cross-machine coordination**: File-locking only works for agents sharing a filesystem

---

## Where Claude Code Matches or Exceeds Expectations

Areas where Claude Code's task system is more capable than a naive reading suggests:

1. **Persistence**: Real file-based persistence in swarm mode, plus TodoWrite disk persistence at `~/.claude/todos/`
2. **File locking**: `proper-lockfile` for atomic operations -- race-condition safe
3. **Ownership model**: Explicit owner field with auto-assign on claim
4. **Claim semantics**: Five distinct failure reasons (not_found, already_claimed, already_resolved, blocked, agent_busy)
5. **Agent busy check**: Prevents an agent from hoarding multiple tasks
6. **Graceful shutdown**: Auto-unassign with notification when an agent terminates
7. **Highwatermark**: IDs are globally unique within a team (never reused, even after deletion)
8. **Dependency enforcement**: blockedBy checked at claim time, cleaned up on deletion
9. **Background task system**: Four typed agent types (bash, agent, remote, in-process teammate) with output files and incremental reads
10. **Zero setup**: All of this works out of the box, no configuration needed
11. **Completion verification hooks**: Sub-agents spawned to validate task completion -- a quality gate mechanism
12. **Plan mode**: Structured plan-approve-execute workflow with team-lead approval and plan file persistence at `~/.claude/plans/`
13. **Inter-agent messaging**: SendMessage with broadcast + direct modes, automatic idle notifications with peer DM visibility
14. **Task reminder system**: Automatic nudges after 10+ idle turns -- agents are proactively reminded to manage tasks
15. **29-type attachment system**: Rich, proactive context injection including task progress, team context, teammate mailbox, diagnostics, and more
16. **In-process teammates**: Agents sharing the same Node.js process for lower-overhead coordination
17. **Delegate mode**: Delegation-specific team context for structured work delegation

---

## When to Use Which

### Use Claude Code Tasks When:
- Working with a team of agents on a single machine
- Everything fits within one or a few related sessions
- You need zero-setup coordination with file locking, ownership, and messaging
- Single project, potentially multiple agents
- You want rich agent context management (reminders, progress tracking, idle notifications)
- You need a plan-approve-execute workflow for implementation planning
- You want the simplest possible infrastructure

### Use Beads When:
- Managing ongoing work across multiple projects and repositories
- Agents operate on different machines and need shared state via git
- Work spans many sessions over days or weeks
- You need repeatable workflows (releases, reviews, deployments)
- Cross-project dependencies exist
- You want persistent history, version control, and traceability
- Long-lived backlogs require compaction

### Use Both Together:
- Beads manages the persistent backlog, multi-project view, and cross-session continuity
- Claude Code's swarm-mode Tasks manages in-session execution, agent coordination, and context management
- An agent reads from `bd ready`, claims a Beads task, uses Claude Code's task system to coordinate subtask execution among team agents (with SendMessage, plan mode, and completion hooks), then closes the Beads issue when done
- Beads provides the durable state layer; Claude Code provides the real-time agent awareness layer

---

## Summary Table

| Concern | Claude Code (Swarm Mode) | Beads | Winner |
|---------|-------------------------|-------|--------|
| Zero-friction start | Built-in, default on | Requires setup | Claude Code |
| Single-session coordination | Excellent (file locking, ownership, busy checks) | Not designed for this | Claude Code |
| Inter-agent messaging | Excellent (broadcast, direct, idle notifications) | None (implicit via shared state) | Claude Code |
| Real-time context management | Excellent (29 attachment types, reminders, progress) | None (agents query CLI) | Claude Code |
| Quality gates | Yes (completion verification hooks, sub-agents) | Yes (gate steps in molecules) | Comparable (different scope) |
| Planning workflow | Yes (plan mode with team-lead approval) | No formal mode | Claude Code |
| Persistence | Yes (filesystem JSON) | Yes (SQLite + JSONL in git) | Beads (git > filesystem) |
| Multi-session continuity | Partial (files persist, no resume workflow) | Excellent (ready, handoff, sync) | Beads |
| Multi-project view | None | Native | Beads |
| Multi-agent (same machine) | Excellent (locking, claiming, auto-unassign, messaging) | Not specifically optimized for this | Claude Code |
| Multi-agent (cross-machine) | None | Excellent (git-based) | Beads |
| Workflow automation | Partial (plan mode, completion hooks) | Full (Molecules + Formulas + Gates) | Beads |
| Dependency richness | Basic (2 types, enforced) | Rich (6 types, computed readiness) | Beads |
| Long-term context management | None (no compaction) | Compaction + decay | Beads |
| Data model simplicity | Simple (10 fields + metadata) | Complex but comprehensive (40+ fields) | Depends on needs |
| Git integration | None | Native | Beads |
| Federation | None | Peer-to-peer | Beads |
| Agent self-monitoring | Partial (idle notifications, in-session only) | Full (agent-as-bead, 8 states, cross-session) | Beads |
| Learning curve | Zero | Moderate | Claude Code |
| ID collision safety | Sequential + highwatermark (safe within team) | Hash-based (safe across organizations) | Beads |
