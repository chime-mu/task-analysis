# Next Step: Extending Claude Code Tasks Toward Multi-Project Management

**[REVISED 2026-02-08]**: Updated with new findings about session-scoped UUIDs (`bR()` fallback), the distinction between file persistence and practical persistence, `--resume` / `--continue` / `CLAUDE_CODE_TASK_LIST_ID` as partial mitigations, and crash fragility as an observed failure mode. Further revised to add strategic risk analysis (vendor coupling, internal API fragility, deterministic orchestration principle) and reframe recommendations. See `orchestration-principle.md`.

**[REVISED 2026-02-08, Temporal update]**: Added Temporal (temporal.io) as a concrete implementation of Approach D. Approach D split into D1 (simple script orchestrator) and D2 (Temporal). Recommendations, effort estimates, and decision matrix updated. See `temporal-analysis.md` and `task-system-comparison.md`.

## The Question

> If I wanted Beads-like multi-project task management today, working primarily with Claude Code, what should I do?

This document explores what it would take to bridge the gap between Claude Code's task systems and Beads' persistent multi-project infrastructure, evaluates the practical approaches available, and concludes with a pro/con assessment of whether the effort is worthwhile.

---

## Current State: What Claude Code Actually Has

Before evaluating approaches, we need an accurate picture of what Claude Code v2.1.34 already provides. The source analysis reveals significantly more capability than surface-level inspection suggests.

### What Already Exists

**Two mutually exclusive task systems:**

1. **TodoWrite (solo mode)**: In-memory, per-session, per-agent. Three fields (content, status, activeForm). Ephemeral -- vanishes when the session ends. Enabled when the Tasks feature gate is off.

2. **Tasks (swarm/team mode)**: File-persisted JSON at `~/.claude/tasks/{team-name}/`. Full CRUD tools (TaskCreate, TaskGet, TaskUpdate, TaskList). Dependency tracking (blocks/blockedBy) with enforcement at claim time. This is enabled by default in interactive sessions.

**Multi-agent coordination already built in:**
- Team management tools: TeamCreate, TeamDelete, SendMessage
- Atomic task claiming with `proper-lockfile` file locks
- Agent busy checks (prevents hoarding multiple tasks)
- Five structured claim-failure reasons (task_not_found, already_claimed, already_resolved, blocked, agent_busy)
- Auto-unassign on agent shutdown with notification for reassignment
- Background task system with typed IDs (local bash, local agent, remote agent, in-process teammate)
- Output files per background task with incremental read support

**Persistence mechanics:**
- Individual JSON files per task (pretty-printed, schema-validated on read)
- `.highwatermark` file ensures IDs never reuse, even after deletion or clear-all
- `.lock` file for global operations; per-task file locks for claiming
- Completion verification hooks before marking tasks complete

### What Does NOT Exist

1. **Multi-project awareness**: Tasks are keyed by team name, not by project/directory. No concept of "which repo does this task belong to?"
2. **Cross-session ready computation**: No `bd ready` equivalent. Task files persist, but there is no tool or workflow to ask "what work is available across all my projects?"
3. **Workflow templates**: No formulas, no reusable patterns, no gate steps for async coordination
4. **Git-native storage**: Local filesystem only. No version control, no distribution to other machines
5. **Compaction / memory decay**: Completed tasks accumulate as JSON files indefinitely
6. **Rich dependency types**: Only blocks/blockedBy. No conditional-blocks, waits-for, related, discovered-from
7. **Agent self-monitoring**: No agent-as-bead pattern. No stuck/dead detection across sessions
8. **Content hashing**: No deduplication or convergent state detection
9. **Cross-machine coordination**: File locking is local-only

---

## The Actual Gap

Given what Claude Code already has, the gap is narrower than initially assumed but still real in specific dimensions:

| Capability | Status | Gap Size |
|-----------|--------|----------|
| File persistence | **Already exists** (JSON files on disk) | None |
| Task discoverability / practical persistence | Does not exist -- session-scoped UUIDs, no discovery tool | **Large** |
| Crash recovery | Does not exist -- tasks stuck in `in_progress` permanently | **Large** |
| Multi-agent coordination | **Already exists** (file locking, claiming, busy checks) | None for same-machine |
| Dependency tracking | **Already exists** (blocks/blockedBy, enforced at claim) | Small (lacks rich types) |
| Auto-incrementing IDs with highwatermark | **Already exists** | None |
| Multi-project awareness | Does not exist | **Large** |
| Cross-session ready computation | Does not exist | **Large** |
| Workflow templates | Does not exist | **Large** |
| Git-native storage / distribution | Does not exist | **Medium-Large** |
| Compaction | Does not exist | **Medium** |
| Cross-machine agent coordination | Does not exist | **Medium** |

