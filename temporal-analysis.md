# Temporal: Durable Execution for AI Agent Orchestration

## Executive Summary

**Temporal** (temporal.io) is a durable execution platform originally developed at Uber as Cadence and open-sourced independently in 2020. It is not a task tracker. It is not an issue management system. It is a workflow engine that guarantees code runs to completion -- even through crashes, network failures, and infrastructure outages -- by recording every step as an event in a persistent history.

This distinction matters. Claude Code Tasks and Beads are systems for tracking *what work exists*. Temporal is a system for ensuring *work actually completes*. It sits above both in the architectural stack: an orchestration layer that can dispatch work to any agent, recover from any failure, and coordinate arbitrarily complex multi-step processes through deterministic replay.

Temporal is not theoretical. OpenAI's Codex runs its agent execution on Temporal. Replit Agent 3 uses Temporal for its agentic coding workflows. The OpenAI Agents SDK integrates with Temporal for durable agent execution. These are production systems serving millions of users, built on the same core primitive: workflows that survive crashes.

The tradeoffs are real. Temporal requires infrastructure (a server cluster or a cloud subscription). It has a steep learning curve rooted in determinism constraints and event sourcing. It is overkill for a solo developer running short sessions on a single project. But for anyone who needs crash recovery, multi-agent coordination, vendor independence, or cross-machine workflow execution, Temporal is the concrete implementation of the orchestration principle described in `orchestration-principle.md`.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Core Concepts](#core-concepts)
3. [Architecture](#architecture)
4. [Crash Recovery: The Killer Feature](#crash-recovery-the-killer-feature)
5. [Mapping to the Orchestration Principle](#mapping-to-the-orchestration-principle)
6. [AI Agent Orchestration Patterns](#ai-agent-orchestration-patterns)
7. [Cross-Session Continuity and Multi-Project](#cross-session-continuity-and-multi-project)
8. [Dependency and Coordination](#dependency-and-coordination)
9. [Setup Cost and Tradeoffs](#setup-cost-and-tradeoffs)
10. [Comparison Table](#comparison-table)
11. [When to Use Temporal](#when-to-use-temporal)

---

## Core Concepts

Temporal's programming model has five fundamental building blocks.

### Workflows

A **Workflow** is a deterministic function that defines the overall process. It can last seconds or years. It must not perform I/O, access the network, or generate random numbers directly -- all of these must go through Activities. This constraint exists because Temporal replays Workflow code during recovery: if the code is deterministic, replaying it with the same inputs produces the same execution path, and Temporal can skip past already-completed steps.

```python
# Pseudocode -- a Temporal Workflow
@workflow.defn
class CodeReviewWorkflow:
    @workflow.run
    async def run(self, pr_url: str) -> ReviewResult:
        # Step 1: Fetch PR diff (Activity -- has side effects)
        diff = await workflow.execute_activity(
            fetch_pr_diff, pr_url, start_to_close_timeout=timedelta(minutes=5)
        )
        # Step 2: Run AI review (Activity -- calls external API)
        review = await workflow.execute_activity(
            run_ai_review, diff, start_to_close_timeout=timedelta(minutes=10)
        )
        # Step 3: Post review comment (Activity -- writes to GitHub)
        await workflow.execute_activity(
            post_review_comment, pr_url, review,
            start_to_close_timeout=timedelta(minutes=2)
        )
        return review
```

If the Worker crashes after Step 1 completes but during Step 2, a new Worker replays the Workflow, sees Step 1 already has a result in the event history, skips it, and retries Step 2.

### Activities

An **Activity** is a non-deterministic function -- the place where side effects happen. Network calls, file I/O, database writes, API requests, invoking an AI agent. Activities can be retried automatically on failure with configurable retry policies (initial interval, backoff coefficient, max attempts, non-retryable error types).

Activities are the boundary between the deterministic orchestration logic and the messy real world.

### Workers

A **Worker** is a process you run on your own infrastructure. It polls the Temporal Server for work, executes Workflows and Activities, and reports results back. Workers are stateless -- any Worker can pick up any task from a Task Queue. You can scale horizontally by running more Workers.

Workers are where your code actually runs. The Temporal Server never executes your Workflow or Activity code -- it only stores the event history and dispatches tasks.

### Task Queues

A **Task Queue** is a named queue that connects Workflow/Activity dispatches to Workers. Workers subscribe to specific Task Queues. This enables routing: you can direct AI-heavy work to GPU-equipped Workers and file-system operations to Workers with local disk access.

Task Queues are lightweight (created on demand, no pre-provisioning) and support both Workflow Task dispatching and Activity Task dispatching.

### Event History

The **Event History** is Temporal's persistence mechanism. Every Workflow execution maintains an append-only log of events:

- `WorkflowExecutionStarted`
- `ActivityTaskScheduled`
- `ActivityTaskStarted`
- `ActivityTaskCompleted` (with the Activity's return value)
- `WorkflowExecutionCompleted`

This history is the source of truth. When a Worker crashes and another Worker picks up the Workflow, it replays the history: Workflow code runs from the beginning, but each Activity call checks the history first. If the Activity already completed, its recorded result is returned immediately without re-executing. This is how Temporal achieves crash recovery without data loss.

The history is stored in the Temporal Server's backing database (Cassandra, PostgreSQL, MySQL, or SQLite for development).

---

## Architecture

### Temporal Server

The Temporal Server is a cluster of four internal services:

```
┌─────────────────────────────────────────────────────┐
│                   Temporal Server                     │
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────┐│
│  │ Frontend │  │ History  │  │ Matching │  │Worker ││
│  │ Service  │  │ Service  │  │ Service  │  │Service││
│  └──────────┘  └──────────┘  └──────────┘  └───────┘│
│                       │                               │
│              ┌────────┴────────┐                      │
│              │   Database      │                      │
│              │ (Postgres/MySQL/│                      │
│              │  Cassandra)     │                      │
│              └─────────────────┘                      │
└─────────────────────────────────────────────────────┘
         ▲                           ▲
         │ gRPC                      │ gRPC
    ┌────┴────┐                 ┌────┴────┐
    │ Worker  │                 │ Worker  │
    │(your    │                 │(your    │
    │ code)   │                 │ code)   │
    └─────────┘                 └─────────┘
```

**Frontend Service**: The API gateway. Handles all client requests (start Workflow, signal Workflow, query Workflow). Rate limiting and authorization happen here.

**History Service**: Maintains the event history for every Workflow execution. Handles the core state machine logic -- recording events, managing timers, and serving replay data to Workers.

**Matching Service**: Manages Task Queues. When a Workflow schedules an Activity, the Matching Service puts it into the appropriate Task Queue. When a Worker polls for work, the Matching Service dispatches the task.

**Worker Service (internal)**: Runs Temporal's own internal Workflows -- system operations like archiving old Workflow histories. Not to be confused with your application Workers.

### Workers (Your Infrastructure)

Workers are long-running processes you deploy and manage. They:

1. Connect to the Temporal Server via gRPC
2. Subscribe to one or more Task Queues
3. Poll for Workflow Tasks and Activity Tasks
4. Execute your code locally
5. Report results back to the server

Workers can run anywhere: local machine, Docker container, Kubernetes pod, cloud VM. They need network access to the Temporal Server and to whatever external services your Activities call.

### Event Sourcing Model

Temporal uses event sourcing -- the event history is the canonical state. The current state of a Workflow is not stored directly; it is derived by replaying the event history through the Workflow code. This is why Workflow code must be deterministic: non-deterministic code would produce a different execution path on replay, causing a "non-determinism error."

This model provides:
- **Complete auditability**: Every step is recorded with timestamps and payloads
- **Time travel**: You can reconstruct the state of any Workflow at any point in its execution
- **Crash recovery**: Replay from history reproduces the exact pre-crash state

### Deployment Options

**Development**: `temporal server start-dev` runs a single-process server with SQLite. Zero configuration, starts in seconds. Suitable for local development and testing.

**Docker Compose**: Official `docker-compose.yml` runs the server with PostgreSQL. Suitable for small teams and staging environments.

**Kubernetes (Helm)**: Official Helm chart for production deployments. Supports horizontal scaling of each service independently.

**Temporal Cloud**: Managed service from Temporal Technologies. Starts at ~$100/month for 1,000 actions/second. No infrastructure to manage. SLA guarantees.

---

## Crash Recovery: The Killer Feature

Crash recovery is Temporal's defining capability and the primary reason to consider it for AI agent orchestration. This section walks through the recovery mechanism in detail.

### The Scenario

An AI agent workflow has three steps:

1. **Activity A**: Fetch the issue description from GitHub
2. **Activity B**: Send the description to an AI model for code generation
3. **Activity C**: Create a pull request with the generated code

The Worker crashes after Activity A completes, during Activity B.

### What Happens in Temporal

```
Timeline:
─────────────────────────────────────────────────────────
1. Workflow starts on Worker-1
2. Activity A scheduled → dispatched to Worker-1
3. Activity A completes → result recorded in history:
     Event: ActivityTaskCompleted { result: "<diff data>" }
4. Activity B scheduled → dispatched to Worker-1
5. Activity B starts on Worker-1
6. *** Worker-1 crashes ***
7. Activity B times out (start-to-close timeout expires)
8. Temporal detects Worker-1 is gone (heartbeat timeout)
9. Worker-2 picks up the Workflow Task from the queue
10. Worker-2 replays the Workflow:
     - Calls execute_activity(fetch_github_issue, ...)
     - History shows ActivityTaskCompleted → returns cached result
     - Calls execute_activity(run_ai_review, ...)
     - No history entry → schedules Activity B for real execution
11. Activity B runs on Worker-2
12. Activity B completes → result recorded in history
13. Activity C scheduled, executed, completed
14. Workflow completes successfully
─────────────────────────────────────────────────────────
```

Key observations:

- **Activity A is NOT re-executed**. Its result was recorded in the event history. Replay returns the cached result. No duplicate API calls, no duplicate side effects.
- **Activity B IS re-executed** on the new Worker. The Temporal Server detected the timeout and made the task available for another Worker.
- **No human intervention required**. The recovery is fully automatic.
- **The Workflow code did not need crash-handling logic**. The same code handles both the happy path and the crash-recovery path.

### What Happens Without Temporal

**Claude Code**: The session crashes. Tasks in `in_progress` status remain stuck permanently. The `qn()` cleanup function only runs during graceful shutdown. There is no mechanism to detect the crash, identify stuck tasks, or re-dispatch work. The user must manually create a new session and re-do the work from scratch (or from `--resume`, which replays the conversation but does not retry failed operations).

**Beads**: Git-persisted state survives the crash -- the issue remains in `in_progress` status and is discoverable via `bd ready` after the agent restarts. However, Beads has no automatic retry. The agent must manually pick up where it left off, and there is no record of which steps within the task completed successfully. Partial work may need to be re-done.

### Recovery Comparison

| Aspect | Temporal | Claude Code | Beads |
|--------|----------|-------------|-------|
| Crash detection | Automatic (heartbeat timeout) | None | Manual (agent timeout detection) |
| State after crash | Preserved in event history | Stuck in `in_progress`, undiscoverable | Preserved in git, discoverable |
| Automatic retry | Yes (re-dispatch to available Worker) | No | No |
| Partial work preserved | Yes (completed Activities cached) | No | No (at issue level, not step level) |
| Recovery requires human | No | Yes | Partially (agent must restart) |
| Duplicate side effects | Prevented (replay uses cached results) | N/A (no retry) | Possible (no step-level tracking) |

---

## Mapping to the Orchestration Principle

`orchestration-principle.md` argues that orchestration belongs in a deterministic system, not in agent internals. Temporal is a concrete, production-proven implementation of this principle.

### Principle: "Orchestration belongs in your own deterministic system"

**Temporal implementation**: Workflows are deterministic functions. The Workflow code decides what to do next, in what order, with what retry policy, and under what conditions. This logic runs as traditional software, not as an LLM prompt. It does not forget instructions. It does not skip steps. It does not hallucinate workflow states.

### Principle: "Agents are workers"

**Temporal implementation**: AI agents are Activities. A Workflow schedules an Activity that invokes `claude --print "Fix the authentication bug in auth.py"`. The agent receives a scoped task, does the thinking-heavy work, and returns a result. The Workflow decides what happens next based on that result.

```python
@activity.defn
async def run_claude_agent(task_description: str) -> str:
    """Invoke Claude Code as an Activity."""
    result = subprocess.run(
        ["claude", "--print", task_description],
        capture_output=True, text=True, timeout=600
    )
    return result.stdout
```

### Principle: "Agent-internal task systems are convenience layers"

**Temporal implementation**: When Claude Code runs as an Activity, it can use TaskCreate/TaskList internally to organize its own work during that invocation. The Workflow does not depend on Claude's internal task state. If Claude crashes, Temporal retries the Activity with a fresh agent invocation. Claude's internal tasks were session conveniences; the real state is in Temporal's event history.

### Principle: "The orchestrator is vendor-independent"

**Temporal implementation**: Switching from Claude to Codex means changing the Activity implementation:

```python
# Before
@activity.defn
async def run_agent(task: str) -> str:
    result = subprocess.run(["claude", "--print", task], ...)
    return result.stdout

# After
@activity.defn
async def run_agent(task: str) -> str:
    result = subprocess.run(["codex", "--quiet", task], ...)
    return result.stdout
```

The Workflow code -- the orchestration logic -- does not change. The same Workflow can even use different agents for different Activities:

```python
# Use Claude for complex reasoning, a cheaper model for boilerplate
analysis = await workflow.execute_activity(run_claude, complex_task)
boilerplate = await workflow.execute_activity(run_codex, simple_task)
```

### The Orchestration Principle, Materialized

| Principle | Abstract (orchestration-principle.md) | Concrete (Temporal) |
|-----------|--------------------------------------|---------------------|
| Deterministic orchestrator | "Your own system" | Temporal Workflow |
| Agent workers | "Receive tasks, return results" | Activities that invoke CLI agents |
| Crash recovery | "Re-dispatch if agent crashes" | Automatic replay + retry |
| Vendor independence | "Changing a command" | Changing an Activity implementation |
| State ownership | "Your repo, your database" | Temporal's event history |
| Agent tasks as convenience | "Within-session helper" | TaskCreate inside an Activity |

---

## AI Agent Orchestration Patterns

### Pattern 1: CLI Agent as Activity

The simplest pattern. Each Activity invokes an agent CLI tool as a subprocess.

```python
@activity.defn
async def code_with_claude(prompt: str, working_dir: str) -> str:
    result = subprocess.run(
        ["claude", "--print", "--output-format", "text", prompt],
        cwd=working_dir, capture_output=True, text=True, timeout=900
    )
    if result.returncode != 0:
        raise ApplicationError(f"Agent failed: {result.stderr}")
    return result.stdout

@workflow.defn
class FixBugWorkflow:
    @workflow.run
    async def run(self, bug_description: str, repo_path: str):
        # Agent generates fix
        fix = await workflow.execute_activity(
            code_with_claude,
            f"Fix this bug: {bug_description}",
            repo_path,
            start_to_close_timeout=timedelta(minutes=15),
            retry_policy=RetryPolicy(maximum_attempts=3)
        )
        # Deterministic step: run tests
        await workflow.execute_activity(
            run_tests, repo_path,
            start_to_close_timeout=timedelta(minutes=10)
        )
        # Deterministic step: create PR
        await workflow.execute_activity(
            create_pull_request, repo_path, bug_description,
            start_to_close_timeout=timedelta(minutes=5)
        )
```

The Workflow is deterministic: fix, test, PR. The agent does the creative work. The test runner and PR creation are deterministic Activities. If any step fails, Temporal retries it (up to the configured limit).

### Pattern 2: Production Examples

**OpenAI Codex**: Runs agent execution loops on Temporal. Each coding task is a Workflow. Agent interactions (reading files, writing code, running commands) are Activities. Temporal handles the orchestration, retry logic, and crash recovery that makes Codex reliable at scale.

**Replit Agent 3**: Uses Temporal for its agentic coding workflows. The agent's multi-step coding process (understand requirements, plan changes, write code, run tests, iterate) is modeled as a Temporal Workflow with Activities for each step.

**OpenAI Agents SDK**: Provides a Temporal integration module. Agent execution can be wrapped in Temporal Activities, and multi-agent workflows can be expressed as Temporal Workflows. This is the pattern for building durable agent pipelines with the OpenAI ecosystem.

### Pattern 3: Multi-Agent Routing

Different agents have different strengths. A Workflow can route tasks to the appropriate agent:

```python
@workflow.defn
class MultiAgentWorkflow:
    @workflow.run
    async def run(self, task: Task):
        if task.requires_deep_reasoning:
            result = await workflow.execute_activity(
                run_claude_opus, task.description,
                task_queue="gpu-workers"
            )
        elif task.is_boilerplate:
            result = await workflow.execute_activity(
                run_codex, task.description,
                task_queue="standard-workers"
            )
        else:
            result = await workflow.execute_activity(
                run_claude_haiku, task.description,
                task_queue="fast-workers"
            )
        return result
```

Task Queues enable physical routing -- GPU-equipped Workers handle expensive model calls, while standard Workers handle simpler tasks.

### Pattern 4: Human-in-the-Loop via Signals

Temporal **Signals** allow external events to influence a running Workflow. This enables human approval gates:

```python
@workflow.defn
class ReviewWorkflow:
    def __init__(self):
        self.approved = False

    @workflow.signal
    async def approve(self):
        self.approved = True

    @workflow.run
    async def run(self, pr_url: str):
        # Agent generates code
        code = await workflow.execute_activity(generate_code, pr_url)
        # Wait for human approval (could be minutes, hours, or days)
        await workflow.wait_condition(lambda: self.approved)
        # Deploy after approval
        await workflow.execute_activity(deploy, code)
```

The Workflow pauses at `wait_condition` indefinitely. When a human sends an `approve` Signal (via CLI, API, or UI), the Workflow resumes. This is durable -- the Workflow survives server restarts while waiting.

### Pattern 5: Iterative Agent Loop

Many agent tasks require iteration: write code, test, fix, test again. Temporal handles this naturally:

```python
@workflow.defn
class IterativeFixWorkflow:
    @workflow.run
    async def run(self, task: str, repo_path: str, max_iterations: int = 5):
        for i in range(max_iterations):
            # Agent attempts the fix
            await workflow.execute_activity(
                code_with_claude,
                f"{'Fix the failing tests' if i > 0 else task}",
                repo_path
            )
            # Run tests
            test_result = await workflow.execute_activity(
                run_tests, repo_path
            )
            if test_result.passed:
                await workflow.execute_activity(create_pr, repo_path, task)
                return "Success"
        return "Failed after max iterations"
```

If the Worker crashes during iteration 3, Temporal replays iterations 1-2 from history (using cached results) and retries iteration 3. No work is lost.

---

## Cross-Session Continuity and Multi-Project

### No "Sessions" in Temporal

Temporal does not have a concept of sessions that expire. A Workflow Execution starts and runs until it completes, fails, times out, or is cancelled. This can take milliseconds or years. There is no "session ended, state lost" failure mode.

When you start a Workflow, you assign it a **Workflow ID** -- a user-chosen string like `fix-auth-bug-2024-02-08` or `project-backend/issue-42`. This ID is stable and queryable. You can:

- **Query** a running Workflow for its current state
- **Signal** a running Workflow with new information
- **Cancel** a running Workflow
- **Terminate** a running Workflow (hard kill)
- **Describe** a Workflow to see its status, start time, and metadata

This eliminates the cross-session problem entirely. There are no random UUIDs to track, no `--resume` flags, no `CLAUDE_CODE_TASK_LIST_ID` env vars. You choose the Workflow ID, and it persists until the Workflow completes.

### Namespaces for Project Isolation

Temporal **Namespaces** provide logical isolation between groups of Workflows. Each Namespace has its own:

- Workflow execution history
- Task Queues
- Search Attributes (custom indexed metadata)
- Retention period (how long completed Workflow histories are kept)
- Access control (on Temporal Cloud)

A natural mapping for multi-project use:

```
Namespace: "project-backend"
  Workflow: "fix-auth-bug"
  Workflow: "add-rate-limiting"
  Workflow: "refactor-db-layer"

Namespace: "project-frontend"
  Workflow: "redesign-dashboard"
  Workflow: "fix-mobile-layout"

Namespace: "project-infra"
  Workflow: "upgrade-kubernetes"
```

Namespaces are lightweight and can be created programmatically. They provide the multi-project isolation that Claude Code lacks entirely.

### Nexus for Cross-Namespace Coordination

**Temporal Nexus** (GA in 2025) enables Workflows in one Namespace to call operations in another Namespace. This is the cross-project dependency mechanism:

```python
# In "project-frontend" namespace
@workflow.defn
class FrontendFeatureWorkflow:
    @workflow.run
    async def run(self):
        # Call an operation in "project-backend" namespace
        api_endpoint = await workflow.execute_nexus_operation(
            BackendService.deploy_api,
            DeployRequest(feature="user-profiles"),
        )
        # Now build the frontend against the new API
        await workflow.execute_activity(
            build_frontend, api_endpoint
        )
```

Nexus operations are durable -- they participate in the event history and survive crashes just like Activities.

### Signals, Queries, and Updates

Three mechanisms for interacting with running Workflows from outside:

**Signals**: Fire-and-forget messages delivered to a Workflow. Used for events: "a human approved this PR", "CI passed", "new issue filed." The Workflow decides how to react.

**Queries**: Synchronous reads of Workflow state. Used for status checks: "what step is this Workflow on?", "how many retries have happened?", "what's the current result?" Queries do not mutate state.

**Updates** (added 2024): Synchronous request-response interactions with a running Workflow. The caller sends data, the Workflow validates and processes it, and returns a response. Used for operations that need confirmation: "add this file to the review" → "file added, 3 files now pending."

---

## Dependency and Coordination

### Child Workflows

A Workflow can start other Workflows as children. The parent can wait for children, cancel them, or let them run independently. This is Temporal's composition mechanism:

```python
@workflow.defn
class ReleaseWorkflow:
    @workflow.run
    async def run(self, version: str):
        # Run in parallel: backend and frontend builds
        backend = await workflow.start_child_workflow(
            BuildBackendWorkflow, version
        )
        frontend = await workflow.start_child_workflow(
            BuildFrontendWorkflow, version
        )
        # Wait for both
        await backend
        await frontend
        # Sequential: deploy
        await workflow.execute_child_workflow(
            DeployWorkflow, version
        )
```

Child Workflows have their own event histories. If the parent crashes, children continue running (configurable). This provides the equivalent of Beads' `parent-child` dependency type but with execution semantics, not just tracking.

### Sagas and Compensation

For long-running processes that may need to be rolled back, Temporal supports the **Saga pattern**: each step has a corresponding compensation step. If a later step fails, compensations run in reverse order.

```python
@workflow.defn
class ProvisionWorkflow:
    @workflow.run
    async def run(self):
        compensations = []
        try:
            db = await workflow.execute_activity(create_database)
            compensations.append(("delete_database", db.id))

            cache = await workflow.execute_activity(create_cache)
            compensations.append(("delete_cache", cache.id))

            await workflow.execute_activity(deploy_service, db, cache)
        except Exception:
            # Roll back in reverse order
            for comp_name, comp_arg in reversed(compensations):
                await workflow.execute_activity(comp_name, comp_arg)
            raise
```

This has no equivalent in Claude Code or Beads. Neither system can automatically undo partially-completed work.

### Signals for Async Events

Signals enable event-driven coordination between Workflows and external systems:

- CI pipeline completes → Signal the deployment Workflow
- Human approves a PR → Signal the review Workflow
- A dependent service comes online → Signal the integration test Workflow
- An agent finishes a subtask → Signal the parent orchestration Workflow

Signals are durable -- they are recorded in the event history and survive crashes.

### Schedules for Recurring Work

Temporal **Schedules** trigger Workflow executions on a cron-like basis:

```python
# Run a code quality audit every Monday at 9am
schedule = await client.create_schedule(
    "weekly-code-audit",
    Schedule(
        action=ScheduleActionStartWorkflow(
            CodeAuditWorkflow.run,
            id="code-audit",
            task_queue="audit-workers"
        ),
        spec=ScheduleSpec(
            cron_expressions=["0 9 * * MON"]
        ),
    ),
)
```

This replaces external cron jobs or CI pipeline triggers. The Schedule itself is managed by the Temporal Server and survives server restarts.

---

## Setup Cost and Tradeoffs

### Development Setup

```bash
# Install Temporal CLI
brew install temporal  # or: curl -sSf https://temporal.download/cli.sh | sh

# Start development server (SQLite, single-process)
temporal server start-dev

# Server running at localhost:7233, Web UI at localhost:8233
```

Time: 5 minutes. The development server is a single binary with an embedded SQLite database. No Docker, no Postgres, no external dependencies.

### Production Setup

**Self-hosted (Docker Compose)**:
- Temporal Server + PostgreSQL/MySQL
- 2-4 GB RAM minimum for the server
- Persistent volume for the database
- Time: 1-2 hours for initial setup

**Self-hosted (Kubernetes)**:
- Helm chart: `helm install temporal temporalio/temporal`
- 3+ replicas per service for HA
- External database (PostgreSQL/MySQL/Cassandra)
- Monitoring: Prometheus metrics + Grafana dashboards
- Time: 1-2 days for production-grade setup

**Temporal Cloud**:
- Managed service, no infrastructure to run
- Pricing: ~$100/month base + per-action charges
- SLA: 99.99% availability
- Time: 30 minutes (sign up, get connection details, configure Workers)

### Learning Curve

Temporal's learning curve is the steepest of the three systems in this analysis. The core concepts require understanding:

1. **Determinism constraints**: Workflow code cannot use `time.now()`, `random()`, or non-deterministic library calls. This is counter-intuitive and a source of bugs for new users.

2. **Event sourcing mental model**: Understanding that Workflow state is derived from replay, not stored directly. This affects how you write Workflow code (no global mutable state, no closures over non-deterministic values).

3. **Activity design**: Deciding what belongs in an Activity vs. the Workflow. Rule of thumb: anything with side effects or I/O is an Activity. Everything else is Workflow logic.

4. **Versioning**: Changing Workflow code after executions have started requires careful versioning (`workflow.patched()`) to avoid non-determinism errors on replay.

5. **Debugging**: Temporal's Web UI shows the event history, but debugging replay-related issues requires understanding the replay model.

Estimated ramp-up time: 1-2 days for basic usage, 1-2 weeks for production confidence.

### Limits

- **Event history size**: Maximum ~51,200 events per Workflow Execution. Long-running Workflows must use "Continue-As-New" to reset the history while preserving state.
- **Payload size**: 2 MB default limit per Activity input/output. Large data should be stored externally (S3, database) with references passed through the Workflow.
- **Workflow Execution timeout**: Configurable, no hard limit. Workflows can run for years.
- **Task Queue capacity**: No hard limit, but very large backlogs can affect latency.

### When Temporal Is Overkill

Temporal adds infrastructure complexity. It is not justified when:

- **Solo developer, short sessions**: If you work alone on one project with sessions under an hour, Claude Code's built-in tasks are sufficient. Temporal's crash recovery is irrelevant when sessions are short enough that re-doing work is trivial.
- **No crash recovery requirements**: If a failed task can simply be re-run from scratch with no negative consequences, Temporal's event history adds overhead without benefit.
- **Single-step tasks**: If your "workflow" is "ask an agent to do something," there is no multi-step process to orchestrate. A shell script suffices.
- **Budget-constrained**: Self-hosting requires infrastructure; Temporal Cloud costs money. If the only alternative is a free tool, the cost may not be justified for small-scale use.

### When Temporal Is Justified

- **Long-running workflows**: Multi-hour or multi-day processes that must not lose progress
- **Multi-agent coordination**: Workflows that dispatch to multiple agents (potentially different providers) and aggregate results
- **Crash recovery is critical**: Infrastructure provisioning, deployment pipelines, financial transactions -- anywhere partial completion is worse than not starting
- **Vendor independence**: You want to switch between Claude, Codex, and open-source agents without rewriting orchestration logic
- **Cross-machine execution**: Agents run on different machines, and you need centralized orchestration
- **Compliance and auditability**: The event history provides a complete, immutable record of every step taken

---

## Comparison Table

Focused 3-way comparison on key dimensions relevant to AI agent orchestration.

| Dimension | Temporal | Claude Code Tasks | Beads |
|-----------|----------|-------------------|-------|
| **Primary function** | Workflow execution engine | In-session agent coordination | Persistent issue tracker |
| **Architectural layer** | Orchestration (above agents) | Execution (within agent) | Planning (across sessions) |
| **Crash recovery** | Automatic (replay + retry) | None (stuck in `in_progress`) | State survives (git); no auto-retry |
| **Persistence model** | Event history in database | JSON files scoped to session UUIDs | SQLite + JSONL in git |
| **Cross-session** | No sessions; Workflows run to completion | Effectively none (random UUID scoping) | Full (git-persisted, `bd ready`) |
| **Multi-project** | Namespaces | None | Multi-repo hydration |
| **Multi-agent** | Activity routing + Task Queues | File locking + claiming (same machine) | Hash IDs + git distribution |
| **Cross-machine** | Native (Workers anywhere) | None (filesystem only) | Git-based |
| **Vendor independence** | Full (Activities wrap any CLI) | None (Claude Code-specific) | Partial (Beads-specific, but CLI-accessible) |
| **Determinism guarantee** | Enforced (replay-safe Workflow code) | None (agent-driven) | None (agent-driven) |
| **Dependency model** | Child Workflows, Signals, Sagas | blocks/blockedBy (2 types) | 6 dependency types |
| **Workflow templates** | Workflow code (versioned, tested) | None | Formulas (TOML) |
| **Human-in-the-loop** | Signals + wait_condition | None | Gates |
| **Setup cost** | Medium-High (server + Workers) | Zero (built-in) | Low (CLI + `bd init`) |
| **Learning curve** | Steep (determinism, event sourcing) | Zero | Moderate (chemistry metaphors) |
| **Infrastructure** | Server cluster or Cloud subscription | None | Git (existing) |
| **Compaction** | Continue-As-New (manual) | None | Semantic memory decay |
| **Auditability** | Complete event history | None | Git history |

---

## When to Use Temporal

### Decision Framework

**Use Temporal when:**

1. **Crash recovery is non-negotiable**. Your workflows must not lose progress on failure. Agent sessions that crash mid-task should resume automatically, not require human intervention.

2. **You orchestrate multiple agents** (potentially different providers). Claude for reasoning, Codex for code generation, a local model for quick tasks. Temporal routes to the right agent and handles failures uniformly.

3. **Vendor independence matters**. You want to switch AI providers without rewriting workflow logic. Temporal's Activity abstraction makes agents interchangeable.

4. **Workflows span machines**. Agents run on different servers. You need centralized orchestration with distributed execution.

5. **Workflows are long-running**. Multi-hour code reviews, multi-day migration projects, deployment pipelines that wait for human approvals. These need durable state that survives restarts.

6. **You need auditability**. Regulatory or compliance requirements demand a complete record of what was done, when, and by which agent.

**Do not use Temporal when:**

1. **You are a solo developer** with short sessions on a single project. The infrastructure overhead is not justified. Use Claude Code's built-in tasks.

2. **Tasks are independent and idempotent**. If each task can be re-run from scratch with no cost, crash recovery adds complexity without value. A simple script that reads a task list and invokes `claude --print` is sufficient.

3. **Your budget is zero**. Temporal Cloud costs money. Self-hosting requires infrastructure. If you cannot allocate resources, use Beads (free, git-backed) or a simple file-based orchestrator.

4. **You only use one agent provider** and expect to continue doing so. Vendor independence has no value if you are committed to a single vendor. (Though this commitment may be riskier than it appears in a fast-moving landscape.)

### Recommended Combinations

**Temporal + Claude Code Tasks**: Temporal handles the outer orchestration loop (what to work on, crash recovery, multi-agent routing). Claude Code's Tasks handle the inner loop (agent self-organization during a session). This is the natural layering.

**Temporal + Beads**: Beads provides the planning and issue-tracking layer (persistent backlog, ready computation, multi-project view, compaction). Temporal provides the execution layer (run workflows to completion, handle crashes, coordinate agents). A Temporal Workflow reads from Beads (`bd ready`), claims work, orchestrates agents, and closes the Beads issue when done.

**Temporal + Beads + Claude Code Tasks**: The full stack. Beads is the planning layer (what needs doing). Temporal is the orchestration layer (ensuring it gets done). Claude Code Tasks is the execution layer (how the agent organizes its work within a session). Each layer operates at its natural level of abstraction.

```
┌─────────────────────────────────────────┐
│           Beads (Planning)              │  What needs doing?
│  Issues, dependencies, ready work,      │  Multi-project backlog
│  compaction, git distribution            │
├─────────────────────────────────────────┤
│         Temporal (Orchestration)         │  Ensuring it gets done
│  Workflows, crash recovery, routing,     │  Multi-agent coordination
│  retry policies, event history           │
├─────────────────────────────────────────┤
│     Claude Code Tasks (Execution)        │  How the agent works
│  In-session subtasks, dependencies,      │  Agent self-organization
│  file locking, team coordination         │
└─────────────────────────────────────────┘
```

### Temporal Is Not a Replacement

Temporal does not replace Claude Code Tasks or Beads. It addresses a different concern:

- Claude Code Tasks answers: "How does the agent organize work within a session?"
- Beads answers: "What work exists across projects and sessions?"
- Temporal answers: "How do we guarantee work completes despite failures?"

These are complementary questions. The right combination depends on which questions matter for your situation. For most solo developers, only the first question matters. For teams running multi-agent workflows in production, all three matter.

---

*This analysis is part of the task-analysis project. See `task-system-comparison.md` for the 3-way comparison, `orchestration-principle.md` for the underlying principle, and `next-step.md` for practical next steps.*
