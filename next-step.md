# Next Step: Extending Claude Code Tasks Toward Multi-Project Management

## The Question

> If I wanted Beads-like multi-project task management today, working primarily with Claude Code, what should I do?

This document explores what it would take to bridge the gap between Claude Code's task systems and Beads' persistent multi-project infrastructure, evaluates the practical approaches available, and concludes with a pro/con assessment of whether the effort is worthwhile.

---

## Current State: What Claude Code Actually Has

Before evaluating approaches, we need an accurate picture of what Claude Code v2.1.34 already provides. The source analysis reveals significantly more capability than surface-level inspection suggests -- not just in task persistence and coordination, but in agent context management, inter-agent communication, planning workflows, and completion verification.

### What Already Exists

**Two mutually exclusive task systems:**

1. **TodoWrite (solo mode)**: In-memory, per-session, per-agent. Three fields (content, status, activeForm). Primarily ephemeral, though a disk persistence layer exists at `~/.claude/todos/` (6,062 files observed in a live system, 98% empty/cleared). Enabled when the Tasks feature gate is off.

2. **Tasks (swarm/team mode)**: File-persisted JSON at `~/.claude/tasks/{team-name}/`. Full CRUD tools (TaskCreate, TaskGet, TaskUpdate, TaskList). Dependency tracking (blocks/blockedBy) with enforcement at claim time. This is enabled by default in interactive sessions.

**Multi-agent coordination already built in:**
- Team management tools: TeamCreate, TeamDelete, SendMessage (broadcast + direct messaging)
- Atomic task claiming with `proper-lockfile` file locks
- Agent busy checks (prevents hoarding multiple tasks)
- Five structured claim-failure reasons (task_not_found, already_claimed, already_resolved, blocked, agent_busy)
- Auto-unassign on agent shutdown with notification for reassignment
- Background task system with typed IDs (local bash "b", local agent "a", remote agent "r", in-process teammate "t")
- Output files per background task with incremental read support
- In-process teammate system (agents sharing the same Node.js process)

**Inter-agent communication:**
- SendMessage tool with broadcast and direct message types
- Automatic idle notifications when teammates finish their turns
- Peer DM visibility (summaries of direct messages between other agents included in idle notifications)
- Teammate mailbox attachment delivers received messages to agent context
- Team lead receives idle notifications from all teammates
- Notification queue with task notifications prioritized over regular commands

**Planning and verification:**
- Plan mode (EnterPlanMode/ExitPlanMode) with structured plan-approve-execute cycle
- Plans stored as markdown at `~/.claude/plans/{adjective-gerund-noun}.md`
- Team-lead approval for plans in team mode (plan_approval_request/response)
- Plan mode reminders injected every 5 turns (alternating full/sparse)
- Completion verification hooks (agent stop hooks) -- sub-agents spawned to validate task completion
- Hook agents run non-interactive with up to 50 turns, return `{ ok: true/false, reason }`

**Agent context management (29 attachment types):**
- `task_progress` / `task_status` / `unified_tasks`: Background task monitoring
- `task_reminder` / `todo_reminder`: Automatic nudges after 10+ idle turns
- `team_context`: Team membership info (first turn only)
- `teammate_mailbox`: Messages from other agents
- `plan_mode` / `plan_mode_exit`: Plan mode context injection
- `changed_files`, `lsp_diagnostics`, `diagnostics`: Code awareness
- `budget_usd`, `token_usage`: Resource tracking
- Plus 17 more attachment types covering hooks, MCP, IDE, skills, memory, etc.

**Persistence mechanics:**
- Individual JSON files per task (pretty-printed, schema-validated on read)
- `.highwatermark` file ensures IDs never reuse, even after deletion or clear-all
- `.lock` file for global operations; per-task file locks for claiming

### What Does NOT Exist

