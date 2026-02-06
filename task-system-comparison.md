# Task System Comparison: Claude Code vs Beads

## Overview

This document compares two approaches to AI-agent task management:

- **Claude Code Tasks**: Claude Code v2.1.34 has **two mutually exclusive** task systems -- TodoWrite (solo/in-memory) and Tasks (team/swarm/persisted-to-disk) -- with a built-in team coordination layer
- **Beads**: A persistent, git-native, multi-project issue tracker designed for multi-agent workflows

The gap between these systems is narrower than a surface-level inspection suggests. Claude Code's swarm-mode Tasks system already provides file-based persistence, dependency tracking, multi-agent ownership, atomic claiming with file locking, and agent busy checks. However, Beads still operates at a fundamentally different scale in several dimensions.

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
| **Workflow Engine** | No | Yes (molecules, formulas, gates) |
| **Cross-session** | **Partial** -- task files persist on disk, but no explicit resume protocol or ready computation | Yes (git-persisted, handoff protocols, `bd ready`) |
| **Compaction** | None | Semantic memory decay (tier 1/2 compression) |
| **Federation** | None | Yes (peer-to-peer via Dolt) |
| **Git integration** | None -- filesystem only | Native (JSONL tracked in git, daemon syncs) |
| **UI** | Terminal status line with spinners, expanded tasks view | CLI output + `--json` for agents |
| **Integration** | Built into Claude Code agent loop | CLI, MCP server, Claude Code plugin |
| **Language** | TypeScript (Bun-compiled binary) | Go |
| **Setup** | Zero (built-in, default enabled in interactive mode) | `bd init` per project |

---

## Detailed Comparison by Category

### 1. Persistence and Lifecycle

**Claude Code Tasks** are persistent in swarm/team mode. Each task is written as an individual JSON file at `~/.claude/tasks/{sanitized-team-name}/{id}.json`. The system uses `proper-lockfile` for concurrent access, maintains a `.highwatermark` file to ensure monotonically increasing IDs even after deletions, and supports a full CRUD lifecycle (create, read, update, delete, list, clear-all). Task files survive across process restarts within the same team context.

However, this persistence is **team-scoped, not project-scoped**. Tasks are keyed by team name, not by working directory or git repository. There is no built-in mechanism to associate tasks with a specific project, resume tasks in a new session, or aggregate tasks across different teams/projects.

In contrast, the TodoWrite system (solo mode) is purely in-memory and vanishes when the session ends.

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

**Key difference**: Claude Code's multi-agent coordination is real and robust, but limited to agents on the same machine sharing a filesystem. Beads coordinates agents across machines, sessions, and organizations via git. Claude Code also lacks the agent-as-bead self-monitoring pattern.

### 5. Workflow Capabilities

**Claude Code Tasks** have no workflow engine. The agent creates tasks with dependencies and executes them in order using its own reasoning. There are no templates, no reusable patterns, no gate steps.

**Beads** has a full molecular workflow system:
- Formulas (TOML templates) define reusable multi-step workflows
- Molecules instantiate formulas as active work
- Wisps provide ephemeral local execution without sync overhead
- Gates enable async coordination (wait for CI, wait for human approval)
- Bonding connects work graphs across molecules

**Key difference**: Unchanged -- Beads has a workflow engine; Claude Code does not.

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

### 8. Context Management

**Claude Code Tasks** have no compaction or memory management. In solo mode (TodoWrite), the entire list is cleared when all items are completed. In swarm mode (Tasks), completed tasks remain as JSON files indefinitely.

**Beads** implements semantic memory decay:
- Tier 1: 70% compression after 30 days closed
- Tier 2: 95% compression after 90 days closed
- Tombstone lifecycle for safe deletion propagation
- Wisp TTL-based compaction for ephemeral operations

**Key difference**: Unchanged -- Beads manages long-term context growth; Claude Code does not.

### 9. Setup and Adoption

**Claude Code Tasks** require zero setup. The Tasks system is enabled by default in interactive mode. The agent decides when to use it. No configuration, no initialization, no external binary.

**Beads** requires installation of the `bd` binary, `bd init` in each project, optional daemon setup, and optional git hook installation. It offers progressive adoption (`--stealth` -> contributor -> full team -> federation).

**Key difference**: Unchanged -- Claude Code is zero-friction; Beads has meaningful onboarding cost.

---

## Architectural Philosophy Comparison

### Claude Code: Coordination-Capable Agent Infrastructure

