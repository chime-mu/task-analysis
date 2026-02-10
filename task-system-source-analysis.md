# Claude Code v2.1.34 Task System -- Source Analysis

**Binary**: `/Users/chime/.local/share/claude/versions/2.1.34` (Mach-O 64-bit arm64, Bun-compiled)
**Build**: 2026-02-06T06:37:05Z
**Method**: `strings` extraction + pattern-matched grep analysis of embedded JavaScript

> **Note**: This document analyzes Claude Code's internal implementation. These are undocumented internals -- file paths, data formats, and behaviors may change between versions without notice. See `orchestration-principle.md` for strategic implications.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Two Distinct Systems: TodoWrite vs Tasks](#2-two-distinct-systems-todowrite-vs-tasks)
3. [TodoWrite System (Solo Agent)](#3-todowrite-system-solo-agent)
4. [Task System (Multi-Agent / Team)](#4-task-system-multi-agent--team)
   - 4.3 [Task List Scoping & Session Isolation](#43-task-list-scoping--session-isolation)
5. [Task Persistence Layer](#5-task-persistence-layer)
6. [Task CRUD Tools](#6-task-crud-tools)
7. [Task Dependency Resolution](#7-task-dependency-resolution)
8. [Task Claiming & Ownership](#8-task-claiming--ownership)
9. [Highwatermark File](#9-highwatermark-file)
10. [Task ID Generation](#10-task-id-generation)
11. [Task Notifications & Queue System](#11-task-notifications--queue-system)
12. [Task UI Rendering](#12-task-ui-rendering)
13. [Background Task System (Local Agents)](#13-background-task-system-local-agents)
14. [TaskOutput Tool](#14-taskoutput-tool)
15. [Feature Flags & Enablement Logic](#15-feature-flags--enablement-logic)
16. [Obfuscated Symbol Map](#16-obfuscated-symbol-map)
17. [Complete Function Reference](#17-complete-function-reference)

---

## 1. Architecture Overview

Claude Code has **two entirely separate task-tracking systems** that serve different purposes:

| System | Tool Name | Storage | Scope | Enabled When |
|--------|-----------|---------|-------|-------------|
| **TodoWrite** | `TodoWrite` | In-memory app state (`AppState.todos`) | Per-agent, per-session | Tasks feature is **disabled** (solo mode) |
| **Tasks** | `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList` | Filesystem JSON files in `~/.claude/tasks/{team-name}/` | Shared across team agents | Tasks feature is **enabled** (team/swarm mode) |

These two systems are **mutually exclusive** -- the `isEnabled()` functions are inverses of each other. TodoWrite is enabled when `sq()` returns false (tasks disabled); the Task tools are enabled when `sq()` returns true (tasks enabled).

The broader "task" concept in the app state (`AppState.tasks`) also encompasses background processes: local bash tasks, local agent tasks, remote agents, and in-process teammates. These are separate from the "shared task list" system described above.

---

## 2. Two Distinct Systems: TodoWrite vs Tasks

### Mutual Exclusivity (Feature Gate: `sq()`)

```javascript
// The sq() function determines if the Tasks system is enabled
function sq() {
    if (_H(process.env.CLAUDE_CODE_ENABLE_TASKS)) return false;  // explicitly disabled
    if (AR(process.env.CLAUDE_CODE_ENABLE_TASKS)) return true;   // explicitly enabled
    if (e_()) return false;  // non-interactive sessions: disabled
    return true;             // default: enabled
}

// TodoWrite: enabled when Tasks is DISABLED
isEnabled() { return !sq() }

// TaskCreate/Get/Update/List: enabled when Tasks is ENABLED
isEnabled() { return sq() }
```

The environment variable `CLAUDE_CODE_ENABLE_TASKS` controls this gate. When unset, the default behavior depends on `e_()` (non-interactive mode detection). In interactive mode, Tasks is enabled by default, and TodoWrite is disabled.

---

## 3. TodoWrite System (Solo Agent)

### 3.1 Data Model (Zod Schema)

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

### 3.2 Tool Definition

```javascript
var yu = "TodoWrite";  // Tool name constant

var ZO = {
    name: yu,                           // "TodoWrite"
    maxResultSizeChars: 1e5,            // 100KB max result
    strict: true,

    // Input: full replacement of the todo list
    inputSchema: z.strictObject({
        todos: OOT.describe("The updated todo list")
    }),

    // Output: before and after snapshots
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

    // Render functions all return null (invisible to user)
    renderToolUseMessage: Ko9,          // () => null
    renderToolUseProgressMessage: No9,  // () => null
    renderToolUseRejectedMessage: Io9,  // () => null
    renderToolUseErrorMessage: Zo9,     // () => null
    renderToolResultMessage: Uo9,       // () => null

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

### 3.3 Key Behaviors

1. **Atomic Replacement**: Every call replaces the entire todo list. There is no "add" or "remove" -- the model always sends the full list.
2. **Auto-Clear on Completion**: When every item has `status: "completed"`, the stored list becomes empty (`[]`).
3. **Per-Agent Storage**: Todos are keyed by agent ID in `AppState.todos[agentId]`.
4. **In-Memory Only**: TodoWrite stores data in React app state. There is no file persistence.
5. **Always Permitted**: `checkPermissions` always returns `allow`.
6. **Invisible UI**: All five render functions return `null`. The todo list is rendered elsewhere (via the status line / expanded view).

### 3.4 TodoWrite Description & Prompt

**Description** (returned by `description()`):
```
Update the todo list for the current session. To be used proactively and often to
track progress and pending tasks. Make sure that at least one task is in_progress at
all times. Always provide both content (imperative) and activeForm (present continuous)
for each task.
```

**Prompt** (returned by `prompt()`, stored in `ho9`):
```
Use this tool to create and manage a structured task list for your current coding
session. This helps you track progress, organize complex tasks, and demonstrate
thoroughness to the user.
```

The full prompt includes extensive examples showing when to use and when not to use the todo list:
- **Use**: Multi-step implementations, systematic search-and-replace across files, feature breakdowns, performance optimization tracking
- **Don't use**: Single trivial tasks, informational requests, single-location changes

### 3.5 Input Examples

```javascript
input_examples: [
    {
        todos: [{
            content: "Fix the login bug",
            status: "pending",
            activeForm: "Fixing the login bug"
        }]
    },
    {
        todos: [
            { content: "Implement feature", status: "completed", activeForm: "Implementing feature" },
            { content: "Write unit tests", status: "in_progress", activeForm: "Writing unit tests" }
        ]
    }
]
```

---

## 4. Task System (Multi-Agent / Team)

### 4.1 Data Model (Zod Schema)

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
- **id**: Auto-incrementing string integer (e.g., "1", "2", "3")
- **subject**: Brief title
- **description**: Detailed description of what needs to be done
- **activeForm**: Present continuous form for spinner display (optional)
- **owner**: Agent name that claimed the task (optional)
- **status**: `"pending"` | `"in_progress"` | `"completed"`
- **blocks**: Array of task IDs that this task blocks (forward dependency)
- **blockedBy**: Array of task IDs that block this task (reverse dependency)
- **metadata**: Arbitrary key-value pairs; `_internal` key marks hidden tasks

### 4.2 Task List Identification

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

The task list is identified by a "list ID" resolved through a priority chain: environment variable override (`CLAUDE_CODE_TASK_LIST_ID`) -> teammate context (team name from AsyncLocalStorage) -> dynamic team name (`v7()`) -> manually set name (`nBA`) -> **random UUIDv7 via `bR()`**. In team/swarm mode, the team name typically provides the ID, so all agents share the same task list. In solo sessions, the chain falls through to `bR()`, which generates a random UUIDv7 per session -- meaning each solo session gets an isolated, undiscoverable task namespace. See Section 4.3 for the implications.

### 4.3 Task List Scoping & Session Isolation

In solo interactive sessions (the default mode), `nW()` falls through its entire priority chain to `bR()`, a UUIDv7 generator. This means:

1. **Each session gets a random namespace.** The task list ID is a random UUID like `019508a2-3f4e-7b8c-9d1a-...`. Tasks created during that session are stored at `~/.claude/tasks/{uuid}/`. There is no connection to the working directory, git repository, or project name.

2. **`nW()` never references `cwd` or `.git`.** The function's entire priority chain is: env var -> team context -> team name -> manual name -> random UUID. Project context is never consulted.

3. **Files persist but become orphaned.** After a session ends, the task directory remains on disk but there is no index, no discovery mechanism, and no way for a subsequent session to find it. The new session generates a fresh UUID and creates a fresh, empty task namespace.

4. **Empirical evidence.** Analysis of `~/.claude/tasks/` reveals 15 UUID-named directories. Cross-referencing against `~/.claude/projects/*/sessions-index.json` (which records `sessionId`, `projectPath`, `gitBranch`, `firstPrompt`, `summary`, and timestamps for 2,508 sessions across 12 projects), only 5 of the 15 directories are traceable to known sessions. The remaining 10 are orphans -- and all 10 are empty (contain only `.lock` and `.highwatermark` files, no task JSON).

5. **Crash behavior.** If a session crashes or is interrupted, tasks in `in_progress` status remain stuck indefinitely. There is no cleanup mechanism, no crash recovery, and no way for the user to discover which sessions had unfinished work. The `qn()` unassign-on-shutdown function (Section 8.4) only runs during graceful shutdowns.

6. **Recovery mechanisms.** Three partial mitigations exist:
   - **`--resume <sessionId>`**: CLI flag that loads a previous session's transcript. Whether it preserves the original session ID (and thus reconnects to the same task list) is unconfirmed from binary analysis alone.
   - **`--continue`**: Resumes the most recent session. Same uncertainty about task list reconnection.
   - **`CLAUDE_CODE_TASK_LIST_ID=<uuid>`**: Environment variable override. This is the guaranteed escape hatch -- setting it to the UUID of an existing task directory will reconnect to that task list regardless of session ID. Usage: `CLAUDE_CODE_TASK_LIST_ID=<uuid> claude --resume <sessionId>`.

7. **What does NOT exist.** There is no `/resume` slash command, no `TaskDiscover` tool, no CLI task browser, no way to list task directories with their contents, and no way to map a project or directory to its associated task lists. The `sessions-index.json` file records session metadata but has no link to task list IDs.

**The critical distinction is file persistence vs practical persistence.** Task files survive on disk indefinitely, but without the ability to discover and reconnect to them, they are functionally ephemeral for the user.

**Strategic implication**: The session-scoped nature of task lists, combined with the absence of any external API, reinforces that Claude Code's task system is an internal convenience layer, not an external interface to build on. See `orchestration-principle.md` for why orchestration should be independent of agent internals.

---

## 5. Task Persistence Layer

### 5.1 Directory Structure

```
~/.claude/
  tasks/
    {sanitized-team-name}/
      .lock                  # File lock for concurrent access
      .highwatermark         # Highest task ID ever used (survives deletion)
      1.json                 # Task #1
      2.json                 # Task #2
      3.json                 # Task #3
      ...
```

### 5.2 Path Construction

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

### 5.3 File Locking

The system uses the `proper-lockfile` npm package (obfuscated as `EfT`) for file-level locking:

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

Operations that modify tasks acquire a lock:
- `WOT` (create task) locks the `.lock` file
- `oBA` (claim task) locks the individual task file
- `SyD` (claim with busy check) locks the `.lock` file (global lock)

### 5.4 CRUD Operations

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
        let D = yyD.safeParse(B);  // Validate against schema
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
        fs.unlinkSync(A);

        // Clean up dependency references in other tasks
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

### 5.5 Clear All Tasks (`z7R`)

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

## 6. Task CRUD Tools

### 6.1 TaskCreate

```javascript
var uIB = {
    name: FP,  // "TaskCreate" constant
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
        // Switch UI to tasks expanded view
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

### 6.2 TaskGet

```javascript
var aIB = {
    name: O0T,  // "TaskGet" constant

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
    userFacingName() { return "TaskGet" },
    isEnabled() { return sq() },
    isConcurrencySafe() { return true },
    isReadOnly() { return true },

    async call({ taskId }) {
        let listId = nW();
        let task = bu(listId, taskId);  // Read from filesystem
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
    },

    mapToolResultToToolResultBlockParam(T, R) {
        let { task } = T;
        if (!task) return { tool_use_id: R, type: "tool_result", content: "Task not found" };
        let lines = [
            `Task #${task.id}: ${task.subject}`,
            `Status: ${task.status}`,
            `Description: ${task.description}`
        ];
        if (task.blockedBy.length > 0)
            lines.push(`Blocked by: ${task.blockedBy.map(b => `#${b}`).join(", ")}`);
        if (task.blocks.length > 0)
            lines.push(`Blocks: ${task.blocks.map(b => `#${b}`).join(", ")}`);
        return { tool_use_id: R, type: "tool_result", content: lines.join("\n") };
    }
};
```

### 6.3 TaskUpdate

```javascript
var _ZB = {
    name: Yz,  // "TaskUpdate" constant

    inputSchema: z.strictObject({
        taskId: z.string().describe("The ID of the task to update"),
        subject: z.string().optional().describe("New subject for the task"),
        description: z.string().optional().describe("New description for the task"),
        activeForm: z.string().optional().describe(
            'Present continuous form shown in spinner when in_progress (e.g., "Running tests")'
        ),
        status: R9T.or(z.literal("deleted")).optional().describe("New status for the task"),
        addBlocks: z.array(z.string()).optional().describe("Task IDs that this task blocks"),
        addBlockedBy: z.array(z.string()).optional().describe("Task IDs that block this task"),
        owner: z.string().optional().describe("New owner for the task"),
        metadata: z.record(z.string(), z.unknown()).optional().describe(
            "Metadata keys to merge into the task. Set a key to null to delete it."
        )
    }),

    outputSchema: z.object({
        success: z.boolean(),
        taskId: z.string(),
        updatedFields: z.array(z.string()),
        error: z.string().optional(),
        statusChange: z.object({ from: z.string(), to: z.string() }).optional()
    }),

    description() { return "Update a task in the task list" },
    userFacingName() { return "TaskUpdate" },
    isEnabled() { return sq() },
    isConcurrencySafe() { return true },
    isReadOnly() { return false },

    async call({ taskId, subject, description, activeForm, status, owner,
                 addBlocks, addBlockedBy, metadata }, context) {
        let listId = nW();

        // Switch to tasks view
        context.setAppState((Q) => {
            if (Q.expandedView === "tasks") return Q;
            return { ...Q, expandedView: "tasks" };
        });

        let currentTask = bu(listId, taskId);
        if (!currentTask)
            return { data: { success: false, taskId, updatedFields: [], error: "Task not found" } };

        let updatedFields = [];
        let updates = {};

        // Collect changed fields
        if (subject !== undefined && subject !== currentTask.subject)
            { updates.subject = subject; updatedFields.push("subject"); }
        if (description !== undefined && description !== currentTask.description)
            { updates.description = description; updatedFields.push("description"); }
        if (activeForm !== undefined && activeForm !== currentTask.activeForm)
            { updates.activeForm = activeForm; updatedFields.push("activeForm"); }
        if (owner !== undefined && owner !== currentTask.owner)
            { updates.owner = owner; updatedFields.push("owner"); }

        // Auto-assign owner when setting to in_progress in swarm mode
        if (d9() && status === "in_progress" && owner === undefined && !currentTask.owner) {
            let agentName = v8();  // getAgentName()
            if (agentName) { updates.owner = agentName; updatedFields.push("owner"); }
        }

        // Metadata: merge with null-deletion support
        if (metadata !== undefined) {
            let merged = { ...currentTask.metadata ?? {} };
            for (let [key, value] of Object.entries(metadata)) {
                if (value === null) delete merged[key];
                else merged[key] = value;
            }
            updates.metadata = merged;
            updatedFields.push("metadata");
        }

        // Handle status changes
        if (status !== undefined) {
            if (status === "deleted") {
                let success = f7R(listId, taskId);
                return { data: { success, taskId, updatedFields: success ? ["deleted"] : [],
                         error: success ? undefined : "Failed to delete task",
                         statusChange: success ? { from: currentTask.status, to: "deleted" } : undefined } };
            }
            if (status !== currentTask.status) {
                if (status === "completed") {
                    // Run completion hooks/verification before marking complete
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
                updates.status = status;
                updatedFields.push("status");
            }
        }

        // Apply dependency additions
        // (addBlocks/addBlockedBy are handled via rBA() -- see Section 7)

        // Persist the update
        Qw(listId, taskId, updates);
        return { data: { success: true, taskId, updatedFields,
                 statusChange: status ? { from: currentTask.status, to: status } : undefined } };
    }
};
```

### 6.4 TaskList

```javascript
var WZB = {
    name: G0T,  // "TaskList" constant

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

    description() { return "List all tasks in the task list" },
    userFacingName() { return "TaskList" },
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
    },

    mapToolResultToToolResultBlockParam(T, R) {
        let { tasks } = T;
        if (tasks.length === 0)
            return { tool_use_id: R, type: "tool_result", content: "No tasks found" };
        let lines = tasks.map((t) => {
            let ownerStr = t.owner ? ` (${t.owner})` : "";
            let blockedStr = t.blockedBy.length > 0
                ? ` [blocked by ${t.blockedBy.map(b => `#${b}`).join(", ")}]` : "";
            return `#${t.id} [${t.status}] ${t.subject}${ownerStr}${blockedStr}`;
        });
        return { tool_use_id: R, type: "tool_result", content: lines.join("\n") };
    }
};
```

---

## 7. Task Dependency Resolution

### 7.1 Adding Dependencies

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

### 7.2 Dependency Checking During Claims

When an agent tries to claim a task (via `oBA`), blockers are checked:

```javascript
// Get all non-completed task IDs
let allTasks = D5(listId);
let activeIds = new Set(
    allTasks.filter((t) => t.status !== "completed").map((t) => t.id)
);

// Check if any blockers are still active
let activeBlockers = task.blockedBy.filter((id) => activeIds.has(id));
if (activeBlockers.length > 0)
    return { success: false, reason: "blocked", task, blockedByTasks: activeBlockers };
```

### 7.3 Cleanup on Deletion

When a task is deleted, all references are cleaned up:

```javascript
// In f7R (delete function):
let allTasks = D5(listId);
for (let task of allTasks) {
    let newBlocks = task.blocks.filter((q) => q !== deletedId);
    let newBlockedBy = task.blockedBy.filter((q) => q !== deletedId);
    if (newBlocks.length !== task.blocks.length || newBlockedBy.length !== task.blockedBy.length)
        Qw(listId, task.id, { blocks: newBlocks, blockedBy: newBlockedBy });
}
```

### 7.4 Smart Display in TaskList

TaskList filters out completed tasks from `blockedBy` arrays in its output so agents see only active blockers:

```javascript
blockedBy: t.blockedBy.filter((dep) => !completedIds.has(dep))
```

---

## 8. Task Claiming & Ownership

### 8.1 Basic Claim (`oBA`)

```javascript
function oBA(T, R, A, _ = {}) {
    // T = listId, R = taskId, A = agentName, _ = options
    let taskPath = LfT(T, R);
    if (!fs.existsSync(taskPath))
        return { success: false, reason: "task_not_found" };

    if (_.checkAgentBusy) return SyD(T, R, A);  // Use stricter check

    let lock;
    try {
        lock = lockfileSync(taskPath);
        let task = bu(T, R);
        if (!task) return { success: false, reason: "task_not_found" };

        // Already owned by someone else
        if (task.owner && task.owner !== A)
            return { success: false, reason: "already_claimed", task };

        // Already completed
        if (task.status === "completed")
            return { success: false, reason: "already_resolved", task };

        // Check dependency blockers
        let allTasks = D5(T);
        let activeIds = new Set(allTasks.filter(t => t.status !== "completed").map(t => t.id));
        let blockers = task.blockedBy.filter(id => activeIds.has(id));
        if (blockers.length > 0)
            return { success: false, reason: "blocked", task, blockedByTasks: blockers };

        // Claim it!
        return { success: true, task: Qw(T, R, { owner: A }) };
    } finally {
        if (lock) lock();
    }
}
```

### 8.2 Claim with Busy Check (`SyD`)

A stricter version that prevents an agent from claiming multiple tasks:

```javascript
function SyD(T, R, A) {
    let lockPath = yo9(T), lock;
    try {
        lock = lockfileSync(lockPath);  // Global lock (not per-task)
        let allTasks = D5(T);
        let task = allTasks.find(t => t.id === R);

        // ... same checks as oBA ...

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

### 8.3 Claim Failure Reasons

| Reason | Meaning |
|--------|---------|
| `task_not_found` | Task ID does not exist |
| `already_claimed` | Another agent owns this task |
| `already_resolved` | Task is already completed |
| `blocked` | Task has unresolved blockers |
| `agent_busy` | Agent already owns another active task (strict mode) |

### 8.4 Unassign on Agent Shutdown (`qn`)

When an agent shuts down or is terminated:

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

---

## 9. Highwatermark File

### 9.1 Purpose

The `.highwatermark` file at `~/.claude/tasks/{team-name}/.highwatermark` preserves the highest task ID ever assigned to this task list, even after tasks are deleted. This ensures that new task IDs are always unique and monotonically increasing, even across clear-all operations.

### 9.2 Constant

```javascript
var byD = ".highwatermark";
```

### 9.3 Read/Write

```javascript
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
        return 0;
    }
}

// Write new highwatermark value
function wo9(T, R) {
    let A = Vo9(T);
    writeFileSync(A, String(R));
}
```

### 9.4 Usage

The highwatermark is consulted in two places:

1. **Creating a new task** (`WOT`): The next ID is `max(highest_file_id, highwatermark) + 1`
2. **Deleting a task** (`f7R`): If the deleted task's ID exceeds the current highwatermark, update it
3. **Clearing all tasks** (`z7R`): Before deleting files, the highest file-based ID is written to the highwatermark

```javascript
function kyD(T) {
    let R = Po9(T);   // Highest ID from .json files
    let A = aBA(T);   // Highwatermark value
    return Math.max(R, A);
}
```

---

## 10. Task ID Generation

Task IDs are **sequential integers stored as strings**:

```javascript
function WOT(T, R) {
    let lockPath = yo9(T), lock;
    try {
        lock = lockfileSync(lockPath);
        let currentMax = kyD(T);     // max(file IDs, highwatermark)
        let newId = String(currentMax + 1);  // "1", "2", "3", ...
        let task = { id: newId, ...R };
        let filePath = LfT(T, newId);
        writeFileSync(filePath, JSON.stringify(task, null, 2));
        COT();  // Notify listeners
        return newId;
    } finally {
        if (lock) lock();
    }
}
```

The highest existing ID is determined by scanning `.json` filenames in the task directory:

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

---

## 11. Task Notifications & Queue System

### 11.1 Change Listeners

```javascript
var iBA = new Set();  // Set of listener functions

function fo9(T) {     // Register a listener
    iBA.add(T);
    return () => iBA.delete(T);
}

function COT() {      // Notify all listeners
    for (let T of iBA) {
        try { T(); } catch {}
    }
}
```

These listeners are called after every task mutation (create, update, delete, clear).

### 11.2 Task Notification Tags

Task notifications use XML-like tags in the message stream:

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

### 11.3 Notification Queue

```javascript
function NG(T, R) {
    // If this is a task-notification and there are listeners, add to priority queue
    if (T.mode === "task-notification" && D5R.size > 0) {
        n2T.push(typeof T.value === "string" ? T.value : "");
        H5R();  // Increment counter and notify
    } else {
        // Regular queued command
        R((A) => ({ ...A, queuedCommands: [...A.queuedCommands, T] }));
    }
}
```

---

## 12. Task UI Rendering

### 12.1 Expanded View

When a task tool is called, it sets `expandedView: "tasks"` in the app state:

```javascript
context.setAppState(($) => {
    if ($.expandedView === "tasks") return $;
    return { ...$, expandedView: "tasks" };
});
```

### 12.2 Teammate Status Display

Each teammate in the team view shows:
- Agent name with color
- Status (running/idle/stopping/awaiting approval)
- Progress: tool use count and token count
- Recent activity description
- Duration

```javascript
// Status colors:
// "running" -> "warning" (yellow)
// "completed" -> "success" (green)
// "failed" -> "error" (red)
// other -> "inactive" (dim)

// Progress display:
let progressStr = progress ? ` (${progress.toolUseCount} tools, ${progress.tokenCount} tokens)` : "";
```

### 12.3 Task List Output Format

The `mapToolResultToToolResultBlockParam` in TaskList formats output as:

```
#1 [pending] Fix authentication bug
#2 [in_progress] Write unit tests (agent-1)
#3 [pending] Deploy to staging [blocked by #2]
```

---

## 13. Background Task System (Local Agents)

Separate from the shared task list, Claude Code has a background task system for running local agents and bash commands.

### 13.1 Task Types

```javascript
var ML7 = {
    local_bash: "b",           // Background bash commands
    local_agent: "a",          // Background agent sessions
    remote_agent: "r",         // Remote agent sessions
    in_process_teammate: "t"   // In-process teammate agents
};
```

### 13.2 Task ID Generation (Background Tasks)

Background task IDs use a different scheme -- a type prefix + random UUID segment:

```javascript
function Kc(T) {  // T = task type string
    let prefix = ML7[T] ?? "x";
    let random = crypto.randomUUID().replace(/-/g, "").substring(0, 6);
    return `${prefix}${random}`;  // e.g., "b3f7a2c", "t9e1d4b"
}
```

### 13.3 Task Output Files

Background tasks write output to files at `~/.claude/tasks/{taskId}.output`:

```javascript
function b4(T) {
    return path.join(rkT(), `${T}.output`);
}

// Append output
function $0T(T, R) {
    // Ensures directory exists, then appends
    await fs.appendFile(outputPath, R, "utf8");
}

// Read output with offset (for incremental reads)
function hQA(T, R) {
    // Returns { content: string, newOffset: number }
}
```

### 13.4 Task State Machine

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

---

## 14. TaskOutput Tool

The `TaskOutput` tool retrieves output from background tasks:

```javascript
var NIR = {
    name: "TaskOutput",
    maxResultSizeChars: 1e5,
    aliases: ["AgentOutputTool", "BashOutputTool"],

    userFacingName() { return "Task Output" },

    inputSchema: z.strictObject({
        task_id: z.string().describe("The task ID to get output from"),
        block: z.boolean().default(true).describe("Whether to wait for completion"),
        timeout: z.number().min(0).max(600000).default(30000).describe("Max wait time in ms")
    }),

    description() { return "Retrieves output from a running or completed task" },
    isReadOnly() { return true },
    isEnabled() { return true },  // Always available

    async prompt() {
        return "- Retrieves output from a running or completed task " +
               "(background shell, agent, or remote session)\n" +
               "- Returns the task status and any available output\n" +
               "- Can optionally block until the task completes\n" +
               "- Returns a success or failure status\n" +
               "- Use this tool when you need to terminate a long-running task";
    }
};
```

The TaskOutput tool supports three retrieval states:
- `"success"` -- output available and returned
- `"timeout"` / `"not_ready"` -- task still running
- Various error states for failed/killed tasks

---

## 15. Feature Flags & Enablement Logic

### 15.1 Primary Feature Gate

```javascript
function sq() {
    if (_H(process.env.CLAUDE_CODE_ENABLE_TASKS)) return false;
    if (AR(process.env.CLAUDE_CODE_ENABLE_TASKS)) return true;
    if (e_()) return false;  // Non-interactive -> disabled
    return true;             // Default: enabled
}
```

- `_H()` checks for falsy string values ("false", "0", "no", etc.)
- `AR()` checks for truthy string values ("true", "1", "yes", etc.)
- `e_()` detects non-interactive mode

### 15.2 Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_ENABLE_TASKS` | Master toggle for the Tasks system |
| `CLAUDE_CODE_TASK_LIST_ID` | Override the task list identifier |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | Force plan mode for teammates |

### 15.3 Swarm Mode Detection

```javascript
function d9() {
    // Returns true if running in swarm/team mode
    // (checks for team context, parent session, etc.)
}

// Auto-assign owner when setting task to in_progress in swarm mode:
if (d9() && status === "in_progress" && owner === undefined && !currentTask.owner) {
    let agentName = v8();
    if (agentName) updates.owner = agentName;
}
```

---

## 16. Obfuscated Symbol Map

Key variable/function mappings discovered in the binary:

| Obfuscated | Actual Purpose |
|-----------|---------------|
| `fyD` | Todo status enum (`z.enum(["pending", "in_progress", "completed"])`) |
| `VyD` | Todo item schema |
| `OOT` | Todo array schema (`z.array(VyD)`) |
| `yu` | TodoWrite tool name constant (`"TodoWrite"`) |
| `ZO` | TodoWrite tool definition object |
| `ho9` | TodoWrite prompt text |
| `Eo9` | TodoWrite description text |
| `R9T` | Task status enum |
| `yyD` | Task schema (Zod) |
| `byD` | Highwatermark filename constant (`".highwatermark"`) |
| `sBA` | Tasklist constant (`"tasklist"`) |
| `FP` | TaskCreate tool name |
| `O0T` | TaskGet tool name |
| `Yz` | TaskUpdate tool name |
| `G0T` | TaskList tool name |
| `r2T` | TaskOutput tool name (`"TaskOutput"`) |
| `uIB` | TaskCreate tool definition |
| `aIB` | TaskGet tool definition |
| `_ZB` | TaskUpdate tool definition |
| `WZB` | TaskList tool definition |
| `NIR` | TaskOutput tool definition |
| `WOT` | Create task function |
| `bu` | Read task function |
| `Qw` | Update task function |
| `f7R` | Delete task function |
| `D5` | List all tasks function |
| `z7R` | Clear all tasks function |
| `nW` | Get current task list ID |
| `zF` | Get task list directory path |
| `LfT` | Get task file path |
| `KfT` | Ensure task directory exists |
| `Po9` | Get highest task ID from files |
| `kyD` | Get highest task ID (files + highwatermark) |
| `aBA` | Read highwatermark |
| `wo9` | Write highwatermark |
| `Vo9` | Get highwatermark file path |
| `JOT` | Sanitize name for filesystem |
| `oBA` | Claim task (basic) |
| `SyD` | Claim task (with agent busy check) |
| `rBA` | Add dependency between tasks |
| `qn` | Unassign tasks on agent shutdown |
| `sq` | Is Tasks feature enabled |
| `COT` | Notify task change listeners |
| `iBA` | Task change listener set |
| `nBA` | Manually set task list name |
| `bR` | UUIDv7 generator -- session ID fallback in `nW()` |
| `EfT` | proper-lockfile module |
| `T9T` | path module |
| `eG` | fs module |
| `mBA` | AsyncLocalStorage for teammate context |
| `FF` | Get teammate context (from AsyncLocalStorage) |
| `H2` | Get agent ID |
| `v8` | Get agent name |
| `v7` | Get team name |
| `a1` | Is teammate (boolean) |
| `iW` | Is team lead |
| `d9` | Is swarm mode active |
| `Kc` | Generate background task ID |
| `ML7` | Task type prefix map |
| `$O` | Task notification tag name (`"task-notification"`) |
| `EX` | Task ID tag name (`"task-id"`) |
| `DsT` | Task type tag name (`"task-type"`) |
| `a5` | Status tag name (`"status"`) |
| `r5` | Summary tag name (`"summary"`) |
| `NG` | Enqueue command/notification |
| `b4` | Get background task output file path |
| `$0T` | Append to background task output |
| `hQA` | Read background task output with offset |
| `EL` | Create background task state object |
| `evT` | Task completion verification hook iterator |
| `tvT` | Format blocking error message |

---

## 17. Complete Function Reference

### Persistence Layer (`Gq` module)

| Function | Signature | Purpose |
|----------|-----------|---------|
| `WOT(listId, taskData)` | `(string, object) -> string` | Create task, returns new ID |
| `bu(listId, taskId)` | `(string, string) -> Task \| null` | Read single task |
| `Qw(listId, taskId, updates)` | `(string, string, object) -> Task \| null` | Update task fields |
| `f7R(listId, taskId)` | `(string, string) -> boolean` | Delete task |
| `D5(listId)` | `(string) -> Task[]` | List all tasks |
| `z7R(listId)` | `(string) -> void` | Clear all tasks |
| `rBA(listId, blockerId, blockedId)` | `(string, string, string) -> boolean` | Add dependency |
| `oBA(listId, taskId, agent, opts)` | `(...) -> ClaimResult` | Claim task |
| `SyD(listId, taskId, agent)` | `(string, string, string) -> ClaimResult` | Claim with busy check |
| `qn(listId, agentId, agentName, reason)` | `(...) -> UnassignResult` | Unassign on shutdown |
| `nW()` | `() -> string` | Get current task list ID (falls back to `bR()` in solo sessions) |
| `bR()` | `() -> string` | Generate UUIDv7 (session ID / task list fallback) |
| `zF(listId)` | `(string) -> string` | Get task directory path |
| `LfT(listId, taskId)` | `(string, string) -> string` | Get task file path |
| `KfT(listId)` | `(string) -> void` | Ensure directory exists |
| `Po9(listId)` | `(string) -> number` | Highest file-based ID |
| `kyD(listId)` | `(string) -> number` | Highest ID (files + watermark) |
| `aBA(listId)` | `(string) -> number` | Read highwatermark |
| `wo9(listId, value)` | `(string, number) -> void` | Write highwatermark |
| `JOT(name)` | `(string) -> string` | Sanitize for filesystem |
| `sq()` | `() -> boolean` | Is Tasks feature enabled |
| `fo9(listener)` | `(fn) -> unsubscribe` | Register change listener |
| `COT()` | `() -> void` | Notify all listeners |

### Tool Definitions

| Variable | Tool Name | Type |
|----------|-----------|------|
| `ZO` | `TodoWrite` | Solo todo list management |
| `uIB` | `TaskCreate` | Create shared task |
| `aIB` | `TaskGet` | Read shared task |
| `_ZB` | `TaskUpdate` | Update/delete shared task |
| `WZB` | `TaskList` | List all shared tasks |
| `NIR` | `TaskOutput` | Read background task output |

### Background Task Functions

| Function | Purpose |
|----------|---------|
| `Kc(type)` | Generate task ID with type prefix |
| `EL(id, type, desc)` | Create task state object |
| `b4(taskId)` | Get output file path |
| `$0T(taskId, content)` | Append to output file |
| `hQA(taskId, offset)` | Read output from offset |
| `_5R(taskId)` | Read full output |
| `l2T(taskId)` | Create empty output file |
| `i2T(taskId, target)` | Create symlink to output |
| `Ek_()` | Clean up all output files |

---

*Analysis generated from Claude Code v2.1.34 binary (build 2026-02-06T06:37:05Z).*
*All code snippets are deobfuscated reconstructions from `strings` output with variable names mapped to their functional equivalents.*