1. **Multi-project awareness**: Tasks are keyed by team name, not by project/directory. No concept of "which repo does this task belong to?"
2. **Cross-session ready computation**: No `bd ready` equivalent. Task files persist, but there is no tool or workflow to ask "what work is available across all my projects?"
3. **Reusable workflow templates**: No formulas, no reusable patterns. Plan mode is ad-hoc per session, not templated. No gate steps at arbitrary workflow points (only at task completion).
4. **Git-native storage**: Local filesystem only. No version control, no distribution to other machines
5. **Compaction / memory decay**: Completed tasks accumulate as JSON files indefinitely. No long-term context management.
6. **Rich dependency types**: Only blocks/blockedBy. No conditional-blocks, waits-for, related, discovered-from
7. **Agent self-monitoring across sessions**: No agent-as-bead pattern. No stuck/dead detection across sessions (idle notifications only work within a session)
8. **Content hashing**: No deduplication or convergent state detection
9. **Cross-machine coordination**: File locking is local-only

---

## The Actual Gap

Given what Claude Code already has, the gap is narrower than initially assumed but still real in specific dimensions:

| Capability | Status | Gap Size |
|-----------|--------|----------|
| Task persistence | **Already exists** (swarm mode) | None |
| Multi-agent coordination | **Already exists** (file locking, claiming, busy checks) | None for same-machine |
| Inter-agent communication | **Already exists** (SendMessage, idle notifications, mailbox) | None for same-machine |
| Planning workflow | **Already exists** (plan mode with team-lead approval) | None (but ad-hoc, not templated) |
| Completion verification | **Already exists** (agent stop hooks, sub-agent validation) | None for completion gates |
| Agent context management | **Already exists** (29 attachment types, reminders, progress) | None for in-session context |
| Dependency tracking | **Already exists** (blocks/blockedBy, enforced at claim) | Small (lacks rich types) |
| Auto-incrementing IDs with highwatermark | **Already exists** | None |
| Multi-project awareness | Does not exist | **Large** |
| Cross-session ready computation | Does not exist | **Large** |
| Reusable workflow templates | Does not exist | **Large** |
| Git-native storage / distribution | Does not exist | **Medium-Large** |
| Long-term compaction | Does not exist | **Medium** |
| Cross-machine agent coordination | Does not exist | **Medium** |
| Arbitrary gate steps (not just completion) | Does not exist | **Small-Medium** |

The gap is not about basic infrastructure (persistence, locking, ownership, dependencies, messaging, planning, context management) -- Claude Code already has all of that, and more than previously recognized. The gap is about **scope** (multi-project), **reusability** (workflow templates), **distribution** (git, cross-machine), and **longevity** (compaction, cross-session resume).

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
- (-) Two task systems running simultaneously -- but this is less problematic now since Claude Code's Tasks system handles in-session coordination (with SendMessage, idle notifications, plan mode, and completion hooks), while Beads handles the cross-session/cross-project layer
- (-) Beads is a young project -- API and concepts may evolve
- (-) Beads has no inter-agent messaging or context injection -- Claude Code fills that role

### Approach B: Build a Thin Multi-Project Layer on Top of Claude Code's Existing Tasks

**What**: Since Claude Code already has persistence, locking, dependencies, coordination, messaging, planning, and context management, build only the missing pieces: multi-project awareness, cross-session resume, and ready computation.

**Implementation sketch**:

Claude Code's task files already live at `~/.claude/tasks/{team-name}/`. The key insight is that the team name is configurable -- you can set `CLAUDE_CODE_TASK_LIST_ID` to any value, or use different team names per project. The missing pieces are:

1. **Project registry**: A `~/.claude/projects.json` mapping project names to directories and their corresponding task list IDs
2. **Ready computation script**: A shell script or Claude Code hook that scans all registered task list directories, reads the JSON files, filters for unblocked pending tasks, and presents them
3. **Session resume instructions**: CLAUDE.md instructions that tell the agent to check the project's task list at session start using `TaskList`
4. **Cross-project dependency convention**: Use the `metadata` field to store cross-project references (e.g., `metadata: { "cross_dep": "other-project:3" }`)

