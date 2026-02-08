# Task System Comparison: Claude Code vs Beads vs Temporal

## Overview

This document compares three approaches to AI-agent task management, each operating at a different architectural layer:

- **Claude Code Tasks**: Claude Code v2.1.34 has **two mutually exclusive** task systems -- TodoWrite (solo/in-memory) and Tasks (team/swarm/persisted-to-disk) -- with a built-in team coordination layer. Operates at the **execution layer**: how an agent organizes work within a session.
- **Beads**: A persistent, git-native, multi-project issue tracker designed for multi-agent workflows. Operates at the **planning layer**: what work exists across projects and sessions.
- **Temporal**: A durable execution platform (temporal.io) that guarantees workflows run to completion through crash recovery and deterministic replay. Operates at the **orchestration layer**: ensuring work actually completes despite failures. See `temporal-analysis.md`.

These are not interchangeable alternatives. They are complementary layers that address different concerns. Within a single session, Claude Code's swarm-mode Tasks provides file-based persistence, dependency tracking, multi-agent ownership, atomic claiming with file locking, and agent busy checks. Beads provides cross-session persistence, multi-project views, and workflow templates via git. Temporal provides crash recovery, deterministic execution guarantees, and vendor-independent agent dispatch. The question is not "which one?" but "which layers does your situation require?"

---

## Architectural Layers

These three systems occupy distinct positions in the AI agent infrastructure stack:

```
┌─────────────────────────────────────────────────────────────┐
│                   ORCHESTRATION LAYER                        │
│                      (Temporal)                              │
│                                                              │
│  Guarantees: Crash recovery, deterministic execution,        │
│  vendor-independent agent dispatch, durable event history    │
│                                                              │
│  Owns: "Did this work actually complete?"                    │
├─────────────────────────────────────────────────────────────┤
│                    PLANNING LAYER                             │
│                       (Beads)                                │
│                                                              │
│  Guarantees: Persistent backlog, cross-session continuity,   │
│  multi-project views, ready-work computation, compaction     │
│                                                              │
│  Owns: "What work exists and what is ready?"                 │
├─────────────────────────────────────────────────────────────┤
│                   EXECUTION LAYER                            │
│                  (Claude Code Tasks)                          │
│                                                              │
│  Guarantees: In-session coordination, file locking,          │
│  atomic claiming, dependency enforcement, zero setup         │
│                                                              │
│  Owns: "How does the agent organize work right now?"         │
└─────────────────────────────────────────────────────────────┘
```

**Key insight**: Using all three is not redundant. A Temporal Workflow reads from Beads (`bd ready`) to find work, dispatches an Activity that invokes Claude Code, and Claude's internal Tasks help the agent decompose the work into subtasks. Each layer operates at its natural level of abstraction.

Not every situation requires all three layers. A solo developer on one project needs only the execution layer. A team managing multiple projects across sessions needs the planning layer. A production system with crash recovery and multi-agent routing requirements needs the orchestration layer.

---

## Architecture Note: Claude Code's Two Systems

Claude Code does not have one task system. It has two, controlled by a feature gate (`CLAUDE_CODE_ENABLE_TASKS`):

| System | Tool Names | Storage | Scope | Enabled When |
|--------|-----------|---------|-------|-------------|
| **TodoWrite** | `TodoWrite` | In-memory (`AppState.todos`) | Per-agent, per-session | Tasks feature **disabled** (solo mode) |
| **Tasks** | `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList` | Filesystem JSON at `~/.claude/tasks/{team-name}/` | Shared across team agents | Tasks feature **enabled** (team/swarm mode) |

These are **mutually exclusive** -- `isEnabled()` returns opposite values for each. In interactive mode, the Tasks system is enabled by default, and TodoWrite is disabled.

The Tasks system also includes team management tools (`TeamCreate`, `TeamDelete`, `SendMessage`) and background task infrastructure (`TaskOutput`) for local agents, remote agents, and in-process teammates.

This comparison focuses on the **Tasks system** (swarm mode) since it represents Claude Code's full capabilities.

---

## Side-by-Side Comparison

