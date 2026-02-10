# Claude Code Auto Memory System - Deep Source Analysis (v2.1.34)

## Executive Summary

Claude Code's "auto memory" is a **feature-flagged persistent memory system** that gives Claude a writable directory of Markdown files whose `MEMORY.md` is injected into the system prompt on every turn. There is **no dedicated MemoryWrite tool** -- the model uses the standard `Write` and `Edit` tools to update its own memory files. The memory is read from disk on every API turn (not cached), making it a "live" part of the system prompt.

There are actually **two distinct memory systems** in Claude Code v2.1.34:
1. **Auto Memory** (feature flag: `tengu_oboe`) -- per-project memory stored in `~/.claude/projects/<encoded-path>/memory/MEMORY.md`
2. **Agent Memory** (for plugin agents) -- per-agent memory with three scopes: user, project, local

---

## 1. Filesystem Structure

### Auto Memory Location
```
~/.claude/projects/<encoded-project-path>/memory/MEMORY.md
```

Example for this project:
```
~/.claude/projects/-Users-Michael-Arnoldus-Workspace-task-analysis/memory/MEMORY.md
```

The `memory/` directory and `MEMORY.md` file are auto-created by the system if they don't exist.

### Agent Memory Locations (Plugin Agents Only)
```
~/.claude/agent-memory/<agent-type>/MEMORY.md          # user scope
<project>/.claude/agent-memory/<agent-type>/MEMORY.md   # project scope
<project>/.claude/agent-memory-local/<agent-type>/MEMORY.md  # local scope
```

### No Other Memory Files Found
- Only one project (`task-analysis`) had a `memory/` directory across all 11 projects in `~/.claude/projects/`
- This confirms auto memory is feature-flagged and was only recently enabled

---

## 2. Feature Flag Control

### Primary Gate: `tengu_oboe`
```javascript
function hq() {
    if (AR(process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY))
        return false;
    return g9("tengu_oboe", false);
}
```

**Key findings:**
- `hq()` is the master gate function for auto memory
- Controlled by GrowthBook feature flag `tengu_oboe` (default: `false`)
- Can be disabled via environment variable `CLAUDE_CODE_DISABLE_AUTO_MEMORY`
- `g9()` checks the Statsig/GrowthBook feature flag system
- When `hq()` returns `false`, the memory system is completely invisible

### Display Name
```javascript
function Qc() {
    if (hq()) return "auto memory";
    return "";
}
```
When enabled, the display name is literally `"auto memory"`.

---

## 3. Path Encoding Scheme

### The `nIT` Function
```javascript
function nIT(T) {
    return T.replace(/[^a-zA-Z0-9]/g, "-");
}
```

This is the **entire path encoding logic**: every non-alphanumeric character in the absolute project path is replaced with a hyphen. So:
- `/Users/Michael.Arnoldus/Workspace/task-analysis`
- becomes `-Users-Michael-Arnoldus-Workspace-task-analysis`

### Project Directory Resolution
```javascript
function l6(T) {
    return dh.join(Xd(), nIT(T));
}

function Xd() {
    return dh.join(q9(), "projects");
}
```
Where `q9()` returns `~/.claude/` (the Claude config directory).

### Memory Path Construction
```javascript
function Xb_() { return "memory"; }
function NE7() { return "MEMORY.md"; }

function hb_() { return $J(lC()) ?? lC(); }

function xWA() {
    return lBT.join(l6(hb_()), Xb_()) + lBT.sep;
}

function gCR() {
    return lBT.join(l6(hb_()), Xb_(), NE7());
}
```

The `hb_()` function resolves the "original project dir" -- `$J(lC())` returns the original cwd if one was saved (for cases like `--cd` flag), otherwise falls back to `lC()` (current working directory). This means the memory directory is always tied to the **original project root**, not any directory you `cd` into during the session.

### Path Validation
```javascript
function ykT(T) {
    return lBT.normalize(T).startsWith(xWA());
}
```
This checks if a given path is inside the memory directory -- used for permission/validation checks.

---

## 4. Memory Read Mechanism

### System Prompt Integration
Memory is registered as a **cache-breaking system prompt component**:

```javascript
Td("auto_memory", () => uWA(),
   "MEMORY.md is read from disk each turn and can be edited by the model")
```

The `Td()` function creates a system prompt component with `cacheBreak: true`, meaning:
- The content is **re-read from disk on every turn** (not cached across turns)
- Changes made by the model via Write/Edit tools are immediately reflected in the next turn's system prompt

### The `uWA()` Entry Point
```javascript
function uWA() {
    if (hq())
        return vWA({ displayName: Qc(), memoryDir: xWA() });
    return null;
}
```
Returns `null` (no memory prompt) if the feature is disabled.

### The `vWA()` Prompt Generator
```javascript
function vWA(T) {
    let {displayName: R, memoryDir: A, extraGuidelines: _} = T;
    let B = fT();       // get filesystem
    let D = A + xr;      // memoryDir + "MEMORY.md"

    try { B.mkdirSync(A); } catch {}  // auto-create directory

    let $ = "";
    try { $ = B.readFileSync(D, {encoding: "utf-8"}); } catch {}

    let H = [
        `# ${R}`,
        "",
        `You have a persistent ${R} directory at \`${A}\`. Its contents persist across conversations.`,
        "",
        `As you work, consult your memory files to build on previous experience. When you encounter a mistake that seems like it could be common, check your ${R} for relevant notes -- and if nothing is written yet, record what you learned.`,
        "",
        "Guidelines:",
        `- \`${xr}\` is always loaded into your system prompt -- lines after ${bkT} will be truncated, so keep it concise`,
        "- Create separate topic files (e.g., `debugging.md`, `patterns.md`) for detailed notes and link to them from MEMORY.md",
        "- Record insights about problem constraints, strategies that worked or failed, and lessons learned",
        "- Update or remove memories that turn out to be wrong or outdated",
        "- Organize memory semantically by topic, not chronologically",
        "- Use the Write and Edit tools to update your memory files",
        ..._ ?? [],
        ""
    ];

    if ($.trim()) {
        let q = $.trim().split("\n");
        // ... truncation logic and content append
    }
    // ... returns system prompt string
}
```

**Critical constants:**
```javascript
var xr = "MEMORY.md";
var bkT = 200;      // line truncation limit
```

### The 200-Line Truncation
The guideline text says: "lines after 200 will be truncated, so keep it concise". The `bkT` constant is `200`. The system prompt instructs the model that MEMORY.md content after line 200 will be truncated. The content is split by newlines and only the first 200 lines are included in the system prompt.

### CC() -- The Master File Collector
The MEMORY.md file is **also loaded through the CC() function**, which collects all CLAUDE.md and rule files:

```javascript
CC = oR((T = false) => {
    // ... loads Managed, User, Project CLAUDE.md files ...
    // ... loads .claude/rules/ directories ...

    // At the very end:
    if (hq()) {
        let G = HQA(gCR(), "AutoMem");
        if (G && !_.has(G.path))
            _.add(G.path), A.push(G);
    }

    return A;
});
```

This means the MEMORY.md file is loaded **both**:
1. As a `claudeMd`-style file in the `CC()` collector (typed as `"AutoMem"`)
2. As a separate `auto_memory` system prompt component via `uWA()`

The `CC()` path provides it as part of the `<system-reminder>` injection mechanism (the `claudeMd` context that appears in conversations), while the `uWA()` path provides it as a separate system prompt section with the memory guidelines.

---

## 5. Memory Write Mechanism

### No Dedicated Tool
There is **no MemoryWrite tool**. The system prompt explicitly tells the model:

> "Use the Write and Edit tools to update your memory files"

The model writes to memory by using the standard `Write` and `Edit` tools to modify files in the memory directory. Since the memory directory path is given in the system prompt, and the `ykT()` function validates memory paths, the system can recognize when the model is writing to its own memory.

### Auto-Creation
The memory directory is auto-created via `mkdirSync()` every time `vWA()` runs (which is every turn), with a try/catch to silently ignore if it already exists. The MEMORY.md file itself is not pre-created -- the model creates it on its first write.

---

## 6. CLAUDE.md vs Memory -- Separate but Intersecting

### CLAUDE.md System
- Loaded by `CC()` (memoized, re-evaluated per session clear)
- Sources: `CLAUDE.md`, `.claude/CLAUDE.md`, `CLAUDE.local.md`, `.claude/rules/*.md`
- Walks **up the directory tree** from cwd to root
- Supports `@include` directives for transclusion
- Content injected into `<system-reminder>` blocks
- Types: `"Managed"`, `"User"`, `"Project"`, `"Local"`
- Loaded by `SO` (user context) at session start
- Character limit: 40,000 per file (`Ec` constant)

### Auto Memory System
- Loaded by both `CC()` (as type `"AutoMem"`) and `uWA()` (separate prompt)
- Single source: `~/.claude/projects/<path>/memory/MEMORY.md`
- **Cache-breaking**: re-read from disk every turn (CLAUDE.md is read once per session unless cleared)
- Line limit: 200 lines in system prompt
- Writable by the model (CLAUDE.md is read-only to the model in normal operation)
- Feature-flagged behind `tengu_oboe`

### Key Difference: Caching Behavior
```javascript
// CLAUDE.md: cached (em = static component)
SO = oR(async () => {
    let A = R ? null : _k_();
    return { ...A ? {claudeMd: A} : {} };
});

// Memory: re-read every turn (Td = cache-breaking component)
Td("auto_memory", () => uWA(),
   "MEMORY.md is read from disk each turn and can be edited by the model")
```

The `em()` function creates a **static** system prompt component (cached for the session), while `Td()` creates a **cache-breaking** component that re-evaluates on every turn. This is critical because the model can modify MEMORY.md mid-conversation and see the changes on the next turn.

---

## 7. Agent Memory System (Plugin Agents)

A separate memory system exists for **plugin-defined agents**:

### Three Scopes
```javascript
Lb_ = ["user", "project", "local"];
```

### Path Construction
```javascript
function mWA(T, R) {        // T = agent type, R = scope
    let A = IE7(T);          // replace colons with hyphens
    switch (R) {
        case "project": return path.join(ZR(), ".claude", "agent-memory", A) + sep;
        case "local":   return path.join(ZR(), ".claude", "agent-memory-local", A) + sep;
        case "user":    return path.join(os.homedir(), ".claude", "agent-memory", A) + sep;
    }
}
```

### Scope Guidelines
```javascript
function iBT(T, R) {
    // ... rename memory.md -> MEMORY.md if needed (migration) ...
    let A;
    switch (R) {
        case "user":
            A = "Since this memory is user-scope, keep learnings general since they apply across all projects";
            break;
        case "project":
            A = "Since this memory is project-scope and shared with your team via version control, tailor your memories to this project";
            break;
        case "local":
            A = "Since this memory is local-scope (not checked into version control), tailor your memories to this project and machine";
            break;
    }
    return vWA({ displayName: "Persistent Agent Memory", memoryDir: mWA(T, R), extraGuidelines: [A] });
}
```

### Agent Tool Injection
When auto memory is enabled and a plugin agent has a memory scope defined:
```javascript
if (hq() && Y && W !== undefined) {
    let F = new Set(W);
    for (let V of [O8, Y0, eB])  // Write, Edit, Read tools
        if (!F.has(V)) W = [...W, V];
}
```
The system automatically adds `Write`, `Edit`, and `Read` tools to agents that have memory enabled, ensuring they can read/write their memory files.

---

## 8. Configuration & Environment Variables

| Setting | Effect |
|---------|--------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Env var. If truthy, disables auto memory entirely |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Env var. If truthy, disables ALL CLAUDE.md loading (including memory in CC path) |
| `CLAUDE_CODE_SIMPLE` | Env var. If truthy, skips entire system prompt (returns minimal prompt) |
| `tengu_oboe` | GrowthBook feature flag. Master control for auto memory (default: false) |
| `tengu_coral_fern` | GrowthBook feature flag. Controls "session memory" (past session access) |
| `tengu_vinteuil_phrase` | GrowthBook feature flag. Controls simplified system prompt path |

No user-facing settings in `settings.json` or `settings.local.json` control memory. The feature is entirely driven by server-side feature flags.

---

## 9. Scratchpad System (Related)

A separate but related system is the "scratchpad directory":
```javascript
function RaB() {
    if (!ALT()) return null;
    return `# Scratchpad Directory
    // ... scratchpad instructions ...
}
```
This appears to be gated by a different feature flag (`ALT()`). It provides a temporary per-session scratch space, distinct from persistent memory.

---

## 10. System Prompt Assembly Order

The full system prompt is assembled in `rN()` (or `pnB()` for the simplified path):

```javascript
let W = [
    em("session_memory",  () => nnB()),           // past session access
    Td("auto_memory",     () => uWA(), ...),       // MEMORY.md (cache-breaking)
    Td("ant_model_override", () => anB(), ...),     // model override
    em("env_info",        () => FjA(R, A)),         // environment info
    em("language",        () => rnB(O.language)),    // language setting
    Td("output_style",   () => onB(H), ...),        // output style
    Td("mcp_instructions", () => snB(_), ...),      // MCP instructions
    em("scratchpad",      () => RaB())              // scratchpad
];
```

Memory is the **second component** in the dynamic system prompt section, after session memory. The `Td` components are re-evaluated every turn; `em` components are cached.

---

## 11. Summary: How It All Works

1. **Session Start**: `CC()` loads all CLAUDE.md files + MEMORY.md (as type "AutoMem") into the claudeMd context that appears in `<system-reminder>` blocks
2. **Every Turn**: `uWA()` reads MEMORY.md from disk, builds the memory guidelines prompt, and includes the first 200 lines of MEMORY.md content as a cache-breaking system prompt component
3. **Model Writes**: The model uses standard `Write`/`Edit` tools to modify files in `~/.claude/projects/<encoded-path>/memory/`
4. **Next Turn**: The updated MEMORY.md is re-read from disk (cache-breaking) and the new content appears in the system prompt
5. **No Special Write Logic**: There is no "wrote N memories" tracking, no dedicated recall mechanism, no write triggers beyond the model's own agency. The model decides when to write based on the guidelines in its system prompt.

### What the Model Sees in Its System Prompt
```
# auto memory

You have a persistent auto memory directory at `~/.claude/projects/-Users-.../memory/`.
Its contents persist across conversations.

As you work, consult your memory files to build on previous experience.
When you encounter a mistake that seems like it could be common, check your
auto memory for relevant notes -- and if nothing is written yet, record what you learned.

Guidelines:
- `MEMORY.md` is always loaded into your system prompt -- lines after 200 will be truncated
- Create separate topic files (e.g., `debugging.md`, `patterns.md`) for detailed notes
- Record insights about problem constraints, strategies that worked or failed, and lessons learned
- Update or remove memories that turn out to be wrong or outdated
- Organize memory semantically by topic, not chronologically
- Use the Write and Edit tools to update your memory files

[Current MEMORY.md content here, first 200 lines]
```

---

## Appendix: Key Function Reference

| Minified Name | Purpose |
|--------------|---------|
| `hq()` | Master gate: is auto memory enabled? |
| `nIT(path)` | Path encoding: replace non-alphanumeric with `-` |
| `l6(path)` | Get project directory in `~/.claude/projects/` |
| `xWA()` | Get memory directory path |
| `gCR()` | Get MEMORY.md file path |
| `ykT(path)` | Check if path is inside memory directory |
| `vWA(opts)` | Build memory system prompt section |
| `uWA()` | Entry point: get auto memory prompt (or null) |
| `CC()` | Master file collector for CLAUDE.md + memory |
| `_k_()` | Format all CLAUDE.md files for system prompt |
| `em(name, fn)` | Register static system prompt component |
| `Td(name, fn, desc)` | Register cache-breaking system prompt component |
| `mWA(agent, scope)` | Get agent memory directory path |
| `iBT(agent, scope)` | Build agent memory system prompt |
| `Xb_()` | Returns `"memory"` (directory name constant) |
| `NE7()` | Returns `"MEMORY.md"` (file name constant) |
| `Qc()` | Returns `"auto memory"` display name (or empty) |
| `bkT` | Line truncation limit constant: `200` |
| `xr` | MEMORY.md filename constant |
| `Lb_` | Valid agent memory scopes: `["user", "project", "local"]` |