**What you do NOT need to build** (because Claude Code already has it):
- Task persistence (already JSON files on disk)
- File locking (already `proper-lockfile`)
- Ownership and claiming (already `oBA()` / `SyD()`)
- Dependency enforcement (already checked at claim time)
- Agent coordination (already team system with auto-unassign)
- Inter-agent messaging (already SendMessage with broadcast + direct)
- Planning workflow (already plan mode with team-lead approval)
- Completion verification (already agent stop hooks)
- Agent context management (already 29 attachment types with reminders)

**Effort**: Low. ~1 day of scripting. A `projects.json` file, a `ready.sh` script, and CLAUDE.md instructions.

**Tradeoffs**:
- (+) Builds on existing Claude Code infrastructure -- no external binary
- (+) Minimal new code since the hard parts (persistence, locking, coordination, messaging, planning, context management) already exist
- (+) Stays within the Claude Code ecosystem -- retains all 29 attachment types, reminders, idle notifications, plan mode, and completion hooks
- (+) Task files are already human-readable JSON
- (-) No git distribution -- still local filesystem only
- (-) No daemon, no background sync
- (-) No compaction
- (-) No reusable workflow templates or arbitrary gate steps
- (-) The ready computation is a separate script, not integrated into the agent loop
- (-) Cross-project dependencies via metadata are a convention, not enforced

### Approach C: Hybrid -- Beads for Multi-Project Planning, Claude Code for Execution

**What**: Use Beads as the persistent multi-project backend. Use Claude Code's native Tasks system for in-session agent coordination. The two systems operate at different layers.

**Implementation**:

1. Install Beads and init in projects
2. Write CLAUDE.md instructions that teach Claude Code to:
   - Start sessions with `bd ready --json` to load the backlog
   - Claim Beads issues with `bd update --claim`
   - Use Claude Code's TaskCreate/TaskUpdate for in-session subtask decomposition among team agents
   - Use SendMessage for agent communication during execution
   - Use plan mode for designing implementation approaches before writing code
   - Close Beads issues when work completes (after completion hooks pass)
   - Run `bd sync` at session end
3. Optionally write a Claude Code hook that auto-syncs Beads on session start/end

**Why this works better than we initially thought**: The "two systems" problem is less severe because the systems naturally layer and are now recognized as **complementary rather than overlapping**:

- **Beads** provides: multi-project scope, cross-session continuity, git-native distribution, workflow templates, compaction, rich dependency types
- **Claude Code** provides: in-session coordination, inter-agent messaging (SendMessage, idle notifications), plan mode, completion verification hooks, 29-type context injection, automatic task reminders, delegate mode

Neither system covers both layers. Beads has no inter-agent messaging, no context injection, no automatic reminders, and no plan mode. Claude Code has no multi-project awareness, no git distribution, no workflow templates, and no compaction. They address fundamentally different concerns.

**Effort**: Low. Mostly documentation/prompting.

**Tradeoffs**:
- (+) Best of both worlds: persistent multi-project planning + rich in-session agent infrastructure
- (+) Minimal custom code
- (+) Each system does what it is best at -- the complementarity is cleaner than previously assessed
- (+) Can progressively adopt Beads features (molecules, formulas) as needed
- (+) Claude Code's context management (reminders, progress, idle notifications) keeps agents effective during Beads task execution
- (-) Two systems to understand (but they operate at genuinely different layers with minimal overlap)
- (-) Relies on agent following CLAUDE.md instructions to bridge Beads and native Tasks
- (-) Beads is still an external dependency

---

## Revised Recommendation

**Start with Approach C (Hybrid), but with even less urgency than the previous revision suggested.**

Here is why the calculus has shifted further:

### What changed from the previous revision:

1. **Claude Code's within-session capabilities are richer than previously documented**. The addition of inter-agent messaging (SendMessage with broadcast/direct), idle notifications with peer DM visibility, plan mode with team-lead approval, completion verification hooks via sub-agents, and the 29-type attachment system means Claude Code's in-session agent infrastructure is substantially more capable than a task CRUD + file locking system. It is closer to an agent operating environment.