**[REVISED]** The gap has two layers. **Within-session infrastructure is solid**: persistence, locking, ownership, dependencies, multi-agent coordination all work well. But **cross-session continuity is fundamentally broken**: task lists are scoped to random UUIDs (`bR()` generates a UUIDv7 per session), making tasks undiscoverable after the session ends. The `--resume <sessionId>` CLI flag and `CLAUDE_CODE_TASK_LIST_ID` env var provide manual recovery paths, but require knowing the session UUID -- which is not exposed in any user-facing interface. The gap is about **discoverability** (finding previous tasks), **scope** (multi-project), **workflow** (templates, ready computation), and **distribution** (git, cross-machine).

---

## Strategic Risks of Building on Agent Internals

Before evaluating approaches, it is worth stepping back to consider the strategic risks of building tooling on top of any agent's internal task system.

### 1. Internal API Fragility

`~/.claude/tasks/`, `sessions-index.json`, and `CLAUDE_CODE_TASK_LIST_ID` are undocumented implementation details. Anthropic has not exposed tasks as an external interface -- no `claude tasks list` CLI command, no public API, no documented schema. This signals intent to maintain freedom to change these internals. A tool built on them today may break silently with any update.

### 2. Vendor Lock-in

Claude Code is one agent platform. Codex, Gemini CLI, open-source agent frameworks each have their own (or no) task systems. Orchestration logic coupled to Claude Code's internals is non-portable. If a better tool emerges or pricing changes, you start over.

### 3. Agents Are Unreliable Orchestrators

Agents excel at reasoning and code generation. They are unreliable for deterministic workflows: running test suites, committing to git, deciding what to work on next, tracking progress across sessions. They forget instructions, skip steps, and hallucinate workflow states. Traditional software is simply better at predictable execution.

**The strategic response**: Orchestration belongs in your own deterministic system. Agents are workers that receive tasks and return results. Agent-internal task systems are within-session conveniences, not foundations to build on. See `orchestration-principle.md` for the full argument.

**Temporal addresses all three risks.** It is vendor-independent (agents are Activities that invoke CLI commands -- switching providers means changing a command). It is not coupled to any agent's internals (state lives in Temporal's event history, not in `~/.claude/`). And it replaces agent-driven orchestration with deterministic Workflow code that cannot forget instructions or skip steps. See `temporal-analysis.md` for the full analysis.

---

## Three Approaches (Revised)

### Approach A: Use Beads Directly

**What**: Install Beads (`bd`), initialize in each project, use the Claude Code plugin or CLI integration.

**How it works today**:
1. `bd init` in each project repository
2. Configure multi-repo: `bd config set repos.additional "~/repo1,~/repo2"`
3. Add Beads agent instructions to your `CLAUDE.md` or system prompt
4. Claude Code uses `bd ready --json` to find work, `bd update --claim` to claim it, `bd close` to complete it
5. Claude Code's built-in Tasks system handles in-session agent coordination for each Beads issue

**Effort**: Low-medium. Install Beads, init repos, write CLAUDE.md instructions.

**Tradeoffs**:
- (+) Production-ready system with multi-project, multi-agent, workflow support
- (+) Git-native persistence and distribution -- the main thing Claude Code lacks
- (+) Full `bd ready` computation, formulas, compaction
- (+) The Beads claude-plugin provides slash commands out of the box
- (-) External dependency (Go binary)
- (-) Learning curve for the molecule/formula/chemistry concepts
- (-) Two task systems running simultaneously -- but this is less problematic now since Claude Code's Tasks system already handles in-session coordination well, and Beads handles the cross-session/cross-project layer
- (-) Beads is a young project -- API and concepts may evolve

### Approach B: Tactical Discovery Tool (Claude Code-Specific)

> **Caveat**: This tool builds directly on Claude Code's undocumented internals (`~/.claude/tasks/`, `sessions-index.json`). It is a tactical band-aid, not a strategic solution. It solves a real immediate problem but deepens vendor coupling. See "Strategic Risks" above and `orchestration-principle.md`.

**What**: Since Claude Code already has file persistence, locking, dependencies, and coordination, build a missing piece: **task discoverability** -- the ability to find, inspect, and resume previous sessions' tasks.

**Implementation sketch -- a `claude-tasks` shell script**:

