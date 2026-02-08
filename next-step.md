# Next Step: Extending Claude Code Tasks Toward Multi-Project Management

## The Question

> If I wanted Beads-like multi-project task management today, working primarily with Claude Code, what should I do?

This document explores the gap between Claude Code's task systems and persistent multi-project infrastructure, evaluates three approaches ordered by architectural coverage, and recommends a path based on your situation.

---

## The Cybernetic Frame

The three-layer architecture described in `task-system-comparison.md` (Execution / Planning / Orchestration) maps onto Stafford Beer's Viable System Model (VSM). This mapping is not decorative — it explains *why* each layer exists, *what regulatory function* it performs, and *how* to decide which layers your situation requires.

| VSM System | Function | Layer | Implementation |
|---|---|---|---|
| System 1 (Operations) | First-order work: agent executes code, runs tests, edits files | Execution | Claude Code Tasks |
| System 2 (Coordination) | Anti-oscillation: file locking, atomic claiming, dependency enforcement | Execution | Claude Code Tasks |
| System 3 (Control) | Internal regulation: what work exists, what is ready, what is blocked | Planning | Beads |
| System 3* (Audit) | Sporadic monitoring: compaction, content hashing, ready computation | Planning | Beads |
| System 4 (Intelligence) | Environmental scanning: crash detection, retry logic, multi-provider routing | Orchestration | Temporal |
| System 5 (Policy) | Identity and boundaries: what the system is and is not allowed to do | Human governance | You |

**Ashby's Law of Requisite Variety** is the principle that connects these layers to the orchestration problem. A regulator must have at least as much variety as the system it regulates. Deterministic orchestrators (scripts, Temporal Workflows) have the *right variety profile* for workflow control: they produce exactly the variety needed (retry, re-dispatch, route) and no more. Probabilistic agents produce *too much* variety for this role — they forget instructions, skip steps, hallucinate workflow states. This is why `orchestration-principle.md` argues that orchestration belongs in deterministic systems, not in agents. It is Ashby's Law applied.

**The autonomy envelope** — the set of constraints defining what an agent may do within a session (permitted tools, spending thresholds, escalation paths, stop conditions) — is a homeostatic boundary in Ashby's sense. The agent is free to vary within the envelope; crossing it triggers corrective action. Claude Code's task system operates inside this envelope. The orchestrator defines the envelope. You (System 5) define the orchestrator's boundaries. See `goal-cybercontrol.md` for the full cybernetic argument.

---

## What Claude Code Already Has

Claude Code v2.1.34 has two mutually exclusive task systems:

1. **TodoWrite (solo mode)**: In-memory, per-session, per-agent. Three fields (content, status, activeForm). Ephemeral — vanishes when the session ends.

2. **Tasks (swarm/team mode)**: File-persisted JSON at `~/.claude/tasks/`. Full CRUD (TaskCreate, TaskGet, TaskUpdate, TaskList). Dependency tracking with enforcement at claim time. Atomic claiming via `proper-lockfile`. Agent busy checks, auto-unassign on shutdown.

The within-session infrastructure is solid: persistence, locking, ownership, dependencies, multi-agent coordination, background task management with typed IDs. This covers VSM Systems 1+2 — operations and coordination within a single session.

See `claude-task-system.md` for the full technical analysis and `task-system-source-analysis.md` for the binary-level details.

---

## The Gap

Cross-session continuity is fundamentally broken. Task lists are scoped to random UUIDs (`bR()` generates a UUIDv7 per session), making tasks undiscoverable after the session ends. The `--resume` flag and `CLAUDE_CODE_TASK_LIST_ID` env var provide manual recovery, but require knowing session UUIDs that are not exposed in any user-facing interface.

| Missing Capability | VSM Layer Absent | Gap Size |
|---|---|---|
| Task discoverability across sessions | System 3 (no "what exists?" query) | **Large** |
| Crash recovery (tasks stuck in `in_progress`) | System 4 (no failure detection/retry) | **Large** |
| Multi-project awareness | System 3 (no cross-project registry) | **Large** |
| Cross-session ready computation | System 3 (no `bd ready` equivalent) | **Large** |
| Workflow templates / reusable patterns | System 3 (no formulas or gate steps) | **Large** |
| Git-native storage / distribution | System 3* (no version control or cross-machine sync) | **Medium** |
| Compaction of completed tasks | System 3* (no memory decay) | **Medium** |