2. **The complementarity with Beads is cleaner**. Previously, the "two systems" concern was about overlap. With the full picture of both systems, the overlap is actually minimal. Beads provides cross-session/cross-project/cross-machine infrastructure with no real-time agent features. Claude Code provides rich real-time agent infrastructure with no cross-session/cross-project features. They complement rather than compete.

3. **The "workflow gap" is narrower but not closed**. Claude Code's plan mode provides a formal plan-approve-execute cycle, and completion hooks provide quality gates. These are not a workflow engine (no templates, no arbitrary gates, no formulas), but they reduce the urgency of adopting Beads' molecule system for teams that primarily need planning and verification rather than repeatable multi-step workflows.

4. **The "context management" advantage is new**. Claude Code's 29-type attachment system, automatic task reminders, and idle notifications represent a category of capability that Beads does not have at all. This is a genuine Claude Code advantage that was not previously captured in the gap analysis.

### What stayed the same:

1. **Multi-project awareness is still missing** -- the biggest gap
2. **Cross-session ready computation is still missing** -- `bd ready` has no equivalent
3. **Reusable workflow templates are still missing** -- plan mode is ad-hoc, not templated
4. **Git-native storage is still missing** -- no distribution, no version history
5. **Compaction is still missing** -- long-lived backlogs will grow unbounded

### Updated concrete steps:

1. **If you need multi-project management today**: Install Beads, use Approach C. The hybrid works naturally because the systems are genuinely complementary (Beads = durable state + distribution, Claude Code = real-time agent context + coordination).

2. **If you primarily work on one project with multiple agents**: You do not need Beads. Claude Code's swarm-mode Tasks system provides persistence, coordination, dependency enforcement, inter-agent messaging, plan mode, completion verification, and rich context management out of the box.

3. **If you want cross-session continuity on a single project**: Consider Approach B -- a thin script that reads `~/.claude/tasks/{team}/` files and computes ready work. This is a ~1 day effort and does not require Beads. You keep all of Claude Code's agent infrastructure (reminders, idle notifications, plan mode, completion hooks) without adding an external dependency.

4. **If you want to wait**: Claude Code's task infrastructure is actively being built out. The subsystem count (task CRUD + team management + messaging + plan mode + completion hooks + reminder system + idle notifications + 29-type attachment system + in-process teammates + delegate mode) suggests substantial ongoing investment. Multi-project awareness, cross-session resume, and ready computation are natural next steps that may arrive natively.

---

## What Would It Take to Extend Claude Code Natively?

Given that Claude Code already has persistence, locking, ownership, dependencies, multi-agent coordination, inter-agent messaging, plan mode, completion hooks, context injection, and agent reminders, the remaining extensions are a focused set:

### Must-Have Extensions (for multi-project use)

1. **Project-Scoped Task Lists**: Map task list IDs to project directories. Currently, the task list ID comes from team name or `CLAUDE_CODE_TASK_LIST_ID`. Adding a project -> task list mapping would enable multi-project awareness without changing the underlying storage.

2. **Cross-Session Resume**: A `/resume-tasks` command or auto-load behavior. The task files already persist -- the missing piece is a workflow that reads them at session start and presents them to the agent. The attachment system could easily support a `session_resume` type.

3. **Ready Computation**: A `TaskReady` tool that computes tasks with no open blockers across all registered projects/task lists. The dependency enforcement code already exists in `oBA()` -- it just needs to be exposed as a standalone query.

4. **Multi-Project Registry**: A configuration in `~/.claude/settings.json` that lists project directories and their task list IDs. A `/tasks-all` command to show tasks across projects.

### Nice-to-Have Extensions

5. **Git-Backed Storage**: Export task JSON files to a git-tracked location (e.g., `.claude-tasks/` in each repo). This would enable distribution and version history.

6. **Cross-Project Dependencies**: Allow `blockedBy` to reference tasks in other task lists using a `{listId}:{taskId}` format.

7. **Session Handoff**: Auto-generate a handoff summary when a session ends, stored alongside persisted tasks. Next session loads the summary for context. Could leverage the existing plan file infrastructure at `~/.claude/plans/`.