The Claude Code task system in swarm mode is more than a thinking aid -- it is a coordination layer for a team of agents on a single machine. It provides:
- Persistent shared state via filesystem JSON
- Race-condition-safe task claiming via file locks
- Graceful degradation via auto-unassign on shutdown
- Background task monitoring via typed IDs and output files

But it is deliberately **session-centric** and **machine-local**. It does not attempt to be a project management system, a version-controlled issue tracker, or a workflow engine. The intelligence is in the agents; the system provides just enough infrastructure for them to coordinate.

### Beads: Infrastructure-as-Workflow

Beads treats tasks as **persistent infrastructure**. Issues are first-class entities with rich metadata, distributed storage, and formal dependency semantics. The system embodies workflow logic:
- Dependencies ARE the execution graph
- `bd ready` computes what to do next
- Formulas encode repeatable processes
- Compaction manages long-term memory
- Git distribution enables cross-machine, cross-org coordination

The intelligence is shared between agents and infrastructure.

---

## Where Claude Code Falls Short of Beads

Despite having more capabilities than initially apparent, Claude Code's task system still lacks:

1. **Multi-project awareness**: No concept of projects, no aggregated cross-project views
2. **Cross-session ready computation**: No `bd ready` equivalent -- no way to ask "what work is available?"
3. **Workflow templates**: No formulas, no reusable patterns, no gate steps
4. **Git-native storage**: Filesystem JSON, not version-controlled, no distribution
5. **Compaction / memory decay**: No management of long-lived task backlogs
6. **Rich dependency types**: Only blocks/blockedBy, no conditional-blocks, waits-for, related, discovered-from
7. **Agent self-monitoring**: No agent-as-bead pattern, no stuck/dead detection
8. **Federation**: No cross-org synchronization
9. **Content hashing**: No deduplication or convergent state detection
10. **Cross-machine coordination**: File-locking only works for agents sharing a filesystem

---

## Where Claude Code Matches or Exceeds Expectations

Areas where Claude Code's task system is more capable than a naive reading suggests:

1. **Persistence**: Real file-based persistence in swarm mode (not ephemeral)
2. **File locking**: `proper-lockfile` for atomic operations -- race-condition safe
3. **Ownership model**: Explicit owner field with auto-assign on claim
4. **Claim semantics**: Five distinct failure reasons (not_found, already_claimed, already_resolved, blocked, agent_busy)
5. **Agent busy check**: Prevents an agent from hoarding multiple tasks
6. **Graceful shutdown**: Auto-unassign with notification when an agent terminates
7. **Highwatermark**: IDs are globally unique within a team (never reused, even after deletion)
8. **Dependency enforcement**: blockedBy checked at claim time, cleaned up on deletion
9. **Background task system**: Typed IDs, output files, incremental read with offset
10. **Zero setup**: All of this works out of the box, no configuration needed

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

### Use Both Together:
- Beads manages the persistent backlog, multi-project view, and cross-session continuity
- Claude Code's swarm-mode Tasks manages in-session execution and agent coordination
- An agent reads from `bd ready`, claims a Beads task, uses Claude Code's task system to coordinate subtask execution among team agents, then closes the Beads issue when done

---

## Summary Table

| Concern | Claude Code (Swarm Mode) | Beads | Winner |
|---------|-------------------------|-------|--------|
| Zero-friction start | Built-in, default on | Requires setup | Claude Code |
| Single-session coordination | Excellent (file locking, ownership, busy checks) | Not designed for this | Claude Code |
| Persistence | Yes (filesystem JSON) | Yes (SQLite + JSONL in git) | Beads (git > filesystem) |
| Multi-session continuity | Partial (files persist, no resume workflow) | Excellent (ready, handoff, sync) | Beads |
| Multi-project view | None | Native | Beads |
| Multi-agent (same machine) | Excellent (locking, claiming, auto-unassign) | Not specifically optimized for this | Claude Code |
| Multi-agent (cross-machine) | None | Excellent (git-based) | Beads |
| Workflow automation | None | Molecules + Formulas + Gates | Beads |
| Dependency richness | Basic (2 types, enforced) | Rich (6 types, computed readiness) | Beads |
| Context management | None | Compaction + decay | Beads |
| Data model simplicity | Simple (10 fields + metadata) | Complex but comprehensive (40+ fields) | Depends on needs |
| Git integration | None | Native | Beads |
| Federation | None | Peer-to-peer | Beads |
| Agent self-monitoring | None | Agent-as-bead | Beads |
| Learning curve | Zero | Moderate | Claude Code |
| ID collision safety | Sequential + highwatermark (safe within team) | Hash-based (safe across organizations) | Beads |