| Dimension | Claude Code Tasks (Swarm Mode) | Beads | Temporal |
|-----------|-------------------------------|-------|----------|
| **Primary function** | In-session agent coordination | Persistent issue tracking | Workflow execution engine |
| **Architectural layer** | Execution | Planning | Orchestration |
| **Scope** | Single machine, team of agents in one session | Multi-project, multi-agent, multi-org | Multi-machine, multi-agent, multi-provider |
| **Persistence** | **File-level only** -- JSON files on disk, but scoped to random UUIDs in solo mode; undiscoverable after session ends | Yes -- SQLite + JSONL in git | Yes -- event history in database (Postgres/MySQL/Cassandra) |
| **Storage** | Individual JSON files per task + file locking | SQLite + JSONL + optional Dolt | Event history in Temporal Server database |
| **Data Model** | ~10 fields (id, subject, description, activeForm, status, owner, blocks, blockedBy, metadata) | ~40+ fields (full issue tracker) | Workflows + Activities + event log (code-defined, not schema-fixed) |
| **ID System** | Auto-incrementing integers ("1", "2", "3") with highwatermark | Hash-based (`bd-a1b2`) | User-chosen Workflow IDs (e.g., `fix-auth-bug-2024`) |
| **States** | 3 + delete: pending, in_progress, completed, (deleted via TaskUpdate) | 7+: open, in_progress, blocked, deferred, closed, hooked, tombstone | Running, Completed, Failed, Cancelled, Terminated, TimedOut, ContinuedAsNew |
| **Dependencies** | blocks/blockedBy (2 types, enforced at claim time) | 6 types: blocks, parent-child, conditional-blocks, waits-for, related, discovered-from | Child Workflows, Signals, Sagas (code-defined) |
| **Multi-project** | No | Yes (multi-repo hydration, cross-project deps) | Yes (Namespaces + Nexus for cross-namespace) |
| **Multi-agent** | **Yes** -- team system with ownership, atomic claiming, file locking, agent busy checks, auto-unassign on shutdown | Yes -- hash IDs, atomic claims, agent-as-bead, hook pattern | Yes -- Activity routing via Task Queues, any agent as Activity |
| **Workflow Engine** | No | Yes (molecules, formulas, gates) | Yes (Workflows are code, deterministic replay) |
| **Cross-session** | **Effectively none** without manual intervention (`--resume` or `CLAUDE_CODE_TASK_LIST_ID` env var) | Yes (git-persisted, handoff protocols, `bd ready`) | No sessions -- Workflows run to completion (minutes to years) |
| **Crash recovery** | **None** -- tasks stuck in `in_progress` permanently; no cleanup, no discovery | Yes (git-persisted state survives any crash; `bd ready` recomputes) | **Automatic** -- replay from event history, retry Activities on new Workers |
| **Compaction** | None | Semantic memory decay (tier 1/2 compression) | Continue-As-New (manual, for long-running Workflows) |
| **Federation** | None | Yes (peer-to-peer via Dolt) | Nexus (cross-namespace, cross-cluster) |
| **Git integration** | None -- filesystem only | Native (JSONL tracked in git, daemon syncs) | None (database-backed) |
| **UI** | Terminal status line with spinners, expanded tasks view | CLI output + `--json` for agents | Web UI (Workflow history, state, event timeline) |
| **Integration** | Built into Claude Code agent loop | CLI, MCP server, Claude Code plugin | SDKs (Python, Go, TypeScript, Java, .NET, etc.) |
| **Language** | TypeScript (Bun-compiled binary) | Go | Server: Go; SDKs: polyglot |
| **Setup** | Zero (built-in, default enabled in interactive mode) | `bd init` per project | Server deployment + Worker processes |
| **Vendor independence** | None (Claude Code-specific) | Partial (Beads-specific, but CLI-accessible) | Full (Activities wrap any CLI agent) |

---

## Detailed Comparison by Category

### 1. Persistence and Lifecycle

**Claude Code Tasks** have **file persistence** but not **practical persistence**. Each task is written as an individual JSON file at `~/.claude/tasks/{sanitized-list-id}/{id}.json`. The system uses `proper-lockfile` for concurrent access, maintains a `.highwatermark` file to ensure monotonically increasing IDs even after deletions, and supports a full CRUD lifecycle (create, read, update, delete, list, clear-all). Task files survive across process restarts.

