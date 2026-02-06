# Next Step: Extending Claude Code Tasks Toward Multi-Project Management

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
| Task persistence | **Already exists** (swarm mode) | None |
| Multi-agent coordination | **Already exists** (file locking, claiming, busy checks) | None for same-machine |
| Dependency tracking | **Already exists** (blocks/blockedBy, enforced at claim) | Small (lacks rich types) |
| Auto-incrementing IDs with highwatermark | **Already exists** | None |
| Multi-project awareness | Does not exist | **Large** |
| Cross-session ready computation | Does not exist | **Large** |
| Workflow templates | Does not exist | **Large** |
| Git-native storage / distribution | Does not exist | **Medium-Large** |
| Compaction | Does not exist | **Medium** |
| Cross-machine agent coordination | Does not exist | **Medium** |

The gap is not about basic infrastructure (persistence, locking, ownership, dependencies) -- Claude Code already has all of that. The gap is about **scope** (multi-project), **workflow** (templates, ready computation), and **distribution** (git, cross-machine).

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

### Approach B: Build a Thin Multi-Project Layer on Top of Claude Code's Existing Tasks

**What**: Since Claude Code already has persistence, locking, dependencies, and coordination, build only the missing pieces: multi-project awareness, cross-session resume, and ready computation.

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

**Effort**: Low. ~1 day of scripting. A `projects.json` file, a `ready.sh` script, and CLAUDE.md instructions.

**Tradeoffs**:
- (+) Builds on existing Claude Code infrastructure -- no external binary
- (+) Minimal new code since the hard parts (persistence, locking, coordination) already exist
- (+) Stays within the Claude Code ecosystem
- (+) Task files are already human-readable JSON
- (-) No git distribution -- still local filesystem only
- (-) No daemon, no background sync
- (-) No compaction
- (-) No workflow templates or gate steps
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
- (-) Two systems to understand (but they operate at different layers)
- (-) Relies on agent following CLAUDE.md instructions correctly
- (-) Beads is still an external dependency

---

## Revised Recommendation

**Start with Approach C (Hybrid), but with less urgency than originally suggested.**

Here is why the calculus has changed:

### What changed from the original analysis:

1. **Claude Code is not ephemeral** (in swarm mode). The original recommendation was partly driven by the belief that Claude Code tasks vanish on session end. They do not -- they persist as JSON files. This means the "persistence gap" is smaller than assumed.

2. **Claude Code already has multi-agent coordination**. The original analysis positioned this as a major Beads advantage. Claude Code actually has file-locking, atomic claiming, agent busy checks, and auto-unassign -- a robust same-machine coordination system. The advantage Beads has is specifically cross-machine coordination.

3. **The "build persistence" approach (old Approach B) is obsolete**. The original Approach B was "build a lightweight persistence layer for Claude Code Tasks." This is unnecessary -- Claude Code already has persistence. The real missing piece is multi-project awareness and cross-session workflow, which is a thinner layer than previously estimated.

### What stayed the same:

1. **Multi-project awareness is still missing** -- the biggest gap
2. **Cross-session ready computation is still missing** -- `bd ready` has no equivalent
3. **Workflow templates are still missing** -- no formulas, no reusable patterns
4. **Git-native storage is still missing** -- no distribution, no version history
5. **Compaction is still missing** -- long-lived backlogs will grow unbounded

### Updated concrete steps:

1. **If you need multi-project management today**: Install Beads, use Approach C. The hybrid works naturally because the systems layer (Beads = planning, Claude Code = execution).

2. **If you primarily work on one project with multiple agents**: You may not need Beads at all. Claude Code's swarm-mode Tasks already provide persistence, coordination, and dependency enforcement. Just use it as-is.

3. **If you want cross-session continuity on a single project**: Consider Approach B -- a thin script that reads `~/.claude/tasks/{team}/` files and computes ready work. This is a ~1 day effort and does not require Beads.

4. **If you want to wait**: Claude Code's task infrastructure is actively being built out. The swarm-mode system is new and sophisticated. Multi-project and cross-session features may follow.

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

| Extension | Effort | Status |
|-----------|--------|--------|
| Task persistence (file-based) | -- | **Already exists** |
| File locking for concurrency | -- | **Already exists** |
| Ownership and claiming | -- | **Already exists** |
| Dependency enforcement | -- | **Already exists** |
| Agent busy check | -- | **Already exists** |
| Auto-unassign on shutdown | -- | **Already exists** |
| Background task system | -- | **Already exists** |
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

**Total for must-haves** (project registry, resume, ready, multi-project): ~4-6 days
**Total for everything new**: ~14-22 days

This is a significant revision from the original estimate, which included ~3-5 days for persistence and coordination infrastructure that already exists.