8. **Workflow Templates**: YAML files defining reusable task decomposition patterns. A `/workflow release` command that creates a predefined set of tasks with dependencies. Could integrate with plan mode as a "templated plan."

9. **Compaction**: Summarize completed tasks periodically to keep the task directory manageable.

10. **Content Hashing**: Add a content hash to task JSON for deduplication and convergent state detection across machines.

11. **Arbitrary Gate Steps**: Extend the completion verification hooks to support gates at arbitrary workflow points (not just task completion). The sub-agent hook infrastructure is already there.

### Revised Estimated Effort

Note: The "already exists" list has grown substantially from the original estimate.

| Extension | Effort | Status |
|-----------|--------|--------|
| Task persistence (file-based) | -- | **Already exists** |
| File locking for concurrency | -- | **Already exists** |
| Ownership and claiming | -- | **Already exists** |
| Dependency enforcement | -- | **Already exists** |
| Agent busy check | -- | **Already exists** |
| Auto-unassign on shutdown | -- | **Already exists** |
| Background task system | -- | **Already exists** |
| Inter-agent messaging | -- | **Already exists** (SendMessage, broadcast + direct) |
| Idle notifications | -- | **Already exists** (automatic, with peer DM visibility) |
| Planning workflow | -- | **Already exists** (plan mode with team-lead approval) |
| Completion verification | -- | **Already exists** (agent stop hooks, sub-agents) |
| Agent context management | -- | **Already exists** (29 attachment types) |
| Task reminders | -- | **Already exists** (automatic after 10+ idle turns) |
| In-process teammates | -- | **Already exists** (shared Node.js process) |
| Delegate mode | -- | **Already exists** |
| Project-scoped task lists | 1-2 days | New |
| Cross-session resume | 1 day | New |
| Multi-project registry | 1-2 days | New |
| Ready computation | 0.5 day | New (reuses existing dependency logic) |
| Cross-project dependencies | 2-3 days | New |
| Git-backed storage | 2-3 days | New |
| Session handoff | 1-2 days | New |
| Workflow templates | 3-5 days | New |
| Compaction | 2-3 days | New |
| Content hashing | 1 day | New |
| Arbitrary gate steps | 1-2 days | New (extends existing hook infrastructure) |

**Total for must-haves** (project registry, resume, ready, multi-project): ~4-6 days
**Total for everything new**: ~16-25 days

The "already exists" list now includes 15 capabilities. The remaining 11 extensions are all about scope expansion (multi-project, cross-session, distribution, longevity) rather than core infrastructure.

---

## Pro vs Con: Is This Worth the Effort?

### Pros (Yes, Extend or Integrate)

1. **Real problem, real pain**: If you work across multiple projects with AI agents, the lack of cross-session task continuity and multi-project awareness is a genuine productivity drain. Each new session starts without knowledge of the broader backlog.

2. **Compounding value**: Persistent cross-project tasks accumulate value over time. Discovered issues, dependency insights, and workflow patterns compound across sessions instead of evaporating.

3. **Agent effectiveness**: Agents are more effective when they know what was done before. A ready-work list is better context than "figure out what needs doing" from scratch each time. Claude Code's attachment system is excellent at keeping agents aware within a session, but that awareness resets on session end.

4. **The foundation already exists and is richer than recognized**: Claude Code now has 15+ documented subsystems covering persistence, coordination, messaging, planning, verification, and context management. The remaining work is a thin layer focused on scope expansion (multi-project registry, ready computation, resume workflow) rather than core infrastructure.

5. **Beads exists as an option**: For the multi-project and cross-machine dimensions, Beads is a working implementation. The hybrid approach requires minimal custom code, and the two systems are genuinely complementary.

6. **Progressive adoption**: Start by using Claude Code's existing Tasks system as-is (which is more capable than most users realize). Add Beads for multi-project visibility only if you need it. The investment scales with the need.

### Cons (Not Worth It / Wait)