However, **the task list ID is scoped to a random session UUID in solo mode**. The `nW()` function resolves the list ID through a priority chain: `CLAUDE_CODE_TASK_LIST_ID` env var -> teammate context -> team name -> manual name -> `bR()` (random UUIDv7). In solo sessions (the common case), this chain falls through to `bR()`, creating an isolated namespace like `~/.claude/tasks/019508a2-3f4e-...`. The next session generates a fresh UUID and has no way to discover the previous one.

This creates a critical distinction: **files persist, but they are undiscoverable**. Without knowing the exact UUID, there is no way to reconnect to a previous session's tasks. Three manual recovery paths exist:
- **`--resume <sessionId>`**: Loads a previous session's transcript (whether it preserves the task list ID is unconfirmed from binary analysis)
- **`--continue`**: Resumes the most recent session
- **`CLAUDE_CODE_TASK_LIST_ID=<uuid>`**: Environment variable override that guarantees reconnection to a specific task directory

In contrast, the TodoWrite system (solo mode, separate from Tasks) is purely in-memory and vanishes when the session ends.

**Beads** is fundamentally persistent and project-scoped. Issues are stored in SQLite for fast queries and exported to JSONL tracked in git. They survive across sessions, across machines, across teams. The daemon handles background sync, and the "Landing the Plane" protocol ensures agents push state before ending sessions. Compaction manages the long tail of old issues. Crucially, Beads issues are **discoverable** -- `bd ready` computes available work across all projects without requiring any session context.

**Temporal** persists state as an append-only event history in its server database. Every Workflow execution is a sequence of events: `WorkflowExecutionStarted`, `ActivityTaskScheduled`, `ActivityTaskCompleted`, etc. This history is the canonical state -- the current Workflow state is derived by replaying the history through deterministic Workflow code. There are no sessions to expire, no UUIDs to lose. A Workflow starts and runs until it explicitly completes, fails, times out, or is cancelled. Workflow IDs are user-chosen strings (e.g., `fix-auth-bug-2024`), stable and queryable at any time.

**Key difference**: Claude Code has file persistence but lacks practical persistence (discoverability, resume workflow, crash recovery). Beads has both file persistence and practical persistence via git and ready computation. Temporal has the strongest persistence model -- event-sourced state that survives any failure, with no concept of sessions that can expire.

### 2. Data Model Complexity

**Claude Code Tasks** have a focused model: id, subject, description, activeForm, status, owner, blocks, blockedBy, metadata. The metadata field is an arbitrary `Record<string, unknown>` with a special `_internal` key that hides tasks from listing. This is lean but sufficient for coordinating work among a team of agents.

**Beads** has 40+ fields covering issue content, assignment, timestamps, external integration, compaction metadata, tombstone fields, agent identity, molecule/workflow fields, gate fields, slot fields, and event fields. This richness supports the full range of project management scenarios, from simple task tracking to federated multi-org workflows.

**Temporal** does not have a fixed data model for tasks. Workflows and Activities are code -- you define whatever data structures your domain requires. The "data model" is the event history: a sequence of typed events with serialized payloads. This is infinitely flexible but requires writing code to define your domain model, rather than using a pre-built schema.

**Key difference**: Claude Code's model is minimal but extensible via metadata. Beads' model is comprehensive and purpose-built. Temporal's model is code-defined and unlimited but requires development effort.

### 3. Dependency System