Three strategic risks compound this gap:

1. **Internal API fragility**: `~/.claude/tasks/` and `sessions-index.json` are undocumented. Any update can break tooling silently.
2. **Vendor lock-in**: Orchestration tied to Claude Code internals is non-portable across agent platforms.
3. **Agent unpredictability**: Agents forget instructions, skip steps, and hallucinate workflow states — they are unreliable orchestrators.

See `orchestration-principle.md` for the full strategic argument.

---

## Three Approaches

Ordered by how many VSM layers they provide. Choose the first approach that covers your needs.

### 1. Execution Only — Claude Code As-Is

**VSM coverage**: Systems 1+2 (Operations + Coordination)

Use Claude Code's built-in Tasks system for in-session work. Keep a task list in your repo (`TODO.md`, YAML plan file) as the cross-session source of truth that any agent or human can read.

Optionally build a discovery tool (`claude-tasks` shell script) that scans `~/.claude/tasks/`, cross-references `sessions-index.json`, and generates `--resume` commands. This solves the discoverability gap but deepens coupling to undocumented internals.

**Effort**: Zero (as-is) or 1-2 days (with discovery tool).

**Tradeoffs**:
- (+) Zero setup, zero dependencies
- (+) Within-session coordination already works well
- (+) Discovery tool is fast to build, solves an immediate pain point
- (-) No cross-session continuity beyond manual workarounds
- (-) No multi-project awareness
- (-) No crash recovery — tasks stuck permanently on crash
- (-) Discovery tool couples to undocumented internals

**When to use**: Solo developer, single project, short sessions. Work that fits within a single session context.

### 2. Execution + Planning — Claude Code + Beads

**VSM coverage**: Systems 1+2+3+3* (Operations + Coordination + Control + Audit)

Use Beads as the persistent multi-project backend. Use Claude Code's Tasks for in-session agent coordination. The two systems layer naturally: Beads handles cross-session, cross-project, cross-machine concerns; Claude Code handles within-session agent coordination.

**Implementation**:
1. `bd init` in each project, configure multi-repo
2. CLAUDE.md instructions: start sessions with `bd ready --json`, claim with `bd update --claim`, close with `bd close`, sync at session end
3. Claude Code's TaskCreate/TaskUpdate for in-session subtask decomposition
4. Optionally: Claude Code hooks for auto-sync on session start/end

**Effort**: Low. Install Beads, init repos, write CLAUDE.md instructions.

**Tradeoffs**:
- (+) Persistent multi-project backlog with `bd ready` computation
- (+) Git-native storage — distribution, version history, cross-machine sync
- (+) Workflow templates (formulas, molecules) for reusable patterns
- (+) Compaction built in
- (+) Each system does what it's best at — natural layering, not competition
- (-) External dependency (Go binary)
- (-) Two systems to understand (though they operate at different layers)
- (-) Agent must follow CLAUDE.md instructions correctly — integration is only as reliable as the agent's adherence
- (-) In-session subtask state still lost on crash (only Beads issues have crash resilience)
- (-) Beads is a young project — APIs may evolve

**When to use**: Multi-project work, cross-session continuity needed, team backlogs. The sweet spot for most developers managing 2+ projects with AI agents.

### 3. Full Stack — Claude Code + Orchestrator (+ Beads)

**VSM coverage**: Systems 1-5 (all layers, with you as System 5)

Add a deterministic orchestrator that owns the workflow. Agents are workers receiving tasks and returning results. The orchestrator handles progress tracking, crash recovery, retries, and "what next" logic.

Two concrete implementations:

**3a. Simple Script Orchestrator** — A shell/Python script that reads a task file, dispatches work to agents via CLI, tracks completion. Your own state (files, SQLite, YAML — not `~/.claude/`). Switching agents = changing a command. If an agent crashes, the script knows what was in progress and re-dispatches.

