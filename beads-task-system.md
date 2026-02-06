# Beads Task Management System

## Executive Summary

**Beads** is Steve Yegge's git-native, agent-optimized issue tracking and task management system. It stores issues directly in git repositories as JSONL files, uses SQLite for fast local queries, and provides a CLI (`bd`) designed for both human and AI agent use. Beads goes far beyond simple task tracking — it encompasses multi-project management, multi-agent coordination, workflow execution (via "molecules"), cross-organization federation, and semantic memory decay (compaction).

The system is fundamentally designed for AI agents: every command supports `--json` output, hash-based IDs prevent multi-agent conflicts, and the dependency graph drives a `bd ready` command that computes available work.

---

## Table of Contents

1. [Core Philosophy](#core-philosophy)
2. [Architecture](#architecture)
3. [Data Model](#data-model)
4. [Storage Backends](#storage-backends)
5. [CLI and Agent Interface](#cli-and-agent-interface)
6. [Multi-Project Management](#multi-project-management)
7. [AI Agent Integration](#ai-agent-integration)
8. [Dependency System and Ready Work](#dependency-system-and-ready-work)
9. [Molecules: Structured Workflows](#molecules-structured-workflows)
10. [Compaction and Memory Decay](#compaction-and-memory-decay)
11. [Federation](#federation)
12. [Key Features and Differentiators](#key-features-and-differentiators)

---

## Core Philosophy

Beads is built on several key principles:

1. **Git-native**: Issues live in git, travel with code branches, work offline, and have full version history
2. **Agent-first**: Every design decision prioritizes AI agent ergonomics over human UI
3. **Work = issues with dependencies**: No special workflow types — dependencies ARE the workflow
4. **Zero-conflict multi-agent**: Hash-based IDs and content hashing make merge conflicts extremely rare
5. **Progressive adoption**: Start with `bd init --stealth` for personal use, scale to cross-org federation
6. **Dogfooding**: Beads uses Beads for its own issue tracking

---

## Architecture

### Three-Layer Design

Beads has a three-layer architecture:

```
┌─────────────────────────────────┐
│         CLI (bd) / MCP          │  ← User & Agent Interface
├─────────────────────────────────┤
│        Daemon (per-project)     │  ← Background sync, debounce, git hooks
├─────────────────────────────────┤
│   SQLite (fast) + JSONL (git)   │  ← Dual storage
└─────────────────────────────────┘
```

**Layer 1 — CLI (`bd`)**: Command-line interface for humans and agents. Every command supports `--json` for programmatic consumption. A Claude Code plugin and MCP server provide additional integration surfaces.

**Layer 2 — Daemon**: A per-project background process that:
- Debounces rapid changes (batches SQLite → JSONL exports)
- Manages git commit/push cycles
- Handles import after git pull/merge
- Runs at ~1% CPU idle

**Layer 3 — Dual Storage**:
- **SQLite**: Fast local queries, WAL mode, full-text search, materialized caches
- **JSONL**: Git-tracked portable format, one JSON object per line, designed for clean diffs/merges

The flow: CLI writes to SQLite → daemon debounces → exports to JSONL → git commits → pushes to remote.

### Directory Structure

```
.beads/
  config.yaml         # Project configuration
  beads.jsonl         # Git-tracked issue data
  beads.db            # SQLite database (gitignored)
  BD_GUIDE.md         # Auto-generated agent guide
  formulas/           # Workflow templates (TOML)
```

### Source Code Organization

```
beads/
  cmd/bd/              # CLI commands (main.go, create.go, ready.go, etc.)
  internal/
    types/types.go     # Core data types
    storage/sqlite/    # SQLite backend with migrations
    daemon/            # Background sync daemon
    hooks/             # Git hook management
    molecules/         # Molecule/workflow system
    ui/                # CLI output formatting
  integrations/
    beads-mcp/         # Python MCP server for Claude Desktop
  claude-plugin/       # Claude Code plugin
  docs/                # Documentation
```

---

## Data Model

### The Issue (Bead) Model

The core data structure is the `Issue` type (defined in `internal/types/types.go`). It is an extensive struct with fields organized into logical groups:

**Core Identification:**
- `ID` (string): Unique hash-based identifier (e.g., `bd-a1b2`)
- `ContentHash` (string): SHA256 of canonical content for change detection

**Issue Content:**
- `Title`, `Description`, `Design`, `AcceptanceCriteria`, `Notes`, `SpecID`

**Status and Workflow:**
- `Status`: Current lifecycle state (`open`, `in_progress`, `blocked`, `deferred`, `closed`, `hooked`, `tombstone`)
- `Priority` (int): 0-4 scale
- `IssueType`: Classification (task, bug, epic, etc.)

**Assignment:**
- `Assignee`: Who is working on this
- `Owner`: Human owner for attribution
- `EstimatedMinutes`: Time estimate

**Timestamps:**
- `CreatedAt`, `UpdatedAt`, `ClosedAt`, `DueAt`, `DeferUntil`
- `CreatedBy`, `CloseReason`, `ClosedBySession`

**External Integration:**
- `ExternalRef`: Link to external systems (e.g., `gh-9`, `jira-ABC`)
- `SourceSystem`: Which system created this issue (for federation)

**Custom Metadata:**
- `Metadata` (json.RawMessage): Arbitrary JSON for extension points

**Compaction Metadata:**
- `CompactionLevel`, `CompactedAt`, `CompactedAtCommit`, `OriginalSize`

**Tombstone Fields (soft-delete):**
- `DeletedAt`, `DeletedBy`, `DeleteReason`, `OriginalType`

**Agent Identity Fields (agent-as-bead):**
- `HookBead`: Current work on agent's hook
- `RoleBead`: Role definition bead
- `AgentState`: `idle`, `spawning`, `running`, `working`, `stuck`, `done`, `stopped`, `dead`
- `LastActivity`: For timeout detection
- `RoleType`, `Rig`: Agent classification

**Molecule/Workflow Fields:**
- `MolType`: `swarm`, `patrol`, or `work`
- `WorkType`: `mutex` (exclusive) or `open_competition`
- `WispType`: Classification for TTL-based compaction

**Gate Fields (async coordination):**
- `AwaitType`, `AwaitID`, `Timeout`, `Waiters`

**Slot Fields (exclusive access):**
- `Holder`: Who currently holds the slot

**Event Fields:**
- `EventKind`, `Actor`, `Target`, `Payload`

### Content Hashing

Each issue has a deterministic content hash computed from all substantive fields (excluding ID, timestamps, and compaction metadata). Used for:

1. **Change detection during import**: Same ID + same hash = skip; different hash = update
2. **Deduplication**: Identical content detected and merged
3. **Distributed consistency**: All clones converge to the same state

### Issue Lifecycle

```
open --> in_progress --> closed
  |                       ^
  |                       |
  +-------(reopen)--------+
  |
  +--> blocked (by dependencies)
  |
  +--> deferred (deliberately postponed)
  |
  +--> hooked (assigned to an agent)
```

---

## Storage Backends

### SQLite (Primary Query Engine)

- Schema managed with migrations in `internal/storage/sqlite/`
- Tables: `issues`, `dependencies`, `labels`, `comments`, `events`, `blocked_issues_cache`, `dirty_issues`, `export_hashes`, `config`, `counters`
- WAL mode for concurrent reads
- Full-text search support
- Foreign key constraints with CASCADE deletes

### JSONL (Git-Tracked Portable Format)

- One JSON object per line
- Each entity (issue, dependency, label, comment) is a separate line
- Designed for git-friendly diffs and merges
- Serves as the portable source of truth

### Dolt (Experimental)

- Database-native version control
- Enables federation between organizations
- Supports push/pull between Dolt remotes (DoltHub, S3, GCS, local)

### Extensibility

Applications can extend the SQLite database with custom tables:
- Use `UnderlyingDB()` to access the same database connection
- Create namespaced tables (e.g., `myapp_executions`)
- Join with beads tables via foreign keys
- Use batch `CreateIssues()` API for bulk operations (5-30x faster than individual creates)

---

## CLI and Agent Interface

### Core Commands

```bash
bd init                          # Initialize beads in a repo
bd create "Title" -p 1 -t task   # Create an issue
bd ready --json                  # List unblocked work
bd update bd-42 --status in_progress --claim  # Claim work
bd close bd-42 --reason "Done"   # Complete work
bd sync                          # Force immediate flush/commit/push
bd blocked                       # Show blocked issues
bd dep add <B> <A> --type blocks # Add dependency
bd compact                       # Run compaction
bd stats                         # Show statistics
```

Every command supports `--json` for programmatic consumption by AI agents.

### Agent Workflow Pattern

```bash
# 1. Check what's ready
bd ready --json

# 2. Claim a task
bd update bd-42 --status in_progress --claim --json

# 3. Do the work

# 4. If new work discovered:
bd create "Found bug in auth" -p 1 --deps discovered-from:bd-42 --json

# 5. Complete
bd close bd-42 --reason "Implemented auth validation" --json

# 6. Sync at end of session
bd sync
```

---

## Multi-Project Management

### Per-Project Isolation

Each git repository that runs `bd init` gets its own `.beads/` directory with:
- An isolated SQLite database
- Its own JSONL file tracked in git
- Its own daemon process
- Its own configuration

### Multi-Repo Hydration

Aggregate issues from multiple repositories into a unified view:

```bash
bd config set repos.primary "."
bd config set repos.additional "~/repo1,~/repo2,~/repo3"
```

How it works:
1. Beads reads JSONL from all configured repos
2. Imports into a unified SQLite database
3. Maintains `source_repo` field for provenance tracking
4. Exports route issues back to the correct JSONL files

`bd ready` then shows unblocked work across all configured projects.

### Auto-Routing

The routing system automatically detects user role and routes `bd create` accordingly:

**Role detection priority:**
1. Explicit git config: `git config beads.role maintainer|contributor`
2. Git remote URL inspection: SSH = maintainer, HTTPS = contributor
3. Fallback: contributor

**Maintainer**: Issues created in the current repo, visible to all.
**Contributor**: Issues routed to a separate planning repo (e.g., `~/.beads-planning`), keeping experimental planning private.

### Stealth Mode

`bd init --stealth` enables Beads locally without committing files to the shared repository. Perfect for personal task tracking on shared projects without team buy-in.

### Sync Branch Mode

For projects with protected main branches:

```bash
bd init --branch beads-metadata
```

Commits `.beads/` changes to a separate branch via git worktree, avoiding pollution of the main branch.

### Cross-Project Dependencies

```bash
cd ~/projects/myapp-planning
bd create "Design auth" -p 1 -t epic

cd ~/projects/myapp-implementation
bd dep add impl-42 plan-10 --type blocks
```

The `discovered-from` dependency type automatically inherits the parent's `source_repo`.

---

## AI Agent Integration

### Design Principles

- JSON output on every command (~1-2k tokens vs 10-50k for MCP schemas)
- Dependency-aware ready work computation
- Hash-based IDs prevent multi-agent conflicts
- `discovered-from` links maintain context chains
- Compaction manages context window limits
- Agent-as-bead enables self-monitoring

### Integration Surfaces

**CLI (recommended for agents with shell access)**: Lowest token overhead, full feature set.

**MCP Server** (for Claude Desktop and similar): Python MCP server at `integrations/beads-mcp/`. Functions: `ready`, `show`, `update`, `create`, `dep`, `close`, `blocked`, `stats`.

**Claude Code Plugin**: Custom slash commands (`/create`, `/list`, `/ready`, `/close`, `/dep`, `/audit`, `/blocked`) and a task agent definition.

### Agent-as-Bead

Agents themselves can be represented as beads with identity fields:

```go
AgentState: idle | spawning | running | working | stuck | done | stopped | dead
HookBead:   // Current work assignment
RoleBead:   // Role definition
LastActivity: // For timeout detection
```

This enables:
- Agent state tracking and monitoring
- Work assignment via "hooks"
- Timeout detection for dead agents
- Multi-agent coordination via slot primitives

### The "Hook" Pattern

Each agent has a "hook" — a single bead representing their current work assignment. The `hooked` status means work is attached to an agent. This enables tracking, handoffs, and stuck-agent detection.

### "Landing the Plane" Protocol

Key operational protocol for agents ending work sessions:

1. File beads issues for remaining work
2. Run quality gates (tests, linting)
3. Update issue statuses
4. **PUSH TO REMOTE** (mandatory — unpushed work breaks multi-agent coordination)
5. Clean up git state
6. Provide handoff context

> "The plane has NOT landed until `git push` completes successfully."

---

## Dependency System and Ready Work

### Dependency Types

| Type | Effect on Readiness | Purpose |
|------|-------------------|---------|
| `blocks` | Blocking | B cannot start until A completes |
| `parent-child` | Blocking | Parent-child grouping |
| `conditional-blocks` | Blocking | B runs only if A fails |
| `waits-for` | Blocking | Async coordination point |
| `related` | Informational only | Cross-reference |
| `discovered-from` | Informational only | Context chain |

### Ready Work Computation

`bd ready` computes which issues have no open blockers:
- Uses materialized `blocked_issues_cache` for performance
- Supports filtering by assignee, labels, type, priority, molecule, sort policy
- Deferred issues with future `defer_until` excluded by default

### Sort Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `hybrid` (default) | Recent by priority, older by age | General use |
| `priority` | Always priority first, then age | Autonomous execution |
| `oldest` | Oldest first | Backlog clearing |

### Atomic Operations

- `bd update --claim`: Atomically sets assignee + in_progress
- Content hashing: Import merge uses "last writer wins" but detects all changes
- Transaction-safe cache: Blocked issues cache updates atomically
- Debounced export: Batches rapid changes into single JSONL exports

---

## Molecules: Structured Workflows

Molecules are Beads' system for structuring multi-step workflows.

> "Work = issues with dependencies. That's it. No special types needed."

A molecule is an **epic (parent issue) with children** where dependencies control execution order.

### Phase Metaphor (Chemistry)

| Phase | Name | Storage | Synced to Git | Purpose |
|-------|------|---------|---------------|---------|
| **Solid** | Proto | `.beads/` | Yes | Frozen template (reusable pattern) |
| **Liquid** | Mol | `.beads/` | Yes | Active persistent work |
| **Vapor** | Wisp | `.beads/` | No | Ephemeral operations (local only) |

### Commands

```bash
bd mol pour <proto>            # Proto → Mol (persistent instance)
bd mol wisp <proto>            # Proto → Wisp (ephemeral instance)
bd mol squash <id>             # Mol/Wisp → Digest (permanent record)
bd mol burn <id>               # Wisp → nothing (discard)
bd mol bond A B                # Connect two work graphs
```

### Bonding

Bonding creates dependencies between work graphs:

```bash
bd mol bond A B                     # B depends on A (sequential)
bd mol bond A B --type parallel     # Organizational link, no blocking
bd mol bond A B --type conditional  # B runs only if A fails
```

### Wisps: Ephemeral Work

Wisps are intentionally local-only:
- Exist only in the spawning agent's SQLite database
- Never exported to JSONL
- Hard-deleted when squashed (no tombstones)
- Enable fast local iteration without sync overhead

### Formulas: Workflow Templates

TOML files that define reusable workflows:

```toml
formula = "beads-release"
type = "workflow"
phase = "vapor"

[vars.version]
description = "The semantic version to release"
required = true

[[steps]]
id = "preflight-git"
title = "Preflight: Check git status"

[[steps]]
id = "bump-version"
title = "Bump version in version.go"
needs = ["preflight-git"]   # Dependency!
```

### Gate Steps: Async Coordination

Gates block until an external condition is met:

```toml
[steps.gate]
type = "gh:run"        # Wait for GitHub Actions
id = "release.yml"     # Specific workflow
timeout = "30m"
```

---

## Compaction and Memory Decay

Beads implements "semantic memory decay" — progressive summarization of old closed tasks to manage context window limits.

### Tiers

- **Tier 1**: Semantic compression (30 days closed, 70% reduction)
- **Tier 2**: Ultra compression (90 days closed, 95% reduction)

### Modes

- **Prune**: Remove expired tombstones from JSONL
- **Analyze**: Export candidates for agent review
- **Apply**: Accept agent-provided summary
- **Auto**: AI-powered compaction using Anthropic API

### Tombstone Lifecycle

1. Issue deleted → becomes tombstone with `deleted_at` timestamp
2. Tombstone propagates to all clones via JSONL
3. After TTL expires (default 30 days), tombstone can be pruned
4. Clock skew grace period (1 hour) prevents premature pruning

### Wisp Compaction (TTL-based)

| Category | TTL | Examples |
|----------|-----|---------|
| High-churn | 6 hours | Heartbeats, pings |
| Operational | 24 hours | Patrol cycle reports |
| Significant | 7 days | Recovery actions, errors |

---

## Federation

Federation enables peer-to-peer synchronization between independent beads databases.

### Architecture

- **Peer-to-peer**: No central server; each "town" is autonomous
- **Database-native versioning**: Built on Dolt's distributed version control
- **Configurable data sovereignty**: Four tiers from "no restrictions" to "anonymous"

### Supported Endpoints

| Format | Example |
|--------|---------|
| DoltHub | `dolthub://org/repo` |
| Google Cloud | `gs://bucket/path` |
| Amazon S3 | `s3://bucket/path` |
| Local | `file:///path/to/backup` |

### HOP: History of Provenance

Tracking framework for entity provenance and trust chains:
- **EntityRef**: URI format `entity://hop/<platform>/<org>/<id>`
- **Validations**: Records of who approved/rejected work with quality scores
- **QualityScore**: Aggregate quality metric (0.0-1.0)
- **Crystallizes**: Whether work compounds (code) vs evaporates (ops)

---

## Key Features and Differentiators

### 1. Git-Native Distribution
Issues live in git — offline work is first-class, no vendor lock-in, full version history.

### 2. Agent-Optimized Design
JSON output everywhere, dependency-aware ready computation, hash-based IDs, compaction for context windows.

### 3. Zero-Conflict Multi-Agent
Hash-based IDs + content hashing + JSONL format = merge conflicts extremely rare.

### 4. Molecular Workflow Engine
Workflow capabilities comparable to dedicated engines (Temporal, Airflow) using the same primitives as issue tracking.

### 5. Semantic Memory Decay
Progressive summarization of old issues — recent details are vivid, old details fade to summaries.

### 6. Multi-Repo Aggregation
Unified view across multiple repositories without centralizing data.

### 7. Extensible Database
Shared SQLite database enables custom tables alongside beads tables.

### 8. Agent Self-Awareness
Agent-as-bead: state tracking, work assignment, timeout detection, swarm coordination.

### 9. Progressive Adoption
`--stealth` for personal use → `--contributor` for OSS → full team adoption → federation.

### 10. Dogfooding
Beads uses Beads for all its own issue tracking, including a release workflow formula.

---

## Appendix: Key Files

### Source Code
| File | Purpose |
|------|---------|
| `beads.go` | Public API package |
| `internal/types/types.go` | Core data types |
| `cmd/bd/main.go` | CLI entry point |
| `cmd/bd/create.go` | Issue creation |
| `cmd/bd/ready.go` | Ready work computation |
| `cmd/bd/compact.go` | Compaction engine |
| `cmd/bd/sync.go` | Sync command |

### Documentation
| File | Purpose |
|------|---------|
| `README.md` | Project overview |
| `AGENT_INSTRUCTIONS.md` | Detailed agent instructions |
| `docs/ARCHITECTURE.md` | Three-layer architecture |
| `docs/MOLECULES.md` | Workflow execution model |
| `docs/MULTI_REPO_AGENTS.md` | Multi-repo patterns |
| `docs/ROUTING.md` | Auto-routing |
| `docs/INTERNALS.md` | FlushManager, blocked cache |
| `docs/EXTENDING.md` | Custom table extension |
| `FEDERATION-SETUP.md` | Cross-org federation |

### Integration
| File | Purpose |
|------|---------|
| `integrations/beads-mcp/` | Python MCP server |
| `claude-plugin/` | Claude Code plugin |
| `claude-plugin/agents/task-agent.md` | Autonomous task agent |
| `.beads/formulas/beads-release.formula.toml` | Release workflow template |
