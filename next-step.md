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
| System 4 (Intelligence) | Feedback evaluation: comparing results against goals, detecting stalls, adjusting dispatch parameters | Orchestration | Feedback controller |
| System 5 (Policy) | Identity and boundaries: what the system is and is not allowed to do | Human governance | You |

**Ashby's Law of Requisite Variety** is the principle that connects these layers to the orchestration problem. A regulator must have at least as much variety as the system it regulates. Deterministic orchestrators (scripts, Temporal Workflows) have the *right variety profile* for workflow control: they produce exactly the variety needed (retry, re-dispatch, route) and no more. Probabilistic agents produce *too much* variety for this role — they forget instructions, skip steps, hallucinate workflow states. This is why `orchestration-principle.md` argues that orchestration belongs in deterministic systems, not in agents. It is Ashby's Law applied.

**The autonomy envelope** — the set of constraints defining what an agent may do within a session (permitted tools, spending thresholds, escalation paths, stop conditions) — is a homeostatic boundary in Ashby's sense. The agent is free to vary within the envelope; crossing it triggers corrective action. Claude Code's task system operates inside this envelope. The orchestrator defines the envelope. You (System 5) define the orchestrator's boundaries.

**The orchestration loop is not workflow replay — it is iterative convergence.** Dispatch a task, evaluate the result, adjust, re-dispatch. Single-loop: correct execution errors (retry failures, adjust parameters). Double-loop: question whether you're solving the right problem (reframe the task, change the agent, revise the decomposition). This is Argyris's distinction applied to agent orchestration, and it is the mechanism that makes cybernetic control practical. You don't need deterministic replay of identical steps — you need a feedback loop that converges on the right result through successive corrections. See `goal-cybercontrol.md` Section 5a for the full argument.

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

### 3. Full Stack — Feedback Controller (+ Beads)

**VSM coverage**: Systems 1-5 (all layers, with you as System 5)

The orchestrator is a feedback controller: dispatch a task, evaluate the result against goals, adjust, re-dispatch. This is the cybernetic loop applied to agent orchestration. It is simpler, cheaper, and more principled than a workflow replay engine.

**Primary: Feedback controller script** — A shell/Python script that implements the dispatch-evaluate-adjust cycle. It reads a task file, dispatches work to agents via CLI, evaluates results (exit codes, output inspection, test results), and decides what to do next. Your own state (files, SQLite, YAML — not `~/.claude/`). Switching agents = changing a command.

The feedback controller handles:
- **Crash recovery**: scan for stalled tasks (no heartbeat, exceeded timeout), re-dispatch with context about what failed. No replay needed — the loop converges by retrying with better information, not by reproducing identical steps.
- **Vendor independence**: dispatches to any CLI agent (`claude`, `codex`, `aider`, etc.). The agent is an Activity; the controller is the loop.
- **Single-loop correction**: task failed → retry with same parameters (transient failure, timeout, resource contention).
- **Double-loop correction**: task failed repeatedly → evaluate why → adjust the task definition, change the agent, revise the decomposition → re-dispatch with revised approach.

Effort: 2-3 days.

**Tradeoffs**:
- (+) Vendor-independent, minimal infrastructure, no server
- (+) Principled: implements the cybernetic model directly
- (+) Handles crash recovery via timeout + re-dispatch (sufficient for most cases)
- (+) Double-loop: can change approach when stuck, not just retry
- (-) You build and maintain it yourself
- (-) No replay of partial work within a single agent invocation (but the loop compensates by re-dispatching with context)

Optionally add Beads (Approach 2) for the planning layer. Beads handles "what work exists?"; the feedback controller handles "dispatch, evaluate, adjust"; Claude Code handles "how does the agent organize work within a session?"

**For production/enterprise systems**: Temporal (temporal.io) remains an option for cases requiring guaranteed completion of fixed, deterministic workflows — CI/CD pipelines, deployment sequences, compliance-critical processes where every step must execute exactly once in order. See `temporal-analysis.md`. But Temporal's model (deterministic replay of fixed workflows) is not the general recommendation for agent orchestration, where the work is inherently exploratory and the right approach emerges through iteration, not replay.

**When to use**: Crash recovery matters. Multiple agent providers. Long-running, multi-step workflows. Any case where you need the orchestration loop — dispatch, evaluate, adjust — to be deterministic and vendor-independent.

---

## Decision Matrix

| Your Situation | VSM Layers Needed | Approach |
|---|---|---|
| 1 project, short sessions | Systems 1+2 | **1. Execution Only** — built-in tasks + task list in repo |
| 1 project, cross-session work | Systems 1+2+3 | **2. Execution + Planning** — add Beads, or keep task list in repo with Approach 1 |
| 2+ projects, multi-session | Systems 1+2+3+3* | **2. Execution + Planning** — Beads for multi-project backlog |
| Multiple agent providers | Systems 1-4 | **3. Full Stack** — feedback controller dispatches to any CLI agent |
| Crash recovery is critical | Systems 1-4 | **3. Full Stack** — feedback controller with timeout + re-dispatch |
| Long-running, multi-step workflows | Systems 1-4 | **3. Full Stack** — feedback controller for dispatch-evaluate-adjust loop |
| Production systems (CI/CD, deployments) | Systems 1-5 | **3. Full Stack** — feedback controller; consider Temporal for fixed deterministic pipelines |
| Budget is zero | Systems 1+2+3 | **2. Execution + Planning** — Beads is free + git-backed. Or **3** (feedback controller is just a script) |
| Committed to Claude Code only | Systems 1+2 | **1. Execution Only** — with discovery tool. Understand the coupling cost |

---

## Verdict

**Why this matters**: Persistent cross-project tasks accumulate compounding value. Discovered issues, dependency insights, and workflow patterns compound across sessions instead of evaporating. Agents are more effective when they know what was done before. The gap is real and the pain grows with the number of projects and sessions.

**Why it might not**: Claude Code is actively evolving — multi-project awareness and cross-session resume are natural next steps that may arrive natively. For solo developers on one project with short sessions, the built-in tasks are already sufficient. Every tool in your workflow consumes attention; if your bottleneck is elsewhere (context limits, code quality, test coverage), task management infrastructure may not be the highest-leverage investment.

**The organizing principle**: The orchestrator is a feedback controller that iteratively converges on the right result. Dispatch, evaluate, adjust, re-dispatch. Single-loop corrects execution errors; double-loop questions whether the approach is right. This is simpler and cheaper than workflow replay infrastructure, and it matches how agent work actually proceeds — you don't reproduce identical steps, you retry with better information. Claude Code's task system is a useful within-session convenience — let agents use it for self-organization. But the orchestration loop itself should be deterministic, vendor-independent, and yours. And you — the human — are System 5. You define what the system is, what it may do, and when to restructure it.

---

*This document is part of the task-analysis project. See `goal-cybercontrol.md` for the cybernetic theory, `orchestration-principle.md` for the underlying principle, `task-system-comparison.md` for the 3-way comparison, `temporal-analysis.md` for the Temporal deep-dive, `beads-task-system.md` for the Beads deep-dive, and `task-system-source-analysis.md` for the binary analysis.*