- Effort: 2-3 days
- (+) Vendor-independent, minimal infrastructure, no server
- (-) Limited crash recovery — can re-dispatch but no replay of partial work
- (-) You build and maintain it yourself

**3b. Temporal (Durable Execution)** — Temporal (temporal.io) guarantees Workflows run to completion through deterministic replay and automatic retry. Agents are Activities; if a Worker crashes, another replays from event history, skipping completed steps. OpenAI Codex and Replit Agent 3 run on it in production.

- Effort: ~1 week setup; infrastructure at ~$100/month (Cloud) or self-hosted
- (+) Automatic crash recovery with partial-work preservation
- (+) Multi-agent routing via Task Queues (Claude for reasoning, cheaper model for boilerplate)
- (+) Human-in-the-loop via Signals, complete auditability
- (+) Production-proven at scale
- (-) Infrastructure cost and operational burden
- (-) Steep learning curve (determinism constraints, event sourcing, Workflow versioning)
- (-) Overkill for solo developer with short sessions

Add Beads (Approach 2) for the planning layer regardless of which orchestrator you choose. Temporal handles "did this complete?"; Beads handles "what work exists?"; Claude Code handles "how does the agent organize work right now?" This is the three-layer stack from `task-system-comparison.md`.

See `temporal-analysis.md` for the full Temporal analysis.

**When to use**: Crash recovery matters. Multiple agent providers. Production systems (CI/CD, deployments). Multi-agent routing across machines. Long-running, multi-step workflows where losing progress is more expensive than infrastructure cost.

---

## Decision Matrix

| Your Situation | VSM Layers Needed | Approach |
|---|---|---|
| 1 project, short sessions | Systems 1+2 | **1. Execution Only** — built-in tasks + task list in repo |
| 1 project, cross-session work | Systems 1+2+3 | **2. Execution + Planning** — add Beads, or keep task list in repo with Approach 1 |
| 2+ projects, multi-session | Systems 1+2+3+3* | **2. Execution + Planning** — Beads for multi-project backlog |
| Multiple agents across machines | Systems 1-4 | **3. Full Stack** — Temporal (3b) + Beads |
| Multiple agent providers | Systems 1-4 | **3. Full Stack** — Temporal wraps any CLI agent as an Activity |
| Production system (CI/CD, deployments) | Systems 1-5 | **3. Full Stack** — Temporal (3b) for guaranteed completion |
| Crash recovery is critical | Systems 1-4 | **3. Full Stack** — only Temporal (3b) preserves partial work |
| Budget is zero | Systems 1+2+3 | **2. Execution + Planning** — Beads is free + git-backed. Or **3a** (simple script) |
| Committed to Claude Code only | Systems 1+2 | **1. Execution Only** — with discovery tool. Understand the coupling cost |

---

## Verdict

**Why this matters**: Persistent cross-project tasks accumulate compounding value. Discovered issues, dependency insights, and workflow patterns compound across sessions instead of evaporating. Agents are more effective when they know what was done before. The gap is real and the pain grows with the number of projects and sessions.

**Why it might not**: Claude Code is actively evolving — multi-project awareness and cross-session resume are natural next steps that may arrive natively. For solo developers on one project with short sessions, the built-in tasks are already sufficient. Every tool in your workflow consumes attention; if your bottleneck is elsewhere (context limits, code quality, test coverage), task management infrastructure may not be the highest-leverage investment. Temporal's infrastructure cost and learning curve are only justified when losing progress is genuinely expensive.

**The organizing principle**: Orchestration belongs in your own deterministic system; agents are workers that receive tasks and return results. Claude Code's task system is a useful within-session convenience — let agents use it for self-organization. But your source of truth for "what needs doing" should be vendor-independent and deterministic. And you — the human — are System 5. No tool replaces your governance function: defining what the system is, what it may do, and when to restructure it.

---

*This document is part of the task-analysis project. See `goal-cybercontrol.md` for the cybernetic theory, `orchestration-principle.md` for the underlying principle, `task-system-comparison.md` for the 3-way comparison, `temporal-analysis.md` for the Temporal deep-dive, `beads-task-system.md` for the Beads deep-dive, and `task-system-source-analysis.md` for the binary analysis.*