1. **Single-project use is already very well-served**: If you work on one project at a time, Claude Code's existing swarm-mode Tasks system provides persistence, coordination, dependencies, messaging, planning, verification, context injection, and reminders out of the box. The gap only matters for multi-project and cross-session scenarios.

2. **Claude Code is actively evolving and investing heavily**: The sheer number of subsystems (task CRUD, team management, SendMessage, plan mode, completion hooks, reminder system, idle notifications, attachment system, in-process teammates, delegate mode, notification queue) indicates substantial ongoing investment in agent coordination. Multi-project awareness, cross-session resume, and ready computation are natural next steps that may arrive natively. Building custom solutions now risks obsolescence.

3. **Two systems still means two mental models**: Even though Beads and Claude Code Tasks operate at different layers with minimal overlap, developers still need to understand both. For solo developers, this overhead may exceed the coordination benefit.

4. **Agent instruction fragility**: The hybrid approach relies on Claude Code following CLAUDE.md instructions to bridge Beads and native Tasks. Claude Code's built-in task reminder system (10-turn nudges) helps keep the agent on track, but cross-system protocols (like running `bd sync` at session end) are not enforced by the system.

5. **Beads maturity risk**: Beads is actively evolving. The molecule/chemistry metaphors, while creative, add conceptual weight. APIs may change. Relying on a young project carries risk.

6. **Complexity budget**: Every tool in your workflow consumes attention. The question is not "would multi-project task management be useful?" (yes, obviously) but "is this the highest-leverage improvement right now?" If your bottleneck is something else (context limits, code quality, test coverage), task management infrastructure may not be the best investment.

7. **The "capability gap" keeps narrowing**: Each analysis of Claude Code reveals more capabilities than the previous one recognized. Plan mode, completion hooks, SendMessage, idle notifications, and the 29-type attachment system were not in the original gap analysis. The trajectory suggests the remaining gaps (multi-project, ready computation, templates) may also close over time.

---

## Verdict (Revised)

**For solo developers on one project**: You do not need anything beyond Claude Code's built-in Tasks system. It provides persistence, coordination, dependencies, inter-agent messaging, plan mode, completion verification, context injection, and automatic reminders. This is a substantially more complete system than it appears from the tool definitions alone.

**For solo developers on 2-3 projects**: The gap that matters is multi-project awareness and cross-session ready computation. Try Approach B first (a thin script that reads task directories and computes ready work). You keep all of Claude Code's agent infrastructure. If that is not enough, move to Approach C (add Beads for multi-project planning).

**For developers managing 3+ active projects with AI agents**: Try the hybrid approach (Approach C) for one week. Install Beads in stealth mode, add CLAUDE.md instructions, and see if `bd ready` across projects changes how you start sessions. The complementarity between Beads (durable state + distribution) and Claude Code (real-time agent context + coordination) is genuine.

**For teams with multiple AI agents across machines**: Beads (or something like it) is close to mandatory. Claude Code's coordination is excellent for same-machine agents but has no cross-machine story. Git-based or network-based state sharing is required. Adopt Beads fully (Approach A), using Claude Code's agent infrastructure for in-session coordination.

**For everyone: consider waiting with even more confidence than before**. Claude Code v2.1.34's task system is not a simple TODO list -- it is a 15+ subsystem agent coordination environment. The trajectory clearly shows ongoing investment. Multi-project and cross-session features may arrive natively without requiring external tooling.

---

## Decision Matrix (Revised)

| Your Situation | Recommendation |
|---------------|---------------|
| 1 project, any number of agents | Use Claude Code's built-in Tasks. It is richer than it appears. No extension needed. |
| 2-3 projects, daily Claude Code use | Try Approach B (thin multi-project script) for a week |
| 3+ projects, single developer | Try Approach C (Beads hybrid) for a week |
| 3+ projects, multiple agents, same machine | Approach C (Beads for planning, Claude Code for execution + coordination) |
| Multiple agents across machines | Adopt Beads fully (Approach A) |
| Waiting for native support | Use Claude Code's Tasks as-is. It already has messaging, plan mode, completion hooks, reminders, and 29-type context injection. Write CLAUDE.md handoff notes. |
