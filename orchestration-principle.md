# Orchestration Principle: Deterministic Systems, Agent Workers

**Date**: 2026-02-08

---

## The Problem

Binary analysis of Claude Code v2.1.34 reveals a sophisticated task system: file-persisted JSON, atomic claiming with file locks, dependency enforcement, multi-agent coordination with auto-unassign on shutdown. Within a session, it works well.

But the analysis also reveals fundamental limitations:

- **Session-scoped**: Task lists are keyed to random UUIDs (`bR()` generates a UUIDv7 per session). After a session ends, tasks become orphaned -- files persist on disk but are undiscoverable.
- **Undocumented**: The file paths (`~/.claude/tasks/`), data formats, and session index (`sessions-index.json`) are internal implementation details. No public API, no stability guarantees.
- **Vendor-specific**: Every aspect -- the file layout, the tools (TaskCreate/TaskList/etc.), the environment variables (`CLAUDE_CODE_TASK_LIST_ID`) -- is specific to Claude Code.
- **Crash-fragile**: A session crash leaves tasks stuck in `in_progress` permanently. The `qn()` cleanup function only runs during graceful shutdowns.

The temptation is to build tooling on top of these internals -- a discovery script that scans `~/.claude/tasks/`, cross-references `sessions-index.json`, and generates resume commands. This was, in fact, the recommendation of the previous analysis revision.

This document argues that this temptation should be resisted.

---

## Three Strategic Risks

### 1. Internal API Fragility

`~/.claude/tasks/`, `sessions-index.json`, and `CLAUDE_CODE_TASK_LIST_ID` are undocumented implementation details. Anthropic has not exposed tasks as an external interface -- there is no `claude tasks list` CLI command, no public API, no documentation of the JSON schema.

This is not an oversight. It signals intent to maintain freedom to change these internals. The task system is new (swarm mode appeared in recent versions) and actively evolving. File paths may move. JSON schemas may change. The `sessions-index.json` format may be restructured or removed entirely.

A tool built on these internals today may break silently with any update. There will be no deprecation warning, no migration path, and no guarantee that the data even exists in the next version.

### 2. Vendor Lock-in

Claude Code is one agent platform among many. Codex, Gemini CLI, open-source agent frameworks (Aider, Continue, etc.) each have their own task systems -- or none at all. The agent landscape is evolving rapidly; today's dominant tool may not be tomorrow's.

Orchestration logic coupled to Claude Code's internals is non-portable:

- If a better agent tool emerges, you cannot bring your orchestration with you.
- If frontier model companies raise prices, you cannot easily switch providers.
- If you want to use different agents for different tasks (Claude for complex reasoning, a cheaper model for boilerplate), your orchestration system must be agent-independent.

Building on `~/.claude/tasks/` means building on Claude Code specifically. Not on Claude the model, not on Anthropic's API -- on one specific CLI tool's internal file structure.

### 3. Agent Unpredictability for Orchestration

Agents excel at reasoning, code generation, creative problem-solving, and analysis. They are unreliable for deterministic workflows:

- **Running test suites**: An agent might skip tests, run the wrong ones, or misinterpret results. A shell script always runs exactly what you specify.
- **Committing to git**: An agent might stage the wrong files, write an incorrect commit message, or forget to push. A script does exactly what it's told.
- **Deciding what to work on next**: An agent might pick the wrong priority, forget about blocked tasks, or re-do work that was already completed. A deterministic scheduler consults the state and makes the correct decision every time.
- **Tracking progress across sessions**: An agent's context window is finite. It forgets. It hallucinates. A database or file on disk does not.

These are not edge cases -- they are inherent properties of probabilistic systems. Agents are powerful tools, but predictable workflow execution is not their strength.

---

## The Principle

### Orchestration belongs in your own deterministic system.

It decides what to do next. It tracks progress. It handles retries. It recovers from crashes. It runs tests. It commits code. It knows what was done and what remains.

### Agents are workers.

They receive well-scoped tasks. They do the thinking-heavy work: code generation, debugging, analysis, creative problem-solving. They return results. They do not decide the workflow.

### Agent-internal task systems are convenience layers.

Claude Code's TaskCreate/TaskList/TaskUpdate/TaskGet are useful for the agent to organize its own work within a session. An agent breaking a complex coding task into subtasks and tracking them is a good use of these tools. They are not the source of truth. If the agent crashes, the orchestrator still knows what was in progress and can re-dispatch.

### The orchestrator is vendor-independent.

It talks to agents through their CLI interfaces (`claude`, `codex`, etc.), not through internal file structures. Switching agents means changing a command, not rewriting orchestration logic. The orchestrator's state lives in your own system -- files in your repo, a SQLite database, a YAML task file, whatever suits your needs. Not in `~/.claude/`.

---

## What This Means for Agent Task Systems

### Claude Code's built-in tasks

A useful within-session helper. Let agents use TaskCreate/TaskList to organize their own work during a session. Don't depend on it surviving across sessions. Don't build tooling that reads `~/.claude/tasks/` directly. If you need cross-session continuity, put it in your own system.

### Beads (or similar tools)

A possible implementation of the orchestration layer, not a complement to Claude's task system. Evaluate Beads on its own merits as an orchestrator: Does it provide vendor-independent task tracking? Does it handle crash recovery? Does it work with agents other than Claude? These are the right questions -- not "how does it integrate with Claude Code's internal task files?"

### The `claude-tasks` discovery tool (from earlier analysis)

A tactical band-aid, not a strategic solution. It deepens coupling to Claude Code internals rather than reducing it. It reads undocumented file paths, parses undocumented JSON formats, and cross-references undocumented session indexes. It solves a real immediate problem (finding orphaned tasks), but it does so by building on the exact foundation this principle argues against.

If you build it, do so with eyes open about the coupling cost. It is a short-term convenience, not a long-term investment.

---

## Implications

**Don't**:
- Build on `~/.claude/tasks/` or `sessions-index.json` as infrastructure
- Rely on `--resume` or `CLAUDE_CODE_TASK_LIST_ID` as your recovery mechanism
- Treat Claude Code's task system as a foundation for cross-session workflow management
- Couple your orchestration logic to any single agent's internal state

**Do**:
- Let Claude use TaskCreate/TaskList within sessions -- it helps the agent organize work
- Build (or adopt) your own system that tracks what agents should be working on
- Keep your task state independent of any agent's internal state
- Talk to agents through their CLI interfaces, not their internal file structures
- Put your source of truth somewhere you control: your repo, your database, your files

---

*This principle emerged from binary analysis of Claude Code v2.1.34's task system. See `task-system-source-analysis.md` for the technical details and `next-step.md` for practical approaches.*
