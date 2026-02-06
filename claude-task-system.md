# Claude Code Task System

## Executive Summary

The Claude Code **task system** is a mechanism by which the Claude Code agent manages structured work items during coding sessions. It provides tools for creating, tracking, updating, and listing tasks with support for dependencies (blocking/blocked-by relationships), lifecycle state transitions, file-level locking for concurrent access, and progress display. The task system enables Claude Code to break complex user requests into discrete, trackable units of work, coordinate execution across multi-agent teams, and display progress to the user via a terminal status line.

This document is based on **actual source code analysis** of the Claude Code v2.1.34 production binary (build 2026-02-06T06:37:05Z), extracted via `strings` on the Mach-O arm64 Bun-compiled binary with pattern-matched deobfuscation. The companion source analysis document is at `task-system-source-analysis.md`. All code snippets are deobfuscated reconstructions with variable names mapped to their functional equivalents.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Two Mutually Exclusive Systems: TodoWrite vs Tasks](#two-mutually-exclusive-systems-todowrite-vs-tasks)
3. [Feature Gate: The sq() Function](#feature-gate-the-sq-function)
4. [TodoWrite System (Solo Mode)](#todowrite-system-solo-mode)
5. [Task System (Team/Swarm Mode)](#task-system-teamswarm-mode)
6. [Task Persistence Layer](#task-persistence-layer)
7. [Task CRUD Tools](#task-crud-tools)
8. [Task Dependency Resolution](#task-dependency-resolution)
9. [Task Claiming and Ownership](#task-claiming-and-ownership)
10. [Highwatermark: Monotonic ID Generation](#highwatermark-monotonic-id-generation)
11. [Task Lifecycle and State Machine](#task-lifecycle-and-state-machine)
12. [Background Task System](#background-task-system)
13. [Team System](#team-system)
14. [Task Notifications and Queue System](#task-notifications-and-queue-system)
15. [UI Rendering](#ui-rendering)
16. [Environment Variables and Configuration](#environment-variables-and-configuration)
17. [Obfuscated Symbol Map](#obfuscated-symbol-map)

---

## Architecture Overview

Claude Code has **two entirely separate task-tracking systems** that are **mutually exclusive** -- they are never both active at the same time. They are not "generations" or "versions" of each other; they serve different operational modes:

| System | Tool(s) | Storage | Scope | Active When |
|--------|---------|---------|-------|-------------|
| **TodoWrite** | `TodoWrite` | In-memory app state (`AppState.todos`) | Per-agent, per-session | Tasks feature **disabled** (solo mode) |
| **Tasks** | `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList` | Filesystem JSON files in `~/.claude/tasks/{team-name}/` | Shared across team agents | Tasks feature **enabled** (team/swarm mode) |

The broader "task" concept in the app state (`AppState.tasks`) also encompasses **background processes**: local bash tasks, local agent tasks, remote agents, and in-process teammates. These form a separate subsystem from the shared task list described above.

### High-Level Architecture

```
+------------------+     +------------------+     +--------------------+
|   User Request   |---->|  Claude Agent    |---->|   Task Store       |
|  (natural lang)  |     |  (main thread)   |     | (filesystem JSON   |
+------------------+     +--------+---------+     |  OR in-memory)     |
                                  |               +--------+-----------+
                          +-------v-------+                |
                          |  Task Tools   |        +-------v---------+
                          | (TodoWrite    |        |  Status Line    |
                          |  OR Task*)    |        |  Renderer       |
                          +-------+-------+        |  (terminal UI)  |
                                  |                +-----------------+
                     +------------+------------+
                     |            |             |
              +------v---+ +-----v----+ +------v------+
              | Teammate | | Teammate | | Background  |
              | Agent 1  | | Agent 2  | | Bash Task   |
              +----------+ +----------+ +-------------+
```

### Data Flow

1. **Feature Gate**: The `sq()` function determines which system is active based on `CLAUDE_CODE_ENABLE_TASKS` and session interactivity
2. **TodoWrite path** (solo): Agent writes entire todo list to `AppState.todos[agentId]`; purely in-memory
3. **Tasks path** (team): Agent uses CRUD tools that read/write individual JSON files under `~/.claude/tasks/{team-name}/`, protected by `proper-lockfile` locks
4. **UI Update**: The terminal status line reflects task state (spinner text from `activeForm`, progress counts)
5. **Multi-Agent Coordination**: In swarm mode, multiple agents share the same filesystem-backed task list with atomic claiming, busy checks, and auto-unassign on shutdown

---

## Two Mutually Exclusive Systems: TodoWrite vs Tasks

A critical architectural detail: TodoWrite and the Task tools are **mutually exclusive**. Their `isEnabled()` functions are exact inverses of each other, controlled by the same feature gate function `sq()`.

```javascript
// TodoWrite: enabled when Tasks is DISABLED
isEnabled() { return !sq() }

// TaskCreate/Get/Update/List: enabled when Tasks is ENABLED
isEnabled() { return sq() }
```

This means:
- When `sq()` returns `true` (tasks enabled), only `TaskCreate`, `TaskGet`, `TaskUpdate`, and `TaskList` are available. `TodoWrite` is hidden.
- When `sq()` returns `false` (tasks disabled), only `TodoWrite` is available. The Task CRUD tools are hidden.
- There is **no scenario** where both TodoWrite and the Task tools are simultaneously available to the model.

---

## Feature Gate: The sq() Function

The primary feature gate is the `sq()` function (obfuscated name from the binary):

```javascript
function sq() {
    if (_H(process.env.CLAUDE_CODE_ENABLE_TASKS)) return false;  // explicitly disabled
    if (AR(process.env.CLAUDE_CODE_ENABLE_TASKS)) return true;   // explicitly enabled
    if (e_()) return false;  // non-interactive sessions: disabled
    return true;             // default: enabled
}
```

Where:
- `_H()` checks for falsy string values (`"false"`, `"0"`, `"no"`, etc.)
- `AR()` checks for truthy string values (`"true"`, `"1"`, `"yes"`, etc.)
- `e_()` detects non-interactive mode (piped input, CI environments, etc.)

**Default behavior**: In interactive terminal sessions, Tasks is enabled by default (and TodoWrite is disabled). In non-interactive sessions, TodoWrite is the active system.

The environment variable `CLAUDE_CODE_ENABLE_TASKS` provides explicit control to override these defaults.

---

## TodoWrite System (Solo Mode)

TodoWrite is the simpler task-tracking system, active when the Tasks feature is disabled. It operates by **atomic list replacement** -- every call overwrites the entire todo list.

### Data Model (from source)

```javascript
// Obfuscated names: fyD = status enum, VyD = single todo item, OOT = todo array
var fyD = z.enum(["pending", "in_progress", "completed"]);

var VyD = z.object({
    content: z.string().min(1, "Content cannot be empty"),
    status: fyD,
    activeForm: z.string().min(1, "Active form cannot be empty")
});

var OOT = z.array(VyD);  // The full todo list type
```

Each todo item has exactly three fields:
- **content**: Imperative description (e.g., "Fix the login bug")
- **status**: One of `"pending"`, `"in_progress"`, `"completed"`
- **activeForm**: Present continuous form shown in spinner (e.g., "Fixing the login bug")

Note: There is no `id` field, no `priority` field, and no dependency support. The model is intentionally minimal.

### Tool Implementation (from source)

```javascript
var ZO = {
    name: "TodoWrite",
    maxResultSizeChars: 1e5,            // 100KB max result
    strict: true,

    inputSchema: z.strictObject({
        todos: OOT.describe("The updated todo list")
    }),

    outputSchema: z.object({
        oldTodos: OOT.describe("The todo list before the update"),
        newTodos: OOT.describe("The todo list after the update")
    }),

    userFacingName() { return "" },     // Empty -- not shown to user as a tool name
    isEnabled() { return !sq() },       // Only when Tasks is DISABLED
    isConcurrencySafe() { return false },
    isReadOnly() { return false },

    async checkPermissions(T) {
        return { behavior: "allow", updatedInput: T };  // Always allowed
    },

    // All render functions return null (invisible to user as a tool invocation)
    renderToolUseMessage: () => null,
    renderToolUseProgressMessage: () => null,
    renderToolUseRejectedMessage: () => null,
    renderToolUseErrorMessage: () => null,
    renderToolResultMessage: () => null,

    async call({ todos: T }, R) {
        let A = await R.getAppState();
        let _ = R.agentId ?? bR();      // Get agent ID or session ID
        let B = A.todos[_] ?? [];       // Old todos for this agent
        // If all completed, clear the list; otherwise store as-is
        let D = T.every(($) => $.status === "completed") ? [] : T;
        R.setAppState(($) => ({
            ...$,
            todos: { ...$.todos, [_]: D }
        }));
        return { data: { oldTodos: B, newTodos: T } };
    },

    mapToolResultToToolResultBlockParam(T, R) {
        return {
            tool_use_id: R,
            type: "tool_result",
            content: "Todos have been modified successfully. Ensure that you continue " +
                     "to use the todo list to track your progress. Please proceed with " +
                     "the current tasks if applicable"
        };
    }
};
```

### Key Behaviors

1. **Atomic Replacement**: Every call replaces the entire todo list. There is no "add" or "remove" -- the model always sends the full list.
2. **Auto-Clear on Completion**: When every item has `status: "completed"`, the stored list becomes empty (`[]`). This prevents stale completed items from cluttering state.
3. **Per-Agent Storage**: Todos are keyed by agent ID in `AppState.todos[agentId]`.
4. **In-Memory Only**: TodoWrite stores data in React app state (`AppState.todos`). There is **no file persistence** -- data is lost when the session ends or the process terminates. (Note: The `~/.claude/todos/` files visible in some contexts are a separate display/persistence mechanism, not part of TodoWrite's core storage.)
5. **Always Permitted**: `checkPermissions` always returns `allow` -- no user confirmation needed.
6. **Invisible UI**: All five render functions return `null`. TodoWrite calls are not shown as tool invocations in the chat. The todo list is rendered elsewhere via the expanded status view.

### TodoWrite Prompt (from source)

The model receives this guidance (stored in `ho9`):

> Use this tool to create and manage a structured task list for your current coding session. This helps you track progress, organize complex tasks, and demonstrate thoroughness to the user.

The full prompt includes extensive examples:
- **Use for**: Multi-step implementations, systematic search-and-replace across files, feature breakdowns, performance optimization tracking
- **Do not use for**: Single trivial tasks, informational requests, single-location changes

---

## Task System (Team/Swarm Mode)

The Task system is the richer, multi-agent task management system. Unlike TodoWrite, tasks are **persisted to disk as individual JSON files** and can be shared across multiple agents in a team.

### Data Model (from source)

```javascript
// Obfuscated: R9T = status enum, yyD = task schema
var R9T = z.enum(["pending", "in_progress", "completed"]);

var yyD = z.object({
    id: z.string(),
    subject: z.string(),
    description: z.string(),
    activeForm: z.string().optional(),
    owner: z.string().optional(),
    status: R9T,
    blocks: z.array(z.string()),         // Task IDs that THIS task blocks
    blockedBy: z.array(z.string()),      // Task IDs that block THIS task
    metadata: z.record(z.string(), z.unknown()).optional()
});
```

A task has:
- **id**: Auto-incrementing string integer (e.g., `"1"`, `"2"`, `"3"`)
- **subject**: Brief title
- **description**: Detailed description of what needs to be done
- **activeForm**: Present continuous form for spinner display (optional)
- **owner**: Agent name that claimed the task (optional)
- **status**: `"pending"` | `"in_progress"` | `"completed"` (note: there is NO `"deleted"` status -- deletion removes the file entirely)
- **blocks**: Array of task IDs that this task blocks (forward dependency)
- **blockedBy**: Array of task IDs that block this task (reverse dependency)
- **metadata**: Arbitrary key-value pairs; the `_internal` key marks hidden/internal tasks

### Task List Identification

Each task list is identified by a "list ID" which determines the filesystem directory. The list ID is resolved via priority chain:

```javascript
function nW() {
    // Priority 1: Environment variable override
    if (process.env.CLAUDE_CODE_TASK_LIST_ID)
        return process.env.CLAUDE_CODE_TASK_LIST_ID;
    // Priority 2: Teammate context (team name from AsyncLocalStorage)
    let T = FF();
    if (T) return T.teamName;
    // Priority 3: Dynamic team context or explicit team name
    return v7() || nBA || bR();  // team name || manually set name || session ID
}
```

All agents in a team share the same task list because they resolve to the same team name.

---

## Task Persistence Layer

### Directory Structure

Tasks are persisted as individual JSON files on disk:

```
~/.claude/
  tasks/
    {sanitized-team-name}/
      .lock                  # File lock for concurrent access (proper-lockfile)
      .highwatermark         # Highest task ID ever used (survives deletions)
      1.json                 # Task #1
      2.json                 # Task #2
      3.json                 # Task #3
      ...
```

### Path Construction (from source)

```javascript
// Sanitize names: replace non-alphanumeric chars with dashes
function JOT(T) {
    return T.replace(/[^a-zA-Z0-9_-]/g, "-");
}

// Base directory for a task list
function zF(T) {
    return path.join(q9(), "tasks", JOT(T));
    // q9() returns the .claude config directory (e.g., ~/.claude)
}

// Path to a specific task file
function LfT(T, R) {
    return path.join(zF(T), `${JOT(R)}.json`);
}
```

### File Locking

The system uses the `proper-lockfile` npm package (obfuscated as `EfT`) for file-level locking to handle concurrent access from multiple agents:

```javascript
// Lock path
function MyD(T) {
    return path.join(zF(T), ".lock");
}

// Ensure lock file exists and return path
function yo9(T) {
    KfT(T);  // Ensure directory exists
    let R = MyD(T);
    if (!fs.existsSync(R)) writeFileSync(R, "");
    return R;
}
```

Operations that modify tasks acquire locks:
- **`WOT`** (create task): Locks the `.lock` file (global lock)
- **`oBA`** (claim task): Locks the individual task file (per-task lock)
- **`SyD`** (claim with busy check): Locks the `.lock` file (global lock, reads all tasks atomically)

### CRUD Operations (from source)

**Create** (`WOT`):
```javascript
function WOT(T, R) {
    let A = yo9(T), _;  // Get lock path
    try {
        _ = lockfileSync(A);  // Acquire global lock
        let B = kyD(T);       // Get highest ID (max of files and highwatermark)
        let D = String(B + 1);
        let $ = { id: D, ...R };
        let H = LfT(T, D);   // e.g., ~/.claude/tasks/my-team/4.json
        writeFileSync(H, JSON.stringify($, null, 2));
        COT();                // Notify change listeners
        return D;             // Return new task ID
    } finally {
        if (_) _();           // Release lock
    }
}
```

**Read** (`bu`):
```javascript
function bu(T, R) {
    let A = LfT(T, R);  // Path to task file
    try {
        let _ = fs.readFileSync(A, "utf-8");
        let B = JSON.parse(_);
        let D = yyD.safeParse(B);  // Validate against Zod schema
        if (!D.success) {
            log(`[Tasks] Task ${R} failed schema validation: ${D.error.message}`);
            return null;
        }
        return D.data;
    } catch (_) {
        if (_.code === "ENOENT") return null;
        log(`[Tasks] Failed to read task ${R}: ${_ instanceof Error ? _.message : String(_)}`);
        return null;
    }
}
```

**Update** (`Qw`):
```javascript
function Qw(T, R, A) {
    let _ = bu(T, R);         // Read current task
    if (!_) return null;
    let B = { ..._, ...A, id: R };  // Merge updates, preserve ID
    let D = LfT(T, R);
    writeFileSync(D, JSON.stringify(B, null, 2));
    COT();                    // Notify listeners
    return B;
}
```

**Delete** (`f7R`):
```javascript
function f7R(T, R) {
    let A = LfT(T, R);
    if (!fs.existsSync(A)) return false;
    try {
        // Update highwatermark if this ID is higher
        let _ = parseInt(R, 10);
        if (!isNaN(_)) {
            let D = aBA(T);  // Read current highwatermark
            if (_ > D) wo9(T, _);  // Write new highwatermark
        }
        fs.unlinkSync(A);   // Delete the JSON file

        // Clean up dependency references in all remaining tasks
        let B = D5(T);  // List all tasks
        for (let D of B) {
            let $ = D.blocks.filter((q) => q !== R);
            let H = D.blockedBy.filter((q) => q !== R);
            if ($.length !== D.blocks.length || H.length !== D.blockedBy.length)
                Qw(T, D.id, { blocks: $, blockedBy: H });
        }
        COT();
        return true;
    } catch { return false; }
}
```

**List All** (`D5`):
```javascript
function D5(T) {
    let R = zF(T);
    if (!fs.existsSync(R)) return [];
    let A = fs.readdirSync(R), _ = [];
    for (let B of A) {
        if (!B.endsWith(".json")) continue;
        let D = B.replace(".json", "");
        let $ = bu(T, D);
        if ($) _.push($);
    }
    return _;
}
```

**Clear All** (`z7R`):
```javascript
function z7R(T) {
    let R = zF(T);
    KfT(T);  // Ensure dir exists
    let A = path.join(R, ".lock");
    if (!fs.existsSync(A)) writeFileSync(A, "");
    let _;
    try {
        _ = lockfileSync(A);
        // Update highwatermark to preserve ID sequence
        let B = Po9(T);       // Highest ID from files
        if (B > 0) {
            let D = aBA(T);   // Current highwatermark
            if (B > D) wo9(T, B);
        }
        // Delete all .json files (but NOT .highwatermark or .lock)
        if (fs.existsSync(R)) {
            let D = fs.readdirSync(R);
            for (let $ of D) {
                if ($.endsWith(".json") && !$.startsWith(".")) {
                    let H = path.join(R, $);
                    try { fs.unlinkSync(H); } catch {}
                }
            }
        }
        COT();
    } finally {
        if (_) _();
    }
}
```

---

## Task CRUD Tools

### TaskCreate

Creates a new task with `pending` status. Sets the UI to the tasks expanded view.

```javascript
var uIB = {
    name: "TaskCreate",
    maxResultSizeChars: 1e5,

    inputSchema: z.strictObject({
        subject: z.string().describe("A brief title for the task"),
        description: z.string().describe("A detailed description of what needs to be done"),
        activeForm: z.string().optional().describe(
            'Present continuous form shown in spinner when in_progress (e.g., "Running tests")'
        ),
        metadata: z.record(z.string(), z.unknown()).optional().describe(
            "Arbitrary metadata to attach to the task"
        )
    }),

    outputSchema: z.object({
        task: z.object({ id: z.string(), subject: z.string() })
    }),

    description() { return "Create a new task in the task list" },
    userFacingName() { return "TaskCreate" },
    isEnabled() { return sq() },     // Only in team/task mode
    isConcurrencySafe() { return true },
    isReadOnly() { return false },

    async call({ subject, description, activeForm, metadata }, context) {
        let id = WOT(nW(), {
            subject, description, activeForm,
            status: "pending",
            owner: undefined,
            blocks: [],
            blockedBy: [],
            metadata
        });
        context.setAppState(($) => {
            if ($.expandedView === "tasks") return $;
            return { ...$, expandedView: "tasks" };
        });
        return { data: { task: { id, subject } } };
    },

    mapToolResultToToolResultBlockParam(T, R) {
        let { task } = T;
        return {
            tool_use_id: R, type: "tool_result",
            content: `Task #${task.id} created successfully: ${task.subject}`
        };
    }
};
```

New tasks are always created with `status: "pending"`, `owner: undefined`, and empty `blocks`/`blockedBy` arrays.

### TaskGet

Retrieves full details of a single task by ID.

```javascript
var aIB = {
    name: "TaskGet",

    inputSchema: z.strictObject({
        taskId: z.string().describe("The ID of the task to retrieve")
    }),

    outputSchema: z.object({
        task: z.object({
            id: z.string(),
            subject: z.string(),
            description: z.string(),
            status: R9T,
            blocks: z.array(z.string()),
            blockedBy: z.array(z.string())
        }).nullable()
    }),

    description() { return "Get a task by ID from the task list" },
    isEnabled() { return sq() },
    isConcurrencySafe() { return true },
    isReadOnly() { return true },

    async call({ taskId }) {
        let task = bu(nW(), taskId);
        if (!task) return { data: { task: null } };
        return {
            data: {
                task: {
                    id: task.id, subject: task.subject,
                    description: task.description, status: task.status,
                    blocks: task.blocks, blockedBy: task.blockedBy
                }
            }
        };
    }
};
```

Returns `null` if the task does not exist (file not found or schema validation failure).

### TaskUpdate

Updates an existing task. This is the most complex tool, handling status transitions, dependency additions, metadata merging, deletion, auto-owner assignment, and completion verification hooks.

```javascript
var _ZB = {
    name: "TaskUpdate",

    inputSchema: z.strictObject({
        taskId: z.string().describe("The ID of the task to update"),
        subject: z.string().optional(),
        description: z.string().optional(),
        activeForm: z.string().optional(),
        status: R9T.or(z.literal("deleted")).optional(),
        addBlocks: z.array(z.string()).optional(),
        addBlockedBy: z.array(z.string()).optional(),
        owner: z.string().optional(),
        metadata: z.record(z.string(), z.unknown()).optional()
    }),

    outputSchema: z.object({
        success: z.boolean(),
        taskId: z.string(),
        updatedFields: z.array(z.string()),
        error: z.string().optional(),
        statusChange: z.object({ from: z.string(), to: z.string() }).optional()
    }),

    isEnabled() { return sq() },
    isConcurrencySafe() { return true },
    isReadOnly() { return false },

    // ... (see full implementation in Task CRUD Tools section of source analysis)
};
```

**Key behaviors in the `call()` implementation:**

1. **Deletion via status**: Setting `status: "deleted"` calls the `f7R()` delete function, which removes the JSON file and cleans up dependency references in other tasks. There is no actual `"deleted"` status stored -- the file is simply removed.

2. **Auto-assign owner in swarm mode**: When setting a task to `in_progress` and no owner is specified, the system automatically assigns the current agent as owner if running in swarm mode:

```javascript
if (d9() && status === "in_progress" && owner === undefined && !currentTask.owner) {
    let agentName = v8();  // getAgentName()
    if (agentName) { updates.owner = agentName; updatedFields.push("owner"); }
}
```

3. **Completion hooks**: Before marking a task `"completed"`, the system runs verification hooks (`evT()`) that can block completion if validation fails:

```javascript
if (status === "completed") {
    let errors = [];
    let hookIterator = evT(taskId, currentTask.subject, currentTask.description,
                           v8(), v7(), undefined,
                           context?.abortController?.signal, undefined, context);
    for await (let event of hookIterator) {
        if (event.blockingError) errors.push(tvT(event.blockingError));
    }
    if (errors.length > 0) {
        return { data: { success: false, taskId, updatedFields: [],
                 error: errors.join("\n") } };
    }
}
```

4. **Metadata merging**: Metadata is merged with null-deletion support -- setting a key to `null` removes it:

```javascript
if (metadata !== undefined) {
    let merged = { ...currentTask.metadata ?? {} };
    for (let [key, value] of Object.entries(metadata)) {
        if (value === null) delete merged[key];
        else merged[key] = value;
    }
    updates.metadata = merged;
}
```

### TaskList

Lists all tasks in summary form, with intelligent filtering.

```javascript
var WZB = {
    name: "TaskList",

    inputSchema: z.strictObject({}),  // No inputs

    outputSchema: z.object({
        tasks: z.array(z.object({
            id: z.string(),
            subject: z.string(),
            status: R9T,
            owner: z.string().optional(),
            blockedBy: z.array(z.string())
        }))
    }),

    isEnabled() { return sq() },
    isConcurrencySafe() { return true },
    isReadOnly() { return true },

    async call() {
        let listId = nW();
        // Filter out internal tasks (metadata._internal)
        let allTasks = D5(listId).filter((t) => !t.metadata?._internal);
        // Track completed task IDs to filter blockedBy
        let completedIds = new Set(
            allTasks.filter((t) => t.status === "completed").map((t) => t.id)
        );
        return {
            data: {
                tasks: allTasks.map((t) => ({
                    id: t.id,
                    subject: t.subject,
                    status: t.status,
                    owner: t.owner,
                    // Remove completed tasks from blockedBy (they no longer block)
                    blockedBy: t.blockedBy.filter((dep) => !completedIds.has(dep))
                }))
            }
        };
    }
};
```

**Two critical filtering behaviors:**

1. **Internal task hiding**: Tasks with `metadata._internal` set are filtered out of the listing. This allows the system to create bookkeeping tasks that are invisible to agents.
2. **Completed dependency pruning**: The `blockedBy` arrays in the output have completed task IDs removed. This means agents see only *active* blockers, not historical ones. A task whose blockers have all completed will show an empty `blockedBy` array even if it originally had dependencies.

**Output format** (from `mapToolResultToToolResultBlockParam`):
```
#1 [pending] Fix authentication bug
#2 [in_progress] Write unit tests (agent-1)
#3 [pending] Deploy to staging [blocked by #2]
```

---

## Task Dependency Resolution

### Adding Dependencies

Dependencies are bidirectional. When task A blocks task B, both sides are updated:

```javascript
function rBA(T, R, A) {
    // T = listId, R = blocker task ID, A = blocked task ID
    let _ = bu(T, R);  // Read blocker task
    let B = bu(T, A);  // Read blocked task
    if (!_ || !B) return false;

    // Add A to R's "blocks" list (R blocks A)
    if (!_.blocks.includes(A))
        Qw(T, R, { blocks: [..._.blocks, A] });

    // Add R to A's "blockedBy" list (A is blocked by R)
    if (!B.blockedBy.includes(R))
        Qw(T, A, { blockedBy: [...B.blockedBy, R] });

    return true;
}
```

### Dependency Checking During Claims

When an agent tries to claim a task, active blockers are checked:

```javascript
let allTasks = D5(listId);
let activeIds = new Set(
    allTasks.filter((t) => t.status !== "completed").map((t) => t.id)
);
let activeBlockers = task.blockedBy.filter((id) => activeIds.has(id));
if (activeBlockers.length > 0)
    return { success: false, reason: "blocked", task, blockedByTasks: activeBlockers };
```

Only non-completed blockers prevent claiming. Once a blocking task reaches `"completed"` status, it no longer prevents the dependent task from being claimed.

### Cleanup on Deletion

When a task is deleted (file removed), all references to it are cleaned up in remaining tasks:

```javascript
let allTasks = D5(listId);
for (let task of allTasks) {
    let newBlocks = task.blocks.filter((q) => q !== deletedId);
    let newBlockedBy = task.blockedBy.filter((q) => q !== deletedId);
    if (newBlocks.length !== task.blocks.length || newBlockedBy.length !== task.blockedBy.length)
        Qw(listId, task.id, { blocks: newBlocks, blockedBy: newBlockedBy });
}
```

### Smart Display in TaskList

As described above, TaskList filters completed tasks from `blockedBy` arrays in its output:

```javascript
blockedBy: t.blockedBy.filter((dep) => !completedIds.has(dep))
```

This ensures agents see a clear picture of what is actually blocking work, without noise from already-resolved dependencies.

### Example Dependency Graph

```
Task #1 (no dependencies)             -> can start immediately
Task #2 (blockedBy: ["1"])             -> waits for #1
Task #3 (blockedBy: ["1"])             -> waits for #1
Task #4 (blockedBy: ["2", "3"])        -> waits for both #2 and #3
Task #5 (no dependencies)             -> can start immediately, parallel to #1
```

Tasks #1 and #5 can run concurrently. Once #1 completes, #2 and #3 become eligible. #4 can only start when both #2 and #3 are complete.

---

## Task Claiming and Ownership

The task system has sophisticated claiming mechanics to prevent race conditions in multi-agent environments.

### Basic Claim (oBA)

The basic claim function acquires a per-task file lock and validates several conditions:

```javascript
function oBA(T, R, A, _ = {}) {
    // T = listId, R = taskId, A = agentName, _ = options
    let taskPath = LfT(T, R);
    if (!fs.existsSync(taskPath))
        return { success: false, reason: "task_not_found" };

    if (_.checkAgentBusy) return SyD(T, R, A);  // Delegate to stricter check

    let lock;
    try {
        lock = lockfileSync(taskPath);  // Lock the task file itself
        let task = bu(T, R);
        if (!task) return { success: false, reason: "task_not_found" };

        if (task.owner && task.owner !== A)
            return { success: false, reason: "already_claimed", task };

        if (task.status === "completed")
            return { success: false, reason: "already_resolved", task };

        // Check dependency blockers
        let allTasks = D5(T);
        let activeIds = new Set(allTasks.filter(t => t.status !== "completed").map(t => t.id));
        let blockers = task.blockedBy.filter(id => activeIds.has(id));
        if (blockers.length > 0)
            return { success: false, reason: "blocked", task, blockedByTasks: blockers };

        // Claim it
        return { success: true, task: Qw(T, R, { owner: A }) };
    } finally {
        if (lock) lock();  // Release lock
    }
}
```

### Claim with Busy Check (SyD)

A stricter variant that prevents an agent from claiming multiple tasks simultaneously. Uses a global lock (not per-task) to atomically read all tasks:

```javascript
function SyD(T, R, A) {
    let lockPath = yo9(T), lock;
    try {
        lock = lockfileSync(lockPath);  // Global lock
        let allTasks = D5(T);
        let task = allTasks.find(t => t.id === R);

        // ... same validation as oBA ...

        // ADDITIONAL: Check if agent already has an active task
        let busyTasks = allTasks.filter(t =>
            t.status !== "completed" && t.owner === A && t.id !== R
        );
        if (busyTasks.length > 0)
            return {
                success: false, reason: "agent_busy",
                task, busyWithTasks: busyTasks.map(t => t.id)
            };

        return { success: true, task: Qw(T, R, { owner: A }) };
    } finally {
        if (lock) lock();
    }
}
```

### Claim Failure Reasons

| Reason | Meaning |
|--------|---------|
| `task_not_found` | Task ID does not exist (file missing) |
| `already_claimed` | Another agent owns this task |
| `already_resolved` | Task is already completed |
| `blocked` | Task has unresolved dependency blockers |
| `agent_busy` | Agent already owns another active task (strict mode only) |

### Auto-Unassign on Agent Shutdown (qn)

When an agent shuts down or is terminated, its claimed tasks are automatically released back to `"pending"` with no owner:

```javascript
function qn(T, R, A, _) {
    // T = listId, R = agentId, A = agentName, _ = shutdown reason
    let tasks = D5(T).filter(t =>
        t.status !== "completed" && (t.owner === R || t.owner === A)
    );
    for (let task of tasks)
        Qw(T, task.id, { owner: undefined, status: "pending" });

    if (tasks.length > 0)
        log(`[Tasks] Unassigned ${tasks.length} task(s) from ${A}`);

    let message = `${A} ${_ === "terminated" ? "was terminated" : "has shut down"}.`;
    if (tasks.length > 0) {
        let names = tasks.map(t => `#${t.id} "${t.subject}"`).join(", ");
        message += ` ${tasks.length} task(s) were unassigned: ${names}. ` +
                   `Use TaskList to check availability and TaskUpdate with owner ` +
                   `to reassign them to idle teammates.`;
    }
    return {
        unassignedTasks: tasks.map(t => ({ id: t.id, subject: t.subject })),
        notificationMessage: message
    };
}
```

This ensures no tasks are permanently "stuck" when an agent crashes or is killed. The notification message is sent to remaining agents so they can re-claim the orphaned work.

---

## Highwatermark: Monotonic ID Generation

### Purpose

The `.highwatermark` file at `~/.claude/tasks/{team-name}/.highwatermark` preserves the highest task ID ever assigned to a task list, even after tasks are deleted or the list is cleared. This ensures that new task IDs are always **unique and monotonically increasing**.

Without the highwatermark, deleting task #5 and then creating a new task could produce another #5, causing confusion in conversation history that references the original #5.

### Implementation (from source)

```javascript
var byD = ".highwatermark";  // Filename constant

// Path to highwatermark file
function Vo9(T) {
    return path.join(zF(T), byD);
}

// Read current highwatermark value
function aBA(T) {
    let R = Vo9(T);
    try {
        let A = fs.readFileSync(R, "utf-8").trim();
        let _ = parseInt(A, 10);
        return isNaN(_) ? 0 : _;
    } catch {
        return 0;  // Default to 0 if file doesn't exist
    }
}

// Write new highwatermark value
function wo9(T, R) {
    let A = Vo9(T);
    writeFileSync(A, String(R));
}
```

### ID Generation Logic

The next task ID is computed as `max(highest_existing_file_id, highwatermark) + 1`:

```javascript
function kyD(T) {
    let R = Po9(T);   // Highest ID from .json files in directory
    let A = aBA(T);   // Highwatermark value from .highwatermark file
    return Math.max(R, A);
}
```

Where `Po9` scans `.json` filenames to find the highest numeric ID:

```javascript
function Po9(T) {
    let R = zF(T);
    if (!fs.existsSync(R)) return 0;
    let A = fs.readdirSync(R), _ = 0;
    for (let B of A) {
        if (!B.endsWith(".json")) continue;
        let D = parseInt(B.replace(".json", ""), 10);
        if (!isNaN(D) && D > _) _ = D;
    }
    return _;
}
```

### When Highwatermark is Updated

1. **Deleting a task** (`f7R`): If the deleted task's ID exceeds the current highwatermark, the highwatermark is raised
2. **Clearing all tasks** (`z7R`): Before deleting files, the highest file-based ID is written to the highwatermark

---

## Task Lifecycle and State Machine

### Valid States

| State | Description | Stored As |
|-------|-------------|-----------|
| `pending` | Created but not yet started | `"pending"` in JSON |
| `in_progress` | Actively being worked on by an agent | `"in_progress"` in JSON |
| `completed` | Successfully finished | `"completed"` in JSON |
| *(deleted)* | Task removed -- **file is deleted from disk** | No file exists |

There is no `"deleted"` status value. The TaskUpdate tool accepts `status: "deleted"` as input, but this triggers file deletion via `f7R()` rather than storing a deleted status.

### State Machine

```
                    +-----------+
                    |           |
          +-------->  pending   +--------+
          |         |           |        |
          |         +-----+-----+        |
          |               |              |
          |  (claim/start)|              | (delete -> file removed)
          |               v              |
          |         +-----------+        |
          |         |           |        |
 (release +--------+ in_progress+---+    |
  on       |        |           |   |    |
  shutdown)|        +-----+-----+   |    |
                          |         |    |
             (complete,   |         |    |
              hooks pass) |         |    |
                          v         |    v
                    +-----------+   |  (file deleted,
                    |           |   |   deps cleaned,
                    | completed |   |   hwm updated)
                    |           |   |
                    +-----------+   |
                                    |
                                    +--> (delete -> file removed)
```

### Transition Rules

| Transition | Trigger | Validation |
|-----------|---------|------------|
| pending -> in_progress | Agent claims/starts task | Blockers must be resolved; owner set (auto in swarm mode) |
| in_progress -> completed | Agent finishes task | Completion hooks/verification must pass |
| in_progress -> pending | Agent shutdown (`qn`) | Owner cleared, status reset |
| pending -> *(deleted)* | Task no longer needed | File removed, deps cleaned, highwatermark updated |
| in_progress -> *(deleted)* | Task no longer needed | File removed, deps cleaned, highwatermark updated |

---

## Background Task System

Separate from the shared task list, Claude Code has a background task system for running processes asynchronously. This system lives in `AppState.tasks` and uses a different ID scheme.

### Task Types

```javascript
var ML7 = {
    local_bash: "b",           // Background bash commands
    local_agent: "a",          // Background agent sessions
    remote_agent: "r",         // Remote agent sessions
    in_process_teammate: "t"   // In-process teammate agents
};
```

### Background Task ID Generation

Background task IDs use a type prefix + random UUID segment, completely separate from the sequential integer IDs of the shared task list:

```javascript
function Kc(T) {  // T = task type string
    let prefix = ML7[T] ?? "x";
    let random = crypto.randomUUID().replace(/-/g, "").substring(0, 6);
    return `${prefix}${random}`;  // e.g., "b3f7a2c", "t9e1d4b", "a2c8f1e"
}
```

### Task State Object

```javascript
function EL(T, R, A) {
    return {
        id: T,                  // Task ID (e.g., "b3f7a2c")
        type: R,                // "local_bash", "local_agent", etc.
        status: "pending",      // -> "running" -> "completed"/"failed"/"killed"
        description: A,
        startTime: Date.now(),
        outputFile: b4(T),      // Path to .output file
        outputOffset: 0,
        notified: false
    };
}
```

### Task Output Files

Background tasks write output to files that can be read incrementally:

```javascript
function b4(T) {
    return path.join(rkT(), `${T}.output`);
}

// Append output
function $0T(T, R) {
    await fs.appendFile(outputPath, R, "utf8");
}

// Read output with offset (for incremental reads)
function hQA(T, R) {
    // Returns { content: string, newOffset: number }
}
```

### TaskOutput Tool

The `TaskOutput` tool retrieves output from background tasks:

```javascript
var NIR = {
    name: "TaskOutput",
    maxResultSizeChars: 1e5,
    aliases: ["AgentOutputTool", "BashOutputTool"],

    inputSchema: z.strictObject({
        task_id: z.string().describe("The task ID to get output from"),
        block: z.boolean().default(true).describe("Whether to wait for completion"),
        timeout: z.number().min(0).max(600000).default(30000).describe("Max wait time in ms")
    }),

    description() { return "Retrieves output from a running or completed task" },
    isReadOnly() { return true },
    isEnabled() { return true },  // Always available (regardless of sq() state)
};
```

Note: `TaskOutput` is **always enabled** (`isEnabled() { return true }`), unlike the shared task list tools which are gated by `sq()`. This is because background tasks exist in both solo and team modes.

---

## Team System

The task system integrates with Claude Code's team/swarm capabilities through several mechanisms.

### Team Context

Agent identity and team membership are tracked via AsyncLocalStorage:

```javascript
var mBA;  // AsyncLocalStorage for teammate context

function FF() {
    // Returns teammate context: { teamName, agentName, agentId, ... }
    return mBA?.getStore();
}

// Helper functions
function H2() { /* get agent ID */ }
function v8() { /* get agent name */ }
function v7() { /* get team name */ }
function a1() { /* is this agent a teammate (not the lead) */ }
function iW() { /* is this agent the team lead */ }
function d9() { /* is swarm mode active */ }
```

### Team Management Tools

The source analysis reveals team management tools exist alongside the task tools:

- **TeamCreate**: Creates a new team of agents
- **TeamDelete**: Removes a team
- **SendMessage**: Sends messages between agents in a team

These tools work alongside the shared task list -- a team shares a single task list identified by the team name, and agents coordinate through claiming, ownership, and the notification system.

### Swarm Mode

When swarm mode is active (`d9()` returns true):
- Tasks auto-assign the current agent as owner when set to `in_progress`
- The busy check (`SyD`) can prevent an agent from claiming multiple tasks
- Agent shutdown triggers auto-unassign (`qn`) with notification to remaining agents

---

## Task Notifications and Queue System

### Change Listeners

Any part of the system can register for task change notifications:

```javascript
var iBA = new Set();  // Set of listener functions

function fo9(T) {     // Register a listener
    iBA.add(T);
    return () => iBA.delete(T);  // Returns unsubscribe function
}

function COT() {      // Notify all listeners
    for (let T of iBA) {
        try { T(); } catch {}
    }
}
```

`COT()` is called after every task mutation (create, update, delete, clear).

### Task Notification Tags

Task notifications use XML-like tags in the agent message stream:

```javascript
var $O = "task-notification";  // Tag name constant

// Notification format:
// <task-notification>
// Read the output file to retrieve the result: {outputFilePath}
// <task-id>{id}</task-id>
// <task-type>{type}</task-type>
// <status>{status}</status>
// <summary>{summary}</summary>
// </task-notification>
```

### Notification Queue

```javascript
function NG(T, R) {
    if (T.mode === "task-notification" && D5R.size > 0) {
        n2T.push(typeof T.value === "string" ? T.value : "");
        H5R();  // Increment counter and notify
    } else {
        R((A) => ({ ...A, queuedCommands: [...A.queuedCommands, T] }));
    }
}
```

Task notifications are prioritized over regular queued commands when there are active listeners.

---

## UI Rendering

### Expanded Tasks View

When a task tool is called, the UI switches to the tasks expanded view:

```javascript
context.setAppState(($) => {
    if ($.expandedView === "tasks") return $;
    return { ...$, expandedView: "tasks" };
});
```

### Teammate Status Display

In team mode, each teammate shows:
- Agent name with color
- Status (running/idle/stopping/awaiting approval)
- Progress: tool use count and token count
- Recent activity description
- Duration

```javascript
// Status colors:
// "running"   -> "warning" (yellow)
// "completed" -> "success" (green)
// "failed"    -> "error"   (red)
// other       -> "inactive" (dim)
```

### Task List Output Format

The `mapToolResultToToolResultBlockParam` in TaskList renders:

```
#1 [pending] Fix authentication bug
#2 [in_progress] Write unit tests (agent-1)
#3 [pending] Deploy to staging [blocked by #2]
```

### Status Line / Spinner

When a task is `in_progress`, the `activeForm` text is shown in the terminal spinner:
```
 Fixing authentication bug
```

When tasks complete, they show success indicators. TodoWrite's `activeForm` field and the Task system's `activeForm` field both feed into this same spinner infrastructure.

---

## Environment Variables and Configuration

| Variable | Purpose | Default |
|----------|---------|---------|
| `CLAUDE_CODE_ENABLE_TASKS` | Master toggle. `"true"`/`"1"` enables Tasks (disables TodoWrite). `"false"`/`"0"` enables TodoWrite (disables Tasks). | Enabled in interactive mode, disabled in non-interactive mode |
| `CLAUDE_CODE_TASK_LIST_ID` | Override the task list identifier (directory name under `~/.claude/tasks/`) | Team name or session ID |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | Force plan mode for teammate agents | Not set |

---

## Obfuscated Symbol Map

Key variable/function mappings discovered in the v2.1.34 binary. These are useful for anyone performing their own analysis of the binary.

### Persistence Layer

| Obfuscated | Purpose |
|-----------|---------|
| `WOT` | Create task (write JSON file, return new ID) |
| `bu` | Read task (parse JSON file, validate schema) |
| `Qw` | Update task (merge fields, write JSON file) |
| `f7R` | Delete task (remove file, clean deps, update highwatermark) |
| `D5` | List all tasks (scan directory for .json files) |
| `z7R` | Clear all tasks (delete all .json, preserve highwatermark) |
| `nW` | Get current task list ID |
| `zF` | Get task list directory path |
| `LfT` | Get task file path |
| `KfT` | Ensure task directory exists |
| `JOT` | Sanitize name for filesystem |
| `EfT` | proper-lockfile module reference |

### Claiming and Ownership

| Obfuscated | Purpose |
|-----------|---------|
| `oBA` | Claim task (basic, per-task lock) |
| `SyD` | Claim task with agent busy check (global lock) |
| `qn` | Unassign tasks on agent shutdown |
| `rBA` | Add bidirectional dependency between tasks |

### ID Management

| Obfuscated | Purpose |
|-----------|---------|
| `Po9` | Get highest task ID from .json filenames |
| `kyD` | Get highest task ID (files + highwatermark) |
| `aBA` | Read highwatermark file |
| `wo9` | Write highwatermark file |
| `Vo9` | Get highwatermark file path |
| `byD` | Highwatermark filename constant (`".highwatermark"`) |

### Schema and Types

| Obfuscated | Purpose |
|-----------|---------|
| `fyD` | TodoWrite status enum |
| `VyD` | TodoWrite item schema |
| `OOT` | TodoWrite array schema |
| `R9T` | Task status enum |
| `yyD` | Task schema (Zod) |

### Tool Definitions

| Obfuscated | Tool Name | System |
|-----------|-----------|--------|
| `ZO` | `TodoWrite` | Solo mode |
| `uIB` | `TaskCreate` | Team mode |
| `aIB` | `TaskGet` | Team mode |
| `_ZB` | `TaskUpdate` | Team mode |
| `WZB` | `TaskList` | Team mode |
| `NIR` | `TaskOutput` | Both (always enabled) |

### Agent and Team Context

| Obfuscated | Purpose |
|-----------|---------|
| `mBA` | AsyncLocalStorage for teammate context |
| `FF` | Get teammate context from AsyncLocalStorage |
| `H2` | Get agent ID |
| `v8` | Get agent name |
| `v7` | Get team name |
| `a1` | Is teammate (boolean) |
| `iW` | Is team lead |
| `d9` | Is swarm mode active |
| `sq` | Is Tasks feature enabled (feature gate) |

### Background Tasks

| Obfuscated | Purpose |
|-----------|---------|
| `ML7` | Task type prefix map |
| `Kc` | Generate background task ID (prefix + UUID) |
| `EL` | Create background task state object |
| `b4` | Get background task output file path |
| `$0T` | Append to background task output |
| `hQA` | Read background task output with offset |

### Notifications

| Obfuscated | Purpose |
|-----------|---------|
| `iBA` | Task change listener set |
| `fo9` | Register change listener |
| `COT` | Notify all change listeners |
| `$O` | Task notification tag name (`"task-notification"`) |
| `NG` | Enqueue command/notification |
| `evT` | Task completion verification hook iterator |
| `tvT` | Format blocking error message |

---

## Key Design Principles

### 1. Mutual Exclusivity, Not Generations

TodoWrite and the Task CRUD tools are **not** successive generations. They are parallel systems for different operational modes. TodoWrite serves solo agents in non-team contexts; Tasks serves multi-agent teams. The feature gate ensures exactly one system is active at any time.

### 2. Filesystem Persistence for Shared State

Tasks are persisted as individual JSON files on disk (`~/.claude/tasks/{team-name}/*.json`), not held in memory. This enables multiple agent processes to share state through the filesystem, with `proper-lockfile` providing concurrency control. TodoWrite, by contrast, is purely in-memory because it only needs per-session, per-agent scope.

### 3. Atomic Operations with File Locking

All task mutations that could race (create, claim, busy-check) acquire file locks before reading or writing. The system uses two locking granularities: per-task locks (for basic claims via `oBA`) and global directory locks (for operations that need to read all tasks atomically, like `SyD` and `WOT`).

### 4. Monotonic IDs Survive Deletions

The highwatermark file ensures task IDs never repeat, even after deletions or full clears. This prevents confusion in conversation history where agents might reference task #5 -- that always refers to the same task, never a recycled ID.

### 5. Graceful Agent Failure

When an agent shuts down or is terminated, its claimed tasks are automatically released back to `"pending"` status with no owner (`qn`). A notification message is generated for remaining agents to pick up the orphaned work. This prevents tasks from being permanently stuck.

### 6. Completion Verification

Before a task can be marked `"completed"`, the system runs verification hooks (`evT`) that can block the transition with errors. This ensures tasks are not prematurely closed.

### 7. Smart Display Filtering

TaskList actively cleans up its output: internal metadata tasks are hidden, and completed dependencies are pruned from `blockedBy` arrays. Agents see only actionable information.

### 8. Agent-Driven Orchestration

The AI agent decides when to create tasks, how to decompose work, and when to update status. The system provides tools and enforces constraints (locking, dependencies, ownership) but does not autonomously schedule or execute tasks.

---

*Based on source code analysis of Claude Code v2.1.34 binary (build 2026-02-06T06:37:05Z). All code snippets are deobfuscated reconstructions from `strings` extraction with variable names mapped to functional equivalents. See `task-system-source-analysis.md` for the complete analysis.*