The data sources already exist. `~/.claude/projects/*/sessions-index.json` records session metadata (session ID, project path, git branch, first prompt, summary, timestamps) for all known sessions. `~/.claude/tasks/*/` contains the actual task files. The script connects these:

1. **Scan** `~/.claude/projects/*/sessions-index.json` for all known sessions across all projects
2. **Cross-reference** against `~/.claude/tasks/*/` directories -- match session IDs to task list UUIDs
3. **Read task JSON files** to find non-empty task lists with `pending` or `in_progress` work
4. **Display** actionable information:
   - Project path and git branch
   - Session date and first prompt (from sessions-index)
   - Task count, task subjects, statuses, and dependency DAG
   - Whether tasks are stuck (e.g., `in_progress` from a crashed session)
5. **Generate the resume command**: `CLAUDE_CODE_TASK_LIST_ID=<uuid> claude --resume <sessionId>`

**Companion CLAUDE.md instruction**:
```
When CLAUDE_CODE_TASK_LIST_ID is set, call TaskList at the start of the session
to review existing tasks before creating new ones. Check for tasks stuck in
in_progress status (likely from a previous crash) and reset them to pending.
```

**What you do NOT need to build** (because Claude Code already has it):
- Task persistence (already JSON files on disk)
- File locking (already `proper-lockfile`)
- Ownership and claiming (already `oBA()` / `SyD()`)
- Dependency enforcement (already checked at claim time)
- Agent coordination (already team system with auto-unassign)

**Effort**: Low. ~1-2 days. The tool is straightforward shell scripting; the data sources exist and are human-readable JSON.

**Tradeoffs**:
- (+) Builds on existing Claude Code infrastructure -- no external binary
- (+) Solves the critical discoverability gap directly
- (+) Generates ready-to-use resume commands
- (+) Can detect crashed/stuck tasks
- (+) Task files are already human-readable JSON
- (+) Stays within the Claude Code ecosystem
- (-) No git distribution -- still local filesystem only
- (-) No daemon, no background sync
- (-) No compaction
- (-) No workflow templates or gate steps
- (-) The discovery is a separate script, not integrated into the agent loop
- (-) Cross-project dependencies via metadata are a convention, not enforced
- (-) Relies on `sessions-index.json` existing and being populated (it is for interactive sessions)
- (-) **Couples to undocumented internals** -- `~/.claude/tasks/` path, JSON format, and `sessions-index.json` structure are all implementation details that can change without notice
- (-) **Vendor lock-in** -- entirely Claude Code-specific; useless if you switch to Codex or open-source agents
- (-) **Doesn't solve the fundamental problem** -- orchestration decisions still depend on an agent's internal state rather than your own system

### Approach C: Hybrid -- Beads for Multi-Project Planning, Claude Code for Execution

**What**: Use Beads as the persistent multi-project backend. Use Claude Code's native Tasks system for in-session agent coordination. The two systems operate at different layers.

**Implementation**:

1. Install Beads and init in projects
2. Write CLAUDE.md instructions that teach Claude Code to:
   - Start sessions with `bd ready --json` to load the backlog
   - Claim Beads issues with `bd update --claim`
   - Use Claude Code's TaskCreate/TaskUpdate for in-session subtask decomposition among team agents
   - Close Beads issues when work completes
   - Run `bd sync` at session end
3. Optionally write a Claude Code hook that auto-syncs Beads on session start/end

**Why this works better than we initially thought**: The "two systems" problem is less severe because the systems naturally layer. Beads handles the cross-session, cross-project, cross-machine layer. Claude Code's Tasks handles the within-session, same-machine agent coordination layer. They do not compete -- they complement.

**Effort**: Low. Mostly documentation/prompting.