**Claude Code Tasks** support `blocks`/`blockedBy` relationships with **enforcement at claim time**. When an agent attempts to claim a task via `oBA()`, the system checks for active blockers -- if any blockedBy task is still non-completed, the claim returns `{ success: false, reason: "blocked" }`. Dependencies are also cleaned up on deletion (references to the deleted task are removed from all other tasks' blocks/blockedBy arrays). TaskList filters completed tasks from blockedBy in its display output so agents see only active blockers.

This is more than advisory -- it is soft enforcement. The agent can still set status to in_progress via TaskUpdate without going through the claim path, but the claim mechanism provides a formal gate.

**Beads** has 6 dependency types with different semantics:
- `blocks` / `parent-child` / `conditional-blocks` / `waits-for` -- affect readiness computation
- `related` / `discovered-from` -- informational only

Beads materializes a `blocked_issues_cache` in SQLite for O(1) readiness checks and provides `bd ready` to compute available work. Dependencies form the backbone of the workflow engine.

**Temporal** expresses dependencies through code rather than declared relationships. Child Workflows model parent-child dependencies. `workflow.wait_condition()` implements waits-for. Sagas implement conditional compensation. Signals implement async event dependencies. These are more powerful than declared dependency types (arbitrary logic, not just predefined relationship types) but require writing code rather than setting a field.

**Key difference**: Claude Code has one dependency type with enforcement at claim time. Beads has six declared dependency types with computed readiness. Temporal has code-defined dependencies with execution semantics (not just tracking).

### 4. Multi-Agent Coordination

**Claude Code Tasks** have a genuine multi-agent coordination system in swarm mode:

- **Team management**: `TeamCreate`, `TeamDelete`, `SendMessage` tools for team lifecycle
- **Ownership**: Tasks have an `owner` field; auto-assigned to the claiming agent's name in swarm mode
- **Atomic claiming**: `oBA()` acquires a file lock on the individual task file before checking and updating ownership
- **Agent busy check**: `SyD()` acquires a global lock and verifies the agent does not already own another active task before allowing a claim
- **Claim failure reasons**: Structured responses for `task_not_found`, `already_claimed`, `already_resolved`, `blocked`, `agent_busy`
- **Auto-unassign on shutdown**: `qn()` resets all tasks owned by a departing agent back to `pending` with `owner: undefined`, and generates a notification message suggesting reassignment
- **Agent types**: Background system supports local bash tasks, local agents, remote agents, and in-process teammates -- each with typed ID prefixes

This is a real coordination system with race-condition protection and graceful degradation on agent failure.

**Beads** supports true multi-agent coordination at a different level:
- Hash-based IDs prevent creation conflicts between independent agents
- `bd update --claim` provides atomic work assignment
- Agent-as-bead enables monitoring agent health (idle, running, stuck, dead)
- Content hashing ensures convergent state across clones
- The "hook" pattern tracks which agent is working on what
- Git-based distribution means agents on different machines share state

**Temporal** coordinates agents through Task Queues and Activity routing. Different agents can run as different Activity implementations, dispatched to appropriate Task Queues. A Workflow can use Claude for complex reasoning, Codex for boilerplate, and a local model for quick tasks -- all within the same Workflow, routed by Task Queue to Workers with the right capabilities. Temporal's coordination is vendor-independent: an agent is just an Activity that invokes a CLI command. Switching agents means changing the command.

**Key difference**: Claude Code's multi-agent coordination is real and robust, but limited to agents on the same machine sharing a filesystem. Beads coordinates agents across machines, sessions, and organizations via git. Temporal coordinates agents across machines and providers via Task Queues, with the additional guarantee of crash recovery and automatic retry. Claude Code also lacks the agent-as-bead self-monitoring pattern that Beads provides.

### 5. Workflow Capabilities

**Claude Code Tasks** have no workflow engine. The agent creates tasks with dependencies and executes them in order using its own reasoning. There are no templates, no reusable patterns, no gate steps.

**Beads** has a full molecular workflow system:
- Formulas (TOML templates) define reusable multi-step workflows
- Molecules instantiate formulas as active work
- Wisps provide ephemeral local execution without sync overhead
- Gates enable async coordination (wait for CI, wait for human approval)
- Bonding connects work graphs across molecules

**Temporal** is fundamentally a workflow engine. Workflows are code -- versioned, tested, reviewed through normal software development processes. They support:
- Sequential and parallel step execution
- Conditional branching based on Activity results
- Iteration with automatic crash recovery
- Child Workflows for composition
- Signals for external events (CI completion, human approval)
- Sagas for compensation/rollback
- Schedules for recurring execution

**Key difference**: Claude Code has no workflow engine. Beads has a template-based workflow engine (formulas). Temporal is a code-based workflow engine with the strongest execution guarantees (deterministic replay, crash recovery, automatic retry).

### 6. Multi-Project Management

**Claude Code Tasks** are scoped to a team name, not a project directory. While the team name can theoretically be set to anything via `CLAUDE_CODE_TASK_LIST_ID`, there is no built-in concept of multiple projects, cross-project dependencies, aggregated views, or project-aware routing.

**Beads** handles multi-project natively:
- Per-project `.beads/` directories with isolated databases
- Multi-repo hydration aggregates issues into a unified view
- Auto-routing directs new issues to the appropriate repository
- Cross-project dependencies link issues across repo boundaries
- Stealth mode enables personal tracking without team buy-in

**Temporal** supports multi-project through Namespaces -- logically isolated execution environments with their own Workflow histories, Task Queues, and retention policies. Namespaces can be created programmatically and provide the project isolation that Claude Code lacks. Temporal Nexus enables cross-namespace coordination (Workflows in one project calling operations in another).

**Key difference**: Claude Code is team-aware but not project-aware. Beads is project-aware with git-native multi-repo support. Temporal is project-aware through Namespaces, with cross-project coordination via Nexus.

### 7. Cross-Session Continuity

**Claude Code Tasks** persist to disk as JSON files, but cross-session continuity is **effectively broken** for solo users. The root cause: task lists are scoped to random UUIDs generated per session (`bR()`). A new session generates a fresh UUID, creates an empty task namespace, and has no way to discover previous sessions' tasks.

Empirical analysis of `~/.claude/tasks/` reveals 15 UUID-named directories. Cross-referencing against `sessions-index.json` (which records session metadata for 2,508 sessions across 12 projects), only 5 of the 15 are traceable to known sessions. The remaining 10 are orphaned -- and all 10 are empty.

Three manual recovery paths exist, each with significant limitations:

1. **`--resume <sessionId>`**: Loads a previous session's transcript. Requires knowing the session ID. Whether it preserves the original task list ID is unconfirmed from binary analysis. No UI for browsing past sessions.
2. **`--continue`**: Resumes the most recent session. Convenient but limited to exactly one session. Same uncertainty about task list reconnection.
3. **`CLAUDE_CODE_TASK_LIST_ID=<uuid>`**: Environment variable that overrides the task list ID. This is the guaranteed escape hatch -- `CLAUDE_CODE_TASK_LIST_ID=<uuid> claude --resume <sessionId>` will reconnect to any task directory. But it requires the user to manually track and store UUIDs.

**What does NOT exist**: There is no `/resume` slash command, no `TaskDiscover` tool, no CLI task browser, no way to list task directories with their contents, and no way to map a project to its associated task lists.

**Crash behavior** further compounds the problem. If a session crashes, tasks in `in_progress` status remain stuck indefinitely. The `qn()` unassign-on-shutdown function only runs during graceful shutdowns. There is no crash recovery, no orphan detection, and no cleanup mechanism.

**Beads** has full cross-session support: git-persisted state, `bd ready` for computing available work, the "Landing the Plane" protocol for clean session endings, and compaction for managing long-lived backlogs. Git persistence means Beads issues survive any crash -- the state is always recoverable.

**Temporal** eliminates the cross-session problem entirely. There are no sessions. A Workflow Execution starts and runs until it completes -- this can take milliseconds or years. The Workflow ID is a user-chosen stable string, queryable at any time. If a Worker crashes, another Worker picks up the Workflow via replay. There are no UUIDs to track, no `--resume` flags, no env var overrides. The concept of "session loss" does not exist in Temporal's model.

**Key difference**: Claude Code has file persistence but not practical persistence -- files survive but are undiscoverable without manual intervention. Beads has both file persistence and practical persistence, with automatic discoverability and crash resilience. Temporal has no sessions at all -- Workflows are durable by default, with automatic crash recovery.

### 8. Context Management

**Claude Code Tasks** have no compaction or memory management. In solo mode (TodoWrite), the entire list is cleared when all items are completed. In swarm mode (Tasks), completed tasks remain as JSON files indefinitely.

**Beads** implements semantic memory decay:
- Tier 1: 70% compression after 30 days closed
- Tier 2: 95% compression after 90 days closed
- Tombstone lifecycle for safe deletion propagation
- Wisp TTL-based compaction for ephemeral operations

**Temporal** has a different approach to long-running state: Continue-As-New. When a Workflow's event history approaches the ~51K event limit, the Workflow can "continue as new" -- complete the current execution and start a fresh one with carried-over state, resetting the event history. This is a manual mechanism (the Workflow author must implement it) rather than automatic compaction.

**Key difference**: Claude Code does not manage long-term context. Beads has automatic semantic compaction. Temporal has manual history management via Continue-As-New.

### 9. Setup and Adoption

**Claude Code Tasks** require zero setup. The Tasks system is enabled by default in interactive mode. The agent decides when to use it. No configuration, no initialization, no external binary.

**Beads** requires installation of the `bd` binary, `bd init` in each project, optional daemon setup, and optional git hook installation. It offers progressive adoption (`--stealth` -> contributor -> full team -> federation).

**Temporal** has the highest setup cost. Development requires the Temporal CLI (`temporal server start-dev`). Production requires a server cluster (Docker Compose, Kubernetes, or Temporal Cloud at ~$100/month+) plus Worker processes that you deploy and manage. The learning curve is steep: determinism constraints, event sourcing, Activity design, and Workflow versioning all require conceptual ramp-up.

**Key difference**: Claude Code is zero-friction. Beads has meaningful but manageable onboarding. Temporal has significant infrastructure and learning overhead, justified only when its guarantees (crash recovery, determinism, vendor independence) are needed.

---

## Architectural Philosophy Comparison

### Claude Code: Coordination-Capable Agent Infrastructure

The Claude Code task system in swarm mode is more than a thinking aid -- it is a coordination layer for a team of agents on a single machine. It provides:
- Persistent shared state via filesystem JSON
- Race-condition-safe task claiming via file locks
- Graceful degradation via auto-unassign on shutdown
- Background task monitoring via typed IDs and output files

But it is deliberately **session-centric** and **machine-local** -- and session-centricity implies **session-fragility**. Task lists are scoped to session UUIDs, so when a session ends (or crashes), the tasks become orphaned. The system provides just enough infrastructure for agents to coordinate within a single session, but does not attempt to bridge sessions, projects, or machines.

### Beads: Infrastructure-as-Workflow

Beads treats tasks as **persistent infrastructure**. Issues are first-class entities with rich metadata, distributed storage, and formal dependency semantics. The system embodies workflow logic:
- Dependencies ARE the execution graph
- `bd ready` computes what to do next
- Formulas encode repeatable processes
- Compaction manages long-term memory
- Git distribution enables cross-machine, cross-org coordination

The intelligence is shared between agents and infrastructure.

### Temporal: Deterministic Orchestration, Materialized

`orchestration-principle.md` argues that orchestration belongs in a deterministic system, not in agent internals. Temporal is the concrete, production-proven implementation of that argument. It answers the question that neither Claude Code Tasks nor Beads addresses: **who guarantees the workflow completes?**

- Workflows are deterministic code -- they do not forget instructions, skip steps, or hallucinate state
- Agents are Activities -- they receive scoped work and return results
- Crash recovery is automatic -- replay from event history, retry on available Workers
- Vendor independence is architectural -- switching agents means changing an Activity implementation

The "missing layer" described in previous versions of this document is no longer abstract. It is Temporal (or systems like it). The key tradeoff is that materializing this layer requires real infrastructure: a server, a database, Workers, and conceptual investment in the determinism model.

### Three Layers, Not Three Alternatives

The architectural insight is that these systems are not competing solutions to the same problem. They are solutions to different problems at different layers:

| Layer | System | Question Answered | Without This Layer |
|-------|--------|-------------------|-------------------|
| Orchestration | Temporal | "Will this work complete despite failures?" | Crashes lose progress; no automatic retry |
| Planning | Beads | "What work exists across projects?" | No cross-session backlog; no multi-project view |
| Execution | Claude Code Tasks | "How does the agent organize work now?" | Agent cannot decompose and track subtasks |

You can use any subset of these layers. Most solo developers need only the execution layer. Teams managing backlogs across projects need the planning layer. Production systems requiring reliability need the orchestration layer.

---

## Where Claude Code Falls Short

Despite having more capabilities than initially apparent, Claude Code's task system lacks capabilities provided by both Beads and Temporal:

1. **Task discoverability / crash recovery**: Task lists are scoped to random session UUIDs; no way to browse, search, or reconnect to previous sessions' tasks without manually tracking UUIDs. Crashed sessions leave tasks stuck in `in_progress` permanently. *(Beads solves with git + `bd ready`; Temporal solves with durable Workflows)*
2. **Multi-project awareness**: No concept of projects, no aggregated cross-project views. *(Beads solves with multi-repo hydration; Temporal solves with Namespaces)*
3. **Cross-session ready computation**: No `bd ready` equivalent -- no way to ask "what work is available?" *(Beads solves directly; Temporal solves by eliminating sessions)*
4. **Workflow templates**: No formulas, no reusable patterns, no gate steps. *(Beads solves with formulas; Temporal solves with Workflow code)*
5. **Git-native storage**: Filesystem JSON, not version-controlled, no distribution. *(Beads solves with JSONL in git)*
6. **Compaction / memory decay**: No management of long-lived task backlogs. *(Beads solves with semantic compression)*
7. **Rich dependency types**: Only blocks/blockedBy, no conditional-blocks, waits-for, related, discovered-from. *(Beads solves with 6 types; Temporal solves with code-defined dependencies)*
8. **Agent self-monitoring**: No agent-as-bead pattern, no stuck/dead detection. *(Beads solves with agent-as-bead)*
9. **Federation**: No cross-org synchronization. *(Beads solves with Dolt; Temporal solves with Nexus)*
10. **Content hashing**: No deduplication or convergent state detection. *(Beads solves directly)*
11. **Cross-machine coordination**: File-locking only works for agents sharing a filesystem. *(Beads solves with git; Temporal solves with Task Queues)*
12. **Deterministic execution guarantees**: Agent decides workflow progression; no replay, no retry. *(Temporal solves with deterministic Workflows)*
13. **Vendor independence**: Entirely Claude Code-specific. *(Temporal solves with Activity abstraction)*

---

## Where Claude Code Matches or Exceeds Expectations

Areas where Claude Code's task system is more capable than a naive reading suggests:

1. **File persistence**: Real file-based persistence in swarm mode (not ephemeral), but undermined by session-scoped UUIDs that make tasks undiscoverable after the session ends
2. **File locking**: `proper-lockfile` for atomic operations -- race-condition safe
3. **Ownership model**: Explicit owner field with auto-assign on claim
4. **Claim semantics**: Five distinct failure reasons (not_found, already_claimed, already_resolved, blocked, agent_busy)
5. **Agent busy check**: Prevents an agent from hoarding multiple tasks
6. **Graceful shutdown**: Auto-unassign with notification when an agent terminates
7. **Highwatermark**: IDs are globally unique within a team (never reused, even after deletion)
8. **Dependency enforcement**: blockedBy checked at claim time, cleaned up on deletion
9. **Background task system**: Typed IDs, output files, incremental read with offset
10. **Zero setup**: All of this works out of the box, no configuration needed -- neither Beads nor Temporal can match this

---

## When to Use Which

### Use Claude Code Tasks When:
- Working with a team of agents on a single machine
- Everything fits within one or a few related sessions
- You need zero-setup coordination with file locking and ownership
- Single project, potentially multiple agents
- You want the simplest possible infrastructure

### Use Beads When:
- Managing ongoing work across multiple projects and repositories
- Agents operate on different machines and need shared state via git
- Work spans many sessions over days or weeks
- You need repeatable workflows (releases, reviews, deployments)
- Cross-project dependencies exist
- You want persistent history, version control, and traceability
- Long-lived backlogs require compaction

### Use Temporal When:
- Crash recovery is non-negotiable -- workflows must not lose progress on failure
- You orchestrate multiple agents, potentially from different providers (Claude, Codex, open source)
- Vendor independence matters -- you want to switch AI providers without rewriting orchestration logic
- Workflows span machines -- agents run on different servers with centralized coordination
- Workflows are long-running -- multi-hour or multi-day processes that survive restarts
- You need auditability -- a complete, immutable record of every step taken

### Use Beads + Claude Code Tasks Together:
- Beads manages the persistent backlog, multi-project view, and cross-session continuity
- Claude Code's swarm-mode Tasks manages in-session execution and agent coordination
- An agent reads from `bd ready`, claims a Beads task, uses Claude Code's task system to coordinate subtask execution among team agents, then closes the Beads issue when done

### Use Temporal + Claude Code Tasks Together:
- Temporal handles the outer orchestration loop (what to work on, crash recovery, multi-agent routing)
- Claude Code's Tasks handle the inner loop (agent self-organization during a session)
- A Temporal Workflow dispatches Activities that invoke `claude --print`, and Claude uses TaskCreate internally for subtask decomposition

### Use All Three Together:
- Beads is the planning layer -- persistent backlog, multi-project views, ready computation
- Temporal is the orchestration layer -- crash recovery, deterministic execution, agent routing
- Claude Code Tasks is the execution layer -- in-session agent self-organization
- A Temporal Workflow reads from `bd ready`, dispatches an Activity that runs Claude, and Claude uses its internal Tasks for subtask decomposition. Temporal guarantees completion; Beads tracks the backlog; Claude does the work.

### When to Use Neither Beads nor Temporal

If you are a solo developer on a single project with short sessions, Claude Code's built-in tasks are sufficient. The infrastructure overhead of Beads or Temporal is not justified when sessions are short enough that re-doing work is trivial.

---

## Summary Table

| Concern | Claude Code (Swarm Mode) | Beads | Temporal | Best Fit |
|---------|-------------------------|-------|----------|----------|
| Zero-friction start | Built-in, default on | Requires setup | Requires infrastructure | Claude Code |
| Single-session coordination | Excellent (file locking, ownership, busy checks) | Not designed for this | Not designed for this | Claude Code |
| Persistence (file-level) | Yes (filesystem JSON) | Yes (SQLite + JSONL in git) | Yes (event history in database) | Temporal (strongest guarantees) |
| Persistence (practical / discoverable) | No -- session-scoped UUIDs, undiscoverable | Yes -- `bd ready`, git history | Yes -- queryable Workflow IDs, no sessions | Temporal / Beads |
| Multi-session continuity | Effectively none without `--resume` or env var | Excellent (ready, handoff, sync) | No sessions -- Workflows are durable | Temporal |
| Crash recovery | None (tasks stuck in `in_progress` permanently) | State survives (git); no auto-retry | Automatic (replay + retry) | Temporal |
| Multi-project view | None | Native (multi-repo hydration) | Namespaces + Nexus | Beads (richer model) |
| Multi-agent (same machine) | Excellent (locking, claiming, auto-unassign) | Not specifically optimized for this | Overkill for same-machine | Claude Code |
| Multi-agent (cross-machine) | None | Excellent (git-based) | Excellent (Task Queue routing) | Temporal (stronger guarantees) / Beads (simpler) |
| Multi-agent (cross-provider) | None (Claude-specific) | Partial (CLI-accessible) | Full (Activities wrap any agent) | Temporal |
| Workflow automation | None | Molecules + Formulas + Gates | Workflow code + replay + retry | Temporal (execution) / Beads (templates) |
| Dependency richness | Basic (2 types, enforced) | Rich (6 types, computed readiness) | Code-defined (unlimited) | Beads (declarative) / Temporal (flexible) |
| Context management | None | Compaction + decay | Continue-As-New (manual) | Beads |
| Data model simplicity | Simple (10 fields + metadata) | Complex but comprehensive (40+ fields) | Code-defined (flexible) | Depends on needs |
| Git integration | None | Native | None | Beads |
| Federation | None | Peer-to-peer (Dolt) | Nexus (cross-namespace) | Beads (richer) |
| Agent self-monitoring | None | Agent-as-bead | Worker metrics (operational) | Beads |
| Learning curve | Zero | Moderate (chemistry metaphors) | Steep (determinism, event sourcing) | Claude Code |
| Infrastructure cost | None | Git (existing) | Server + database or Cloud ($100+/mo) | Claude Code |
| Determinism guarantees | None (agent-driven) | None (agent-driven) | Enforced (replay-safe Workflow code) | Temporal |
| Vendor independence | No (Claude-specific internals) | Partial (Beads-specific) | Full (Activities wrap any CLI) | Temporal |
| Setup cost | None | Low (~hours) | Medium-High (~days) | Claude Code |
| Auditability | None | Git history | Complete event history | Temporal |

---

*This comparison is part of the task-analysis project. See `temporal-analysis.md` for the standalone Temporal analysis, `orchestration-principle.md` for the underlying principle, `beads-task-system.md` for the Beads deep-dive, and `next-step.md` for practical next steps.*