---

## Pro vs Con: Is This Worth the Effort?

### Pros (Yes, Extend or Integrate)

1. **Real problem, real pain**: If you work across multiple projects with AI agents, the lack of cross-session task continuity and multi-project awareness is a genuine productivity drain. Each new session starts without knowledge of the broader backlog.

2. **Compounding value**: Persistent cross-project tasks accumulate value over time. Discovered issues, dependency insights, and workflow patterns compound across sessions instead of evaporating.

3. **Agent effectiveness**: Agents are more effective when they know what was done before. A ready-work list is better context than "figure out what needs doing" from scratch each time.

4. **The foundation already exists**: Claude Code already has the hardest parts -- persistence, file locking, atomic claiming, dependency enforcement, agent coordination. The remaining work is a thinner layer (multi-project registry, ready computation, resume workflow) than initially estimated.

5. **Beads exists as an option**: For the multi-project and cross-machine dimensions, Beads is a working implementation. The hybrid approach requires minimal custom code.

6. **Progressive adoption**: Start by using Claude Code's existing Tasks system as-is. Add Beads for multi-project visibility only if you need it. The investment scales with the need.

### Cons (Not Worth It / Wait)

1. **Single-project use is already well-served**: If you work on one project at a time, Claude Code's existing swarm-mode Tasks system provides persistence, coordination, and dependencies out of the box. The gap only matters for multi-project and cross-session scenarios.

2. **Claude Code is actively evolving**: The swarm-mode task system is new and sophisticated. Anthropic is clearly investing in agent coordination infrastructure. Multi-project awareness, cross-session resume, and ready computation are natural next steps that may arrive natively. Building custom solutions now risks obsolescence.

3. **Two systems still means two mental models**: Even though Beads and Claude Code Tasks operate at different layers, developers still need to understand both. For solo developers, this overhead may exceed the coordination benefit.

4. **Agent instruction fragility**: The hybrid approach relies on Claude Code following CLAUDE.md instructions to bridge Beads and native Tasks. Agents sometimes forget instructions mid-session, skip steps, or deviate from protocols. The integration is only as reliable as the agent's adherence.

5. **Beads maturity risk**: Beads is actively evolving. The molecule/chemistry metaphors, while creative, add conceptual weight. APIs may change. Relying on a young project carries risk.

6. **Complexity budget**: Every tool in your workflow consumes attention. The question is not "would multi-project task management be useful?" (yes, obviously) but "is this the highest-leverage improvement right now?" If your bottleneck is something else (context limits, code quality, test coverage), task management infrastructure may not be the best investment.

7. **The "persistence gap" illusion**: Much of the original motivation for this analysis was the belief that Claude Code tasks are ephemeral. They are not (in swarm mode). This removes the strongest argument for building or adopting an alternative. The remaining gaps (multi-project, ready computation, templates) are real but may not be urgent for most developers.

---

## Verdict (Revised)

**For solo developers on one project**: You probably do not need anything beyond Claude Code's built-in Tasks system. It already persists to disk, handles agent coordination, and enforces dependencies. Use it as-is.

**For solo developers on 2-3 projects**: The gap that matters is multi-project awareness and cross-session ready computation. Try Approach B first (a thin script that reads task directories and computes ready work). If that is not enough, move to Approach C (add Beads for multi-project planning).

**For developers managing 3+ active projects with AI agents**: Try the hybrid approach (Approach C) for one week. Install Beads in stealth mode, add CLAUDE.md instructions, and see if `bd ready` across projects changes how you start sessions.

**For teams with multiple AI agents across machines**: Beads (or something like it) is close to mandatory. Claude Code's coordination is excellent for same-machine agents, but cross-machine coordination requires git-based or network-based state sharing. Adopt Beads fully (Approach A).

**For everyone: consider waiting**. Claude Code v2.1.34's swarm-mode Tasks system is new, capable, and clearly under active development. The trajectory suggests Anthropic is building toward richer agent coordination. Features like multi-project awareness, cross-session resume, and ready computation may arrive natively.

---

## Decision Matrix (Revised)

| Your Situation | Recommendation |
|---------------|---------------|
| 1 project, any number of agents | Use Claude Code's built-in Tasks. No extension needed. |
| 2-3 projects, daily Claude Code use | Try Approach B (thin multi-project script) for a week |
| 3+ projects, single developer | Try Approach C (Beads hybrid) for a week |
| 3+ projects, multiple agents, same machine | Approach C (Beads for planning, Claude Code for execution) |
| Multiple agents across machines | Adopt Beads fully (Approach A) |
| Waiting for native support | Use Claude Code's Tasks as-is. Write CLAUDE.md handoff notes. |