**Tradeoffs**:
- (+) Best of both worlds: persistent multi-project planning + native agent coordination
- (+) Minimal custom code
- (+) Each system does what it is best at
- (+) Can progressively adopt Beads features (molecules, formulas) as needed
- (+) `--resume` + `CLAUDE_CODE_TASK_LIST_ID` makes the hybrid more viable than previously thought -- in-session subtask state can be reconnected
- (-) Two systems to understand (but they operate at different layers)
- (-) Relies on agent following CLAUDE.md instructions correctly
- (-) Beads is still an external dependency
- (-) In-session subtask state (Claude Code's Tasks) is still lost on crash even when Beads handles the outer layer -- only Beads issues have crash resilience

### Approach D: Vendor-Independent Deterministic Orchestration

**What**: Build or adopt a deterministic orchestration system that owns the workflow. Agents are workers that receive tasks and return results. The orchestrator handles progress tracking, crash recovery, retries, and "what next" logic through traditional software.

This approach has two concrete implementations:

#### Approach D1: Simple Script Orchestrator

**What**: A lightweight script (shell, Python, etc.) that reads a task file (YAML, JSON, or TOML), dispatches work to agents via CLI, and tracks completion state.

**Principle**:
- The orchestrator maintains its own task state (in files, SQLite, a database -- whatever suits your needs). Not in `~/.claude/`.
- It dispatches work to agents via their CLI interfaces (`claude`, `codex`, etc.). Switching agents = changing a command.
- It handles deterministic operations itself: running test suites, committing to git, checking CI status.
- If an agent crashes mid-task, the orchestrator knows what was in progress and re-dispatches.
- Claude Code's internal TaskCreate/TaskList are fine for the agent's self-organization within a session. They're a convenience, not the source of truth.

**What agent-internal tasks become**: A helpful within-session feature. The agent can use TaskCreate to break down a complex coding task into subtasks. But the *outer loop* -- what the agent should be working on, what's been done, what failed -- is owned by the orchestrator.

**Effort**: ~2-3 days for a basic version.

**Tradeoffs**:
- (+) Vendor-independent -- works with any agent that has a CLI
- (+) Crash-resilient -- orchestrator state is not tied to agent session
- (+) Deterministic -- traditional software doesn't forget instructions
- (+) Future-proof -- survives Claude Code internal changes, pricing changes, provider switches
- (+) Minimal infrastructure -- runs as a local script, no server needed
- (-) You must build it yourself -- no off-the-shelf solution
- (-) Limited crash recovery -- the script can re-dispatch, but has no replay of partial work
- (-) No built-in retry policies, timeouts, or activity-level recovery
- (-) You lose the zero-setup convenience of Claude Code's built-in task system as a cross-session mechanism (but as documented, that mechanism is effectively broken anyway)

#### Approach D2: Temporal (Production Durable Execution)

**What**: Use Temporal (temporal.io) as the orchestration layer. Temporal is a durable execution platform that guarantees Workflows run to completion through crash recovery, deterministic replay, and automatic retry. OpenAI Codex and Replit Agent 3 run on it in production.

**How it works**:
- **Workflows** are deterministic functions that define the overall process (e.g., "fetch issue, generate code, run tests, create PR")
- **Activities** are non-deterministic functions where side effects happen -- including invoking AI agents via CLI (`claude --print`, `codex --quiet`, etc.)
- **Workers** are processes you run that poll for work and execute your code
- **Event History** records every step; if a Worker crashes, another Worker replays from history, skipping completed Activities

**Concrete example**:
```python
@workflow.defn
class FixBugWorkflow:
    @workflow.run
    async def run(self, bug_description: str, repo_path: str):
        # Activity 1: Agent generates fix (retried automatically on crash)
        fix = await workflow.execute_activity(
            run_claude_agent, bug_description, repo_path,
            retry_policy=RetryPolicy(maximum_attempts=3)
        )
        # Activity 2: Deterministic test run
        await workflow.execute_activity(run_tests, repo_path)
        # Activity 3: Create PR
        await workflow.execute_activity(create_pr, repo_path)
```

If the Worker crashes after Activity 1, a new Worker replays the Workflow, sees Activity 1 already completed in the event history, skips it, and retries Activity 2. No work is lost.

**Effort**: ~1 week for setup + initial Workflows. Development server: 5 minutes (`temporal server start-dev`). Production: 1-2 days for infrastructure (Docker Compose or Temporal Cloud at ~$100/month).

**Tradeoffs**:
- (+) **Concrete implementation of the orchestration principle** -- not abstract, production-proven at OpenAI/Replit scale
- (+) Vendor-independent -- agents are Activities; switching providers = changing a CLI command
- (+) Automatic crash recovery with partial-work preservation (completed Activities cached in event history)
- (+) Built-in retry policies, timeouts, heartbeats, and configurable backoff
- (+) Multi-agent routing via Task Queues (Claude for reasoning, Codex for boilerplate)
- (+) Human-in-the-loop via Signals (wait for approval before deploying)
- (+) Complete auditability -- every step recorded with timestamps and payloads
- (+) Cross-machine coordination native (Workers can run anywhere)
- (+) Polyglot SDKs (Python, Go, TypeScript, Java, .NET)
- (-) **Infrastructure cost** -- requires a server (self-hosted or Cloud at ~$100/month+)
- (-) **Steep learning curve** -- determinism constraints, event sourcing model, Workflow versioning
- (-) **Overkill for simple cases** -- a solo developer with short sessions does not need durable execution
- (-) **Event history limit** -- ~51K events per execution; long-running Workflows need Continue-As-New
- (-) More upfront investment than D1's shell script approach

See `temporal-analysis.md` for the full analysis.

---

## Revised Recommendation

**The strategic answer is now concrete: the orchestration principle has a production-proven implementation.**

Previous revisions of this document recommended Approach D in the abstract -- "build or adopt a deterministic orchestration system." This left the reader to figure out what that means in practice. Temporal makes Approach D concrete. OpenAI Codex and Replit Agent 3 already run on it. The question is no longer "should you have a deterministic orchestrator?" but "which level of orchestrator matches your needs?"

### What changed from the original analysis:

1. **[REVISED] Claude Code is not ephemeral but IS fragile.** Task files persist as JSON on disk, so the "persistence gap" at the file level is smaller than originally assumed. However, file persistence != practical persistence. Task lists are scoped to random session UUIDs (`bR()` fallback), making them undiscoverable after the session ends. The `--resume <sessionId>` flag and `CLAUDE_CODE_TASK_LIST_ID` env var provide partial recovery paths, but require manually tracking UUIDs that are not exposed in any user-facing interface. Crash fragility is a real, observed failure mode.

2. **Claude Code already has multi-agent coordination**. The original analysis positioned this as a major Beads advantage. Claude Code actually has file-locking, atomic claiming, agent busy checks, and auto-unassign -- a robust same-machine coordination system. The advantage Beads has is specifically cross-machine coordination.

3. **[REVISED] Approach B is NOT obsolete -- it needs to build discoverability, not file persistence.** The original Approach B was "build a lightweight persistence layer for Claude Code Tasks." File persistence already exists, so that specific work is unnecessary. But the real missing piece -- task discoverability across sessions -- is a concrete, buildable tool. A discovery script that maps sessions to tasks and generates the correct `--resume` + `CLAUDE_CODE_TASK_LIST_ID` command would provide genuine value. See revised Approach B below.

4. **[REVISED] Building on agent internals is strategically risky.** The previous revision recommended a `claude-tasks` discovery tool as "critical" mitigation. This deepens coupling to Claude Code's undocumented internals. A strategically sound approach treats agent task systems as within-session conveniences and puts orchestration in your own vendor-independent system. See `orchestration-principle.md`.

5. **[NEW] Temporal makes Approach D concrete.** The orchestration principle is no longer abstract. Temporal is a production-proven durable execution platform that implements every aspect: deterministic Workflows, agents as Activities, automatic crash recovery, vendor-independent dispatch. The tradeoff is infrastructure cost and learning curve. See `temporal-analysis.md`.

### What stayed the same:

1. **Multi-project awareness is still missing** -- the biggest gap
2. **Cross-session ready computation is still missing** -- `bd ready` has no equivalent
3. **Workflow templates are still missing** -- no formulas, no reusable patterns
4. **Git-native storage is still missing** -- no distribution, no version history
5. **Compaction is still missing** -- long-lived backlogs will grow unbounded

### Updated concrete steps:

1. **If you need multi-project management today**: Install Beads, use Approach C. The hybrid works naturally because the systems layer (Beads = planning, Claude Code = execution).

2. **If you primarily work on one project with multiple agents**: Claude Code's swarm-mode Tasks already provide file persistence, coordination, and dependency enforcement. **However**, be aware of crash fragility: if a session crashes, tasks are stuck. Consider recording session UUIDs (visible in `~/.claude/projects/*/sessions-index.json`) or building the discovery tool (Approach B) as a safety net.

3. **If you want cross-session continuity on a single project**: The `--resume <sessionId>` flag + `CLAUDE_CODE_TASK_LIST_ID` env var makes this viable without Beads. Build the discovery tool (Approach B, ~1-2 days) to make resume practical instead of requiring manual UUID tracking.

4. **If crash recovery matters**: Use Temporal (Approach D2). A simple orchestrator script (D1) can re-dispatch failed tasks, but only Temporal preserves partial work -- completed Activities are cached in the event history and never re-executed on retry. The infrastructure cost (~$100/month for Cloud, or self-hosted Docker Compose) is justified when losing progress is more expensive than the subscription.

5. **If infrastructure cost is too high**: Use Approach D1 (simple script orchestrator). It provides vendor independence and basic crash re-dispatch without requiring a server. Combine with Beads for the planning layer.

6. **If you want the full stack**: Temporal (orchestration) + Beads (planning) + Claude Code Tasks (execution). This is three layers, not three competing systems. Temporal guarantees completion, Beads tracks the backlog, Claude does the work. See `task-system-comparison.md` for the architectural layer diagram.

7. **If you want to wait**: Claude Code's task infrastructure is actively being built out, but **waiting carries real risk without at minimum the discovery tool**. A single crash can leave valuable task state orphaned. The discovery tool (Approach B) is a ~1-2 day investment that provides immediate insurance against this failure mode.

---

## What Would It Take to Extend Claude Code Natively?

Given that Claude Code already has persistence, locking, ownership, dependencies, and multi-agent coordination, the remaining extensions are:

### Must-Have Extensions (for multi-project use)

1. **Project-Scoped Task Lists**: Map task list IDs to project directories. Currently, the task list ID comes from team name or `CLAUDE_CODE_TASK_LIST_ID`. Adding a project -> task list mapping would enable multi-project awareness without changing the underlying storage.

2. **Cross-Session Resume**: A `/resume-tasks` command or auto-load behavior. The task files already persist -- the missing piece is a workflow that reads them at session start and presents them to the agent.

3. **Ready Computation**: A `TaskReady` tool that computes tasks with no open blockers across all registered projects/task lists. The dependency enforcement code already exists in `oBA()` -- it just needs to be exposed as a standalone query.

4. **Multi-Project Registry**: A configuration in `~/.claude/settings.json` that lists project directories and their task list IDs. A `/tasks-all` command to show tasks across projects.

### Nice-to-Have Extensions

5. **Git-Backed Storage**: Export task JSON files to a git-tracked location (e.g., `.claude-tasks/` in each repo). This would enable distribution and version history.

6. **Cross-Project Dependencies**: Allow `blockedBy` to reference tasks in other task lists using a `{listId}:{taskId}` format.

7. **Session Handoff**: Auto-generate a handoff summary when a session ends, stored alongside persisted tasks. Next session loads the summary for context.

8. **Workflow Templates**: YAML files defining reusable task decomposition patterns. A `/workflow release` command that creates a predefined set of tasks with dependencies.

9. **Compaction**: Summarize completed tasks periodically to keep the task directory manageable.

10. **Content Hashing**: Add a content hash to task JSON for deduplication and convergent state detection across machines.

### Revised Estimated Effort

Note: Several items from the original estimate are now marked as "already exists" since Claude Code already has them.

| Extension | Effort | Status | Priority |
|-----------|--------|--------|----------|
| Task persistence (file-based) | -- | **Already exists** | -- |
| File locking for concurrency | -- | **Already exists** | -- |
| Ownership and claiming | -- | **Already exists** | -- |
| Dependency enforcement | -- | **Already exists** | -- |
| Agent busy check | -- | **Already exists** | -- |
| Auto-unassign on shutdown | -- | **Already exists** | -- |
| Background task system | -- | **Already exists** | -- |
| Task discoverability tool (`claude-tasks`) | 1-2 days | New | Tactical |
| **Simple script orchestrator** (Approach D1) | **2-3 days** | **New** | **Strategic** |
| **Temporal setup + initial Workflows** (Approach D2) | **~1 week** | **New** | **Strategic** |
| Project-scoped task lists | 1-2 days | New | High |
| Cross-session resume | 1 day | New | High |
| Multi-project registry | 1-2 days | New | Medium |
| Ready computation | 0.5 day | New (reuses existing dependency logic) | High |
| Cross-project dependencies | 2-3 days | New | Low |
| Git-backed storage | 2-3 days | New | Medium |
| Session handoff | 1-2 days | New | Medium |
| Workflow templates | 3-5 days | New | Low |
| Compaction | 2-3 days | New | Low |
| Content hashing | 1 day | New | Low |

**Approach D1** (simple orchestrator): ~2-3 days. Vendor-independent, basic crash re-dispatch, no infrastructure cost.
**Approach D2** (Temporal): ~1 week setup + ongoing infrastructure (~$100/month Cloud or self-hosted). Automatic crash recovery with partial-work preservation, multi-agent routing, complete auditability.
**Total for must-haves** (D1 or D2 + project registry + resume + ready): ~6-10 days
**Total for everything new**: ~20-30 days

The strategic recommendation is Approach D (either D1 or D2, depending on crash recovery requirements). D1 is cheaper and simpler. D2 (Temporal) provides stronger guarantees -- choose it when losing progress is more expensive than the infrastructure cost.

---

## Pro vs Con: Is This Worth the Effort?

### Pros (Yes, Extend or Integrate)

1. **Real problem, real pain**: If you work across multiple projects with AI agents, the lack of cross-session task continuity and multi-project awareness is a genuine productivity drain. Each new session starts without knowledge of the broader backlog.

2. **Compounding value**: Persistent cross-project tasks accumulate value over time. Discovered issues, dependency insights, and workflow patterns compound across sessions instead of evaporating.

3. **Agent effectiveness**: Agents are more effective when they know what was done before. A ready-work list is better context than "figure out what needs doing" from scratch each time.

4. **The foundation already exists**: Claude Code already has the hardest parts -- persistence, file locking, atomic claiming, dependency enforcement, agent coordination. The remaining work is a thinner layer (multi-project registry, ready computation, resume workflow) than initially estimated.

5. **Beads exists as an option**: For the multi-project and cross-machine dimensions, Beads is a working implementation. The hybrid approach requires minimal custom code.

6. **Progressive adoption**: Start by using Claude Code's existing Tasks system as-is. Add Beads for multi-project visibility only if you need it. The investment scales with the need.

7. **[NEW] Temporal makes Approach D concrete, not abstract.** The previous version of this document recommended "build or adopt a deterministic orchestrator" without naming a specific system. Temporal is that system -- production-proven at OpenAI and Replit scale. It provides crash recovery, deterministic replay, vendor-independent agent dispatch, and complete auditability. The strategic recommendation is no longer hypothetical.

8. **[NEW] Production precedent exists.** OpenAI Codex runs agent execution on Temporal. Replit Agent 3 uses Temporal for agentic workflows. The OpenAI Agents SDK integrates with Temporal. These are not experiments -- they are production systems serving millions of users. The architectural pattern is validated.

### Cons (Not Worth It / Wait)

1. **Single-project use is already well-served**: If you work on one project at a time, Claude Code's existing swarm-mode Tasks system provides persistence, coordination, and dependencies out of the box. The gap only matters for multi-project and cross-session scenarios.

2. **Claude Code is actively evolving**: The swarm-mode task system is new and sophisticated. Anthropic is clearly investing in agent coordination infrastructure. Multi-project awareness, cross-session resume, and ready computation are natural next steps that may arrive natively. Building custom solutions now risks obsolescence.

3. **Two systems still means two mental models**: Even though Beads and Claude Code Tasks operate at different layers, developers still need to understand both. For solo developers, this overhead may exceed the coordination benefit.

4. **Agent instruction fragility**: The hybrid approach relies on Claude Code following CLAUDE.md instructions to bridge Beads and native Tasks. Agents sometimes forget instructions mid-session, skip steps, or deviate from protocols. The integration is only as reliable as the agent's adherence.

5. **Beads maturity risk**: Beads is actively evolving. The molecule/chemistry metaphors, while creative, add conceptual weight. APIs may change. Relying on a young project carries risk.

6. **Complexity budget**: Every tool in your workflow consumes attention. The question is not "would multi-project task management be useful?" (yes, obviously) but "is this the highest-leverage improvement right now?" If your bottleneck is something else (context limits, code quality, test coverage), task management infrastructure may not be the best investment.

7. **[REVISED] File persistence without discoverability is functionally equivalent to ephemerality.** The original version of this point argued that since tasks persist to disk, the gap is smaller than assumed. This is only half right. Tasks persist as files, but they are scoped to random session UUIDs and have no discovery mechanism. For the user, undiscoverable persistence is indistinguishable from no persistence. The `--resume` flag and `CLAUDE_CODE_TASK_LIST_ID` env var provide manual escape hatches, but require the user to proactively record UUIDs before they need them.

8. **Crash fragility is a real, observed failure mode.** Session crashes leave tasks stuck in `in_progress` with no cleanup. This is not a theoretical risk -- it was observed during this analysis. The `--resume` flag provides partial mitigation if the session ID is known, but the default experience after a crash is complete loss of task context.

9. **Internal API coupling risk.** Every tool built on `~/.claude/tasks/` or `sessions-index.json` depends on undocumented internals. Anthropic hasn't exposed tasks as an external interface -- any update can break tooling silently.

10. **Vendor lock-in.** Orchestration logic tied to Claude Code is non-portable. Codex, Gemini CLI, and open-source agent frameworks have different (or no) task systems. Switching providers means starting over.

11. **Agents are unreliable orchestrators.** Deciding what to do next, tracking progress, running tests, committing code -- these are deterministic operations. Agents forget instructions, skip steps, and hallucinate workflow states. Traditional software is simply better for this.

12. **[NEW] Temporal's infrastructure cost may not be justified.** Temporal Cloud starts at ~$100/month. Self-hosting requires a server, database, and operational attention. For a solo developer running short sessions, this is overkill. The crash recovery and deterministic replay that justify Temporal's complexity only matter when losing progress is genuinely expensive.

13. **[NEW] Temporal's learning curve is steep.** Determinism constraints (no `time.now()`, no `random()`, no non-deterministic library calls in Workflows), event sourcing, Workflow versioning, and Activity design require significant conceptual investment. Estimate 1-2 days for basic usage, 1-2 weeks for production confidence.

14. **[NEW] Temporal is overkill for simple orchestration.** If your "workflow" is "read a task list, invoke `claude --print`, check the result," a shell script (Approach D1) is simpler, cheaper, and faster to build. Temporal's value proposition -- crash recovery with partial-work preservation, multi-agent routing, Signals for human-in-the-loop -- only materializes for complex, long-running, multi-step workflows.

---

## Verdict (Revised)

**For everyone**: Adopt the principle that orchestration belongs in your own system, not in agent internals. Claude Code's task system is a useful within-session convenience -- let agents use it for self-organization. But your source of truth for "what needs doing" should be vendor-independent and deterministic. See `orchestration-principle.md`.

**For solo developers on one project**: If sessions are short and self-contained, Claude Code's built-in tasks are fine as-is. No external tooling needed. If work spans sessions, put your task list in a file *in your repo* (a TODO.md, a YAML plan file, whatever) that any agent or human can read. Temporal is overkill here.

**For developers managing multiple projects**: Build or adopt an orchestrator. Approach D1 (simple script) for basic needs. Approach D2 (Temporal) if crash recovery matters or you dispatch to multiple agent providers. Beads (Approach A/C) for the planning layer regardless. The discovery tool (Approach B) is a tactical option if you're committed to Claude Code, but understand the coupling cost.

**For teams with multiple agents across machines**: Temporal (Approach D2) is the strongest option -- it provides crash recovery, multi-agent routing via Task Queues, and vendor-independent dispatch. Beads (Approach A) handles the planning layer (persistent backlog, multi-project views). Claude Code Tasks handles the execution layer (in-session agent coordination). This is the full three-layer stack described in `task-system-comparison.md`.

**For production systems (CI/CD, deployment pipelines, batch processing)**: Temporal is the clear recommendation. These systems need guaranteed completion, auditability, and deterministic behavior. Agent crashes must not lose progress. A shell script orchestrator (D1) is insufficient for this use case.

**On the discovery tool**: The `claude-tasks` script (Approach B) is still a valid tactical choice if you accept the coupling to Claude Code internals. It's useful, it's fast to build, and it solves a real immediate problem. But it is not the strategic recommendation. Prefer Approach D for long-term investment.

---

## Decision Matrix (Revised)

| Your Situation | Recommendation |
|---|---|
| Any developer using AI agents | Adopt the orchestration principle: your system owns the workflow, agents are workers |
| 1 project, short sessions | Claude Code's built-in tasks are fine. Keep a task list in your repo as fallback. |
| 1 project, cross-session work | Approach D1 (simple orchestrator) or task list in repo. Approach B if committed to Claude. |
| 2-3 projects | Approach D1 + Beads (Approach C). Temporal (D2) if crash recovery needed. |
| Multiple agents across machines | Temporal (Approach D2) + Beads (Approach C) for full stack. |
| Multiple agent providers (Claude + Codex + open source) | Temporal (Approach D2) -- Activities wrap any CLI agent. |
| Production system (CI/CD, deployments) | Temporal (Approach D2) -- guaranteed completion, auditability, automatic retry. |
| Crash recovery is critical | Temporal (Approach D2) -- only option with automatic partial-work recovery. |
| Budget is zero | Approach D1 (script) + Beads (free, git-backed). |
| Committed to Claude Code only | Approach B (discovery tool) as tactical mitigation. Understand the coupling cost. |

---

*This document is part of the task-analysis project. See `temporal-analysis.md` for the Temporal deep-dive, `task-system-comparison.md` for the 3-way comparison, `orchestration-principle.md` for the underlying principle, and `beads-task-system.md` for the Beads deep-dive.*
