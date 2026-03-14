# Multi-Agent Teams System — Implementation Plan

## What This Is

A faithful recreation of Claude Code's Agent Teams system, built directly against the Claude API (not the CLI). Every lead agent is **Opus 4.6**. Every agent runs the same grind loop that powers Claude Code — the think → tool_use → tool_result → think cycle that makes agents autonomous reasoners, not task-executors.

This is not a task queue. This is not a workflow engine. These are agents that think, reason, react, analyze, use tools, search the web, read code, write code, run commands, and communicate with each other — just like Claude Code's real agent teams.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     USER / CLI INTERFACE                     │
│  Start a team, talk to lead, talk to any teammate directly  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      TEAM LEAD (Opus 4.6)                   │
│                                                             │
│  Grind Loop: think → tool_use → tool_result → think → ...  │
│                                                             │
│  Tools: Read, Edit, Write, Bash, Glob, Grep, WebSearch,    │
│         WebFetch, Agent (subagent), Teammate (team ops),    │
│         TaskCreate, TaskList, TaskUpdate, TaskGet           │
│                                                             │
│  System Prompt: Full Claude Code system prompt +            │
│                 team coordination instructions              │
└──┬──────────┬──────────┬──────────┬─────────────────────────┘
   │          │          │          │
   ▼          ▼          ▼          ▼
┌────────┐┌────────┐┌────────┐┌────────┐
│Agent A ││Agent B ││Agent C ││Agent N │  ← Teammates (Opus 4.6)
│        ││        ││        ││        │
│Own Loop││Own Loop││Own Loop││Own Loop│  ← Each runs grind loop
│Own Ctx ││Own Ctx ││Own Ctx ││Own Ctx │  ← Own context window
│Own Tool││Own Tool││Own Tool││Own Tool│  ← Full tool access
└────────┘└────────┘└────────┘└────────┘
     ▲          ▲          ▲          ▲
     │          │          │          │
     └──────────┴─────┬────┴──────────┘
                      │
              ┌───────┴────────┐
              │  Shared State  │
              │  - Task List   │  ← JSON files on disk
              │  - Inboxes     │  ← Per-agent message files
              │  - Team Config │  ← Members, metadata
              │  - File Locks  │  ← Race condition prevention
              └────────────────┘
```

---

## Core Principles

1. **Every agent is a full reasoning entity** — not a function call, not a task worker. Each agent gets the full Claude Code system prompt, full tool access, and runs its own autonomous grind loop until its work is done. It thinks with extended thinking. It streams its reasoning live. It changes course when things don't go as expected. It forms hypotheses, tests them, and adjusts.

2. **Streaming, always streaming** — every API call is a streaming call. You see agents think in real-time. Tokens arrive word-by-word. Tool calls form as you watch. Bash output streams live. Nothing is batch. Nothing is "wait for response." This is 2026 — agents stream their consciousness.

3. **Communication is peer-to-peer via inboxes** — agents don't just report back to the lead. They message each other directly. A backend agent discovers a schema issue and messages the frontend agent spontaneously — not because a DAG told it to, but because it REASONED that the other agent needs to know. The lead coordinates, but teammates can challenge each other, share discoveries, and argue.

4. **The grind loop IS the agent** — there is no "orchestrator" deciding what to do. Each agent calls the Claude API with streaming, processes the streamed response, executes any tool calls (which also stream), feeds results back, and continues thinking. The agent decides what to do next. Always. It might read 10 files. It might read 1 file and realize the problem is elsewhere. It might call WebSearch to look up a library API. It might spawn a subagent to investigate a hypothesis while it continues on the main thread.

5. **Shared task list is a coordination aid, not a DAG** — tasks are JSON files on disk. But the task list doesn't control the agents. The agents control the task list. They claim tasks, yes. But they also CREATE new tasks when they discover unexpected complexity. They SPLIT tasks when they realize one is too big. They SKIP tasks and tell the lead why. The task list is a shared whiteboard, not a conveyor belt.

6. **Opus 4.6 everywhere that matters** — lead agents and teammates are all `claude-opus-4-6` with extended thinking. These agents don't just follow instructions — they reason about them, question them, and produce better solutions than what was asked for. Subagents (quick throwaway workers within a single agent's loop) can be Sonnet/Haiku for cost efficiency.

---

## Directory Structure

```
project/
├── src/
│   ├── core/
│   │   ├── grind_loop.py          # THE agentic loop — heart of everything
│   │   ├── api_client.py          # Claude API wrapper (Messages API)
│   │   ├── context_manager.py     # Message accumulation, compaction
│   │   └── tool_executor.py       # Parallel/sequential tool execution
│   │
│   ├── tools/
│   │   ├── __init__.py            # Tool registry
│   │   ├── base.py                # Tool base class + schema
│   │   ├── file_ops.py            # Read, Edit, Write, Glob, Grep
│   │   ├── bash.py                # Bash execution (sandboxed)
│   │   ├── web.py                 # WebSearch, WebFetch
│   │   ├── agent.py               # Subagent spawning (within single loop)
│   │   ├── teammate.py            # TeammateTool — all team operations
│   │   ├── tasks.py               # TaskCreate, TaskList, TaskUpdate, TaskGet
│   │   └── todo.py                # TodoWrite for per-agent tracking
│   │
│   ├── team/
│   │   ├── manager.py             # Team lifecycle (create, join, cleanup)
│   │   ├── inbox.py               # Inbox read/write with file locking
│   │   ├── task_store.py          # Task JSON persistence + dependency resolution
│   │   ├── config.py              # Team config.json management
│   │   ├── spawn.py               # Teammate spawning (subprocess/async)
│   │   └── file_lock.py           # Cross-process file locking
│   │
│   ├── prompts/
│   │   ├── system_prompt.py       # Full system prompt (Claude Code faithful)
│   │   ├── lead_prompt.py         # Lead-specific additions
│   │   ├── teammate_prompt.py     # Teammate-specific additions
│   │   └── tool_definitions.py    # All tool JSON schemas for the API
│   │
│   ├── agents/
│   │   ├── lead_agent.py          # Lead agent — wraps grind loop + team tools
│   │   ├── teammate_agent.py      # Teammate agent — wraps grind loop + inbox
│   │   └── subagent.py            # Lightweight subagent (within a single loop)
│   │
│   └── cli/
│       ├── main.py                # Entry point
│       ├── display.py             # Terminal UI (teammate switching, task view)
│       └── router.py              # Routes user input to lead or specific teammate
│
├── state/                          # Runtime state (gitignored)
│   ├── teams/{team-name}/
│   │   ├── config.json
│   │   └── inboxes/{agent-name}.json
│   ├── tasks/{team-name}/
│   │   ├── 1.json
│   │   ├── 2.json
│   │   └── ...
│   └── locks/
│
├── CLAUDE.md                       # Project instructions for agents
├── pyproject.toml
├── requirements.txt
└── README.md
```

---

## Part 1: The Grind Loop (`src/core/grind_loop.py`)

This is the heart. Every agent — lead, teammate, subagent — runs this same loop. It is a direct implementation of the Claude Code agentic loop.

**This is NOT a batch request-response cycle.** This is a streaming, thinking agent. Every API call streams tokens in real-time. You see the agent's reasoning appear word-by-word. You see it decide to call a tool mid-thought. You see the tool results flow back. You see it react, adjust, change course. This is how every serious AI builder works in 2026 — streaming agents that think out loud.

### The Loop

```python
async def grind_loop(
    client: AnthropicClient,
    system_prompt: str,
    tools: list[ToolDefinition],
    initial_messages: list[dict],
    *,
    max_turns: int | None = None,
    max_budget_usd: float | None = None,
    on_text_delta: Callable | None = None,    # STREAMING: each token as it arrives
    on_thinking_delta: Callable | None = None, # STREAMING: extended thinking tokens
    on_tool_start: Callable | None = None,    # Tool about to execute
    on_tool_stream: Callable | None = None,   # Tool output streaming (e.g. bash)
    on_tool_end: Callable | None = None,      # Tool finished
    inbox_check: Callable | None = None,      # Periodic inbox polling
    should_stop: Callable | None = None,      # External stop signal
    extended_thinking: bool = True,            # Enable chain-of-thought
) -> GrindResult:
    """
    The agentic loop. Faithful to Claude Code's execution model.
    STREAMING throughout — no waiting for complete responses.

    The agent THINKS (extended thinking shows reasoning), SPEAKS (text
    output streams token by token), ACTS (tool calls execute and stream
    results), and REACTS (processes results, adjusts approach, continues).

    This is not request → response → request → response.
    This is a continuous stream of consciousness that uses tools as
    extensions of its thinking process.
    """
    messages = list(initial_messages)
    total_cost = 0.0
    turn_count = 0

    while True:
        # --- Pre-turn: check for inbox messages ---
        if inbox_check:
            new_messages = await inbox_check()
            if new_messages:
                # Inject inbox messages as context — the agent reacts
                # to these like a human reads a Slack notification
                messages.append({
                    "role": "user",
                    "content": format_inbox_messages(new_messages)
                })

        # --- Pre-turn: check stop conditions ---
        if should_stop and await should_stop():
            break
        if max_turns and turn_count >= max_turns:
            break
        if max_budget_usd and total_cost >= max_budget_usd:
            break

        # --- Context management: compact if needed ---
        if estimate_tokens(messages) > COMPACTION_THRESHOLD:
            messages = await compact_context(client, messages, system_prompt)

        # --- THE CALL: Stream from Claude API ---
        # We use streaming so the agent's thinking is visible in real-time.
        # Extended thinking shows the agent's internal reasoning chain.
        # Text output shows what it's saying. Tool calls show what it's doing.
        # Everything streams. Nothing batches.

        collected_content = []
        current_tool_inputs = {}

        async with client.messages.stream(
            model="claude-opus-4-6",
            max_tokens=16384,
            system=system_prompt,
            tools=tools,
            messages=messages,
            thinking={                          # Extended thinking enabled
                "type": "enabled",
                "budget_tokens": 10000,
            } if extended_thinking else anthropic.NOT_GIVEN,
        ) as stream:
            async for event in stream:
                match event.type:
                    # --- Agent is THINKING (internal reasoning) ---
                    case "content_block_start" if event.content_block.type == "thinking":
                        pass  # thinking block started
                    case "content_block_delta" if hasattr(event.delta, "thinking"):
                        if on_thinking_delta:
                            await on_thinking_delta(event.delta.thinking)
                        # You see the agent reason: "The user wants me to
                        # refactor auth. Let me first read the current
                        # implementation to understand the structure..."

                    # --- Agent is SPEAKING (visible text output) ---
                    case "content_block_delta" if event.delta.type == "text_delta":
                        if on_text_delta:
                            await on_text_delta(event.delta.text)
                        # You see words appear one by one in the terminal

                    # --- Agent is DECIDING TO ACT (tool call forming) ---
                    case "content_block_start" if event.content_block.type == "tool_use":
                        current_tool_inputs[event.index] = {
                            "id": event.content_block.id,
                            "name": event.content_block.name,
                            "input_json": "",
                        }
                    case "content_block_delta" if event.delta.type == "input_json_delta":
                        current_tool_inputs[event.index]["input_json"] += event.delta.partial_json

                    case "content_block_stop":
                        pass  # Block finalized

            # Collect the complete response after stream ends
            response = await stream.get_final_message()

        total_cost += calculate_cost(response.usage)

        # --- Process complete response ---
        assistant_message = {
            "role": "assistant",
            "content": response.content,
        }
        messages.append(assistant_message)

        # --- Check if done (no tool calls) ---
        tool_use_blocks = [b for b in response.content if b.type == "tool_use"]

        if not tool_use_blocks:
            # Agent decided it's done. Not because we told it to stop.
            # Because it REASONED that the work is complete.
            break

        # --- Execute tools (with streaming for long-running ones) ---
        tool_results = await execute_tools(
            tool_use_blocks,
            tools,
            on_tool_start=on_tool_start,
            on_tool_stream=on_tool_stream,  # Bash output streams live
            on_tool_end=on_tool_end,
        )

        # Tool results go back — agent sees what happened and REACTS
        messages.append({
            "role": "user",
            "content": tool_results,
        })

        turn_count += 1

    return GrindResult(
        messages=messages,
        final_text=extract_final_text(messages),
        total_cost=total_cost,
        turn_count=turn_count,
    )
```

### Why This Is Not a Task DAG

A task DAG says: "Step 1: read file. Step 2: edit file. Step 3: run tests." The steps are pre-determined. The agent is a puppet.

The grind loop says: "Here's your goal. Figure it out." The agent decides:
- **What to investigate first** — maybe it reads the test file before the source, because test failures are more informative
- **When to change approach** — if the first fix breaks something else, it backs up, re-analyzes, tries something different
- **When to communicate** — if it discovers something another teammate needs to know, it messages them spontaneously
- **When it's done** — not when a counter hits zero, but when the agent REASONS that the work is complete and correct

The extended thinking makes this visible. You see the reasoning:

> *"I read the auth middleware and see it strips the session token at line 47. But wait — the test expects the token to be present. Let me check if there's a middleware ordering issue... yes, the cors middleware runs after auth, which means... I need to swap the middleware order in app.ts, not fix the auth logic itself."*

That's not a task. That's thinking.

### Key Design Decisions

- **Streaming throughout.** Every API call uses `messages.stream()`. You see the agent think in real-time. Tokens arrive word-by-word. Tool calls form as you watch. This is not batch processing.
- **Extended thinking enabled.** Opus 4.6 with extended thinking produces internal chain-of-thought before responding. The agent reasons about what to do, considers alternatives, then acts. This is visible in the stream.
- **No orchestrator above the loop.** The loop IS the agent. Claude decides what to do next, what tools to call, when to stop. There is no DAG, no workflow, no pre-planned sequence.
- **Inbox injection between turns.** When a teammate sends a message, it appears as context in the next turn — the agent reacts to it naturally, the way a human would react to a Slack message while working. It might change what it's doing. It might disagree and explain why. It reasons about the message.
- **Compaction preserves agency.** When context gets long, we summarize older turns but keep recent decisions, file paths modified, and task state. The agent doesn't lose track of what it was doing.
- **Parallel tool execution for read-only tools.** If Claude calls `Read` on 3 files simultaneously, we execute them in parallel. `Edit` and `Bash` run sequentially to avoid conflicts.
- **Tool output streams too.** When Bash runs a long command, output streams live. The agent (and the user) can see it happening. Not "wait 30 seconds, then dump output."
- **Real-time steering (mid-task injection).** Claude Code internally uses an async dual-buffer queue (codenamed "h2A") that lets users inject new instructions while the agent is actively working. We replicate this: the user can type a message to any agent mid-turn, and it gets queued to appear in the agent's next context window. The agent adjusts its plan on the fly without restarting. This is critical for the lead-to-teammate communication flow — the lead might redirect a teammate mid-task based on new information.

---

## Part 2: Tool System (`src/tools/`)

### Tool Schema Format

Every tool is defined as a JSON schema that gets sent to the Claude API. The agent sees these definitions and decides which to call.

```python
class Tool:
    """Base class for all tools."""
    name: str
    description: str
    input_schema: dict          # JSON Schema for parameters
    is_read_only: bool = False  # Can run in parallel if True

    async def execute(self, input: dict, context: ToolContext) -> str:
        """Execute the tool and return result as string."""
        raise NotImplementedError
```

### Tool Inventory

These match Claude Code's actual tool set:

| Tool | Description | Read-Only | Notes |
|------|-------------|-----------|-------|
| `Read` | Read file contents | Yes | Supports offset/limit, images, PDFs |
| `Edit` | String replacement in files | No | Requires `old_string` uniqueness |
| `Write` | Create/overwrite files | No | Must read first if file exists |
| `Glob` | Find files by pattern | Yes | Returns paths sorted by mtime |
| `Grep` | Search file contents (ripgrep) | Yes | Regex, file type filters |
| `Bash` | Execute shell commands | No | Sandboxed, timeout support |
| `WebSearch` | Search the web | Yes | Returns formatted results |
| `WebFetch` | Fetch and parse web page | Yes | HTML→markdown conversion |
| `Agent` | Spawn a subagent | Yes* | Runs grind loop in subprocess |
| `Teammate` | Team operations | No | 11 operations (see below) |
| `TaskCreate` | Create a task | No | Subject, description, dependencies |
| `TaskList` | View all tasks | Yes | Computes availability dynamically |
| `TaskUpdate` | Update task status/owner | No | File-locked writes |
| `TaskGet` | Get task details | Yes | Single task by ID |
| `TodoWrite` | Per-agent progress tracking | No | Private to each agent |

### TeammateTool Operations

The `Teammate` tool is the communication backbone. It supports these operations:

```python
TEAMMATE_OPERATIONS = {
    "spawnTeam":        "Create a new team → config.json + task directory",
    "discoverTeams":    "List available teams",
    "requestJoin":      "Request to join a team",
    "approveJoin":      "Leader accepts join request",
    "rejectJoin":       "Leader declines join request",
    "write":            "Send message to specific teammate (by agent ID)",
    "broadcast":        "Send message to ALL teammates (expensive)",
    "requestShutdown":  "Ask teammate to exit (leader only)",
    "approveShutdown":  "Accept shutdown request (teammate only)",
    "rejectShutdown":   "Decline with explanation",
    "approvePlan":      "Approve teammate's plan (leader only)",
    "rejectPlan":       "Reject plan with feedback (leader only)",
    "cleanup":          "Remove team + all state (leader only, no active members)",
}
```

**Critical rule**: Agent text output is NOT visible to the team. Agents MUST use `write` or `broadcast` to communicate. This is enforced in the system prompt.

---

## Part 3: Communication System (`src/team/`)

### Inbox Architecture

Every agent has an inbox file. Communication is asynchronous, file-based, with JSON.

```
state/teams/{team-name}/inboxes/
├── team-lead.json        # Lead's inbox
├── frontend-agent.json   # Teammate inbox
├── backend-agent.json    # Teammate inbox
└── test-agent.json       # Teammate inbox
```

### Message Format

```python
@dataclass
class InboxMessage:
    from_agent: str           # sender agent name
    text: str                 # message content (plain text or stringified JSON)
    summary: str | None       # short preview for UI
    timestamp: str            # ISO 8601
    color: str | None         # agent color for display
    read: bool = False        # mark as read when processed

    # For structured messages, text contains JSON:
    # - shutdown_request: {"type": "shutdown_request", "requestId": "...", "from": "...", "reason": ""}
    # - shutdown_approved: {"type": "shutdown_approved", ...}
    # - idle_notification: {"type": "idle_notification", ...}
    # - task_completed: {"type": "task_completed", "taskId": "...", ...}
    # - plan_approval_request: {"type": "plan_approval_request", "plan": "...", ...}
    # - permission_request: {"type": "permission_request", "tool": "...", ...}
```

### Inbox Read/Write with Locking

```python
class Inbox:
    """Thread-safe, process-safe inbox operations."""

    def __init__(self, team_name: str, agent_name: str, state_dir: str):
        self.path = Path(state_dir) / "teams" / team_name / "inboxes" / f"{agent_name}.json"

    async def read_new(self) -> list[InboxMessage]:
        """Read unread messages, mark them read."""
        async with file_lock(self.path):
            messages = json.loads(self.path.read_text()) if self.path.exists() else []
            unread = [m for m in messages if not m["read"]]
            for m in messages:
                m["read"] = True
            self.path.write_text(json.dumps(messages, indent=2))
        return [InboxMessage(**m) for m in unread]

    async def write_message(self, message: InboxMessage):
        """Append a message to this inbox."""
        async with file_lock(self.path):
            messages = json.loads(self.path.read_text()) if self.path.exists() else []
            messages.append(asdict(message))
            self.path.write_text(json.dumps(messages, indent=2))
```

### Message Types (Complete)

The `text` field in inbox messages is either plain text or stringified JSON with a `type` field:

| Type | Direction | Purpose |
|------|-----------|---------|
| `task_assignment` | lead → teammate | Assign work |
| `message` | any → any | Direct peer message (plain text) |
| `broadcast` | lead → all | Team-wide announcement (writes to every inbox) |
| `shutdown_request` | lead → teammate | Graceful termination request |
| `shutdown_approved` | teammate → lead | Accepts shutdown |
| `shutdown_rejected` | teammate → lead | Declines with reason |
| `plan_approval_request` | teammate → lead | Submit plan for review |
| `plan_approved` | lead → teammate | Approve plan |
| `plan_rejected` | lead → teammate | Reject plan with feedback |
| `idle_notification` | teammate → lead | Heartbeat — "I'm free" signal |
| `task_completed` | teammate → lead/all | Task status update |
| `permission_request` | teammate → lead | Tool permission escalation |
| `join_request` | agent → lead | Request team membership |

### Idle/Wake Protocol

After every LLM turn where a teammate has no more work, it automatically sends `idle_notification` to the lead every ~2-4 seconds (acts as both "I'm available" and heartbeat). Sending any message to an idle teammate **wakes it** on the next poll cycle. This is how the lead re-activates teammates when new tasks are created or when it needs something done.

### Inbox Polling

Each agent's grind loop has an `inbox_check` callback that polls between turns:

```python
async def make_inbox_checker(inbox: Inbox):
    """Returns a callable that checks for new inbox messages."""
    async def check():
        messages = await inbox.read_new()
        if messages:
            return messages
        return None
    return check
```

Messages are injected into the agent's context as a user message like:
```
[INBOX] Message from backend-agent:
I found the root cause — the auth middleware is stripping the session token
before it reaches the route handler. See src/middleware/auth.ts:47.
```

The agent then naturally reacts to this information in its next thinking step.

---

## Part 4: Task System (`src/team/task_store.py`)

### Task Schema

Each task is a separate JSON file in `state/tasks/{team-name}/`:

```python
@dataclass
class Task:
    id: str                         # "1", "2", etc.
    subject: str                    # Short title
    description: str                # Full description of what to do
    active_form: str                # Present continuous ("Implementing auth")
    status: str                     # "pending" | "in_progress" | "completed"
    owner: str | None = None        # Agent name that claimed it
    blocked_by: list[str] = field(default_factory=list)  # Task IDs
    created_at: str = ""            # ISO 8601
    completed_at: str | None = None
```

The task directory also contains:
- `.lock` — filesystem-level `flock()` mutex for atomic task claiming
- `.highwatermark` — next available task ID (auto-incrementing counter)

Example `state/tasks/my-project/3.json`:
```json
{
    "id": "3",
    "subject": "Integration testing",
    "description": "Write integration tests for the auth + API layer interaction",
    "active_form": "Writing integration tests",
    "status": "pending",
    "owner": null,
    "blocked_by": ["1", "2"],
    "created_at": "2026-03-14T10:00:00Z",
    "completed_at": null
}
```

### Dependency Resolution

Dependencies are **declarative, not imperative**. The `blocked_by` field never changes after creation. Availability is computed dynamically:

```python
class TaskStore:
    async def get_available_tasks(self) -> list[Task]:
        """Return tasks that are pending and have all dependencies completed."""
        all_tasks = await self.list_all()
        completed_ids = {t.id for t in all_tasks if t.status == "completed"}

        available = []
        for task in all_tasks:
            if task.status != "pending":
                continue
            if task.owner is not None:
                continue
            # Check if all blocking tasks are completed
            if all(dep_id in completed_ids for dep_id in task.blocked_by):
                available.append(task)

        return available

    async def claim_task(self, task_id: str, agent_name: str) -> bool:
        """Atomically claim a task using file locking."""
        task_path = self.task_dir / f"{task_id}.json"
        async with file_lock(task_path):
            task = self.read_task(task_path)
            if task.status != "pending" or task.owner is not None:
                return False  # Already claimed
            task.status = "in_progress"
            task.owner = agent_name
            self.write_task(task_path, task)
            return True
```

### Self-Claiming Flow

After completing a task, a teammate automatically:
1. Marks current task as `completed`
2. Calls `TaskList` to see available tasks
3. Calls `TaskUpdate` to claim the next unblocked, unowned task
4. Begins working on it

This is driven by the system prompt — not by code. The agent decides to do this because the prompt tells it to.

---

## Part 5: Agent Types (`src/agents/`)

### What Makes These Agents, Not Functions

A function takes input, produces output. An agent:
- **Perceives** its environment (reads files, searches code, checks inbox)
- **Reasons** about what it sees (extended thinking, chain of thought)
- **Plans** its next action (might change mid-execution based on what it learns)
- **Acts** through tools (reads, writes, runs commands, communicates)
- **Reflects** on the results (did the test pass? did the teammate's message change my approach?)
- **Adapts** when things don't go as expected (tries a different approach, asks for help, creates new tasks)

Every agent runs the same grind loop. The difference is context and permissions.

### Lead Agent

```python
class LeadAgent:
    """
    The team lead. Opus 4.6 with extended thinking.

    The lead doesn't follow a script. It receives a user's intent, REASONS
    about how to decompose it, decides how many teammates to spawn and what
    each should focus on, creates tasks (but knows it might need to revise
    them), and then monitors + synthesizes.

    The lead can also DO WORK ITSELF. If it sees something quick that needs
    doing, it just does it. It's not a pure coordinator — it's the most
    capable agent on the team.
    """

    def __init__(self, team_name: str, api_key: str):
        self.model = "claude-opus-4-6"
        self.team_name = team_name
        self.client = AnthropicClient(api_key)
        self.inbox = Inbox(team_name, "team-lead", STATE_DIR)
        self.task_store = TaskStore(team_name, STATE_DIR)

        # Lead gets ALL tools — it's the most capable agent
        self.tools = [
            ReadTool(), EditTool(), WriteTool(), GlobTool(), GrepTool(),
            BashTool(), WebSearchTool(), WebFetchTool(),
            AgentTool(),         # Spawn subagents within own loop
            TeammateTool(),      # Team operations (spawn, message, shutdown)
            TaskCreateTool(), TaskListTool(), TaskUpdateTool(), TaskGetTool(),
            TodoWriteTool(),
        ]

        self.system_prompt = build_system_prompt(
            role="lead",
            team_name=team_name,
            claude_md=load_claude_md(),
        )

    async def run(self, user_prompt: str):
        """
        Start the lead's grind loop. STREAMING.

        The lead will:
        1. Think about the problem (you see extended thinking stream)
        2. Maybe explore the codebase first (tool calls stream)
        3. Decide on team structure (reasoning visible)
        4. Create tasks and spawn teammates
        5. Monitor inbox, react to teammate messages
        6. Maybe intervene, redirect, do work itself
        7. Synthesize results when teammates finish
        8. Report back to user

        Steps 1-8 are NOT pre-programmed. The lead DECIDES to do these
        things because it's an intelligent agent reasoning about the problem.
        """
        return await grind_loop(
            client=self.client,
            system_prompt=self.system_prompt,
            tools=[t.to_api_schema() for t in self.tools],
            initial_messages=[{"role": "user", "content": user_prompt}],
            inbox_check=make_inbox_checker(self.inbox),
            on_text_delta=self.stream_text,       # Live text output
            on_thinking_delta=self.stream_thinking, # Live reasoning
            on_tool_start=self.display_tool_start,
            on_tool_stream=self.display_tool_output,
            on_tool_end=self.display_tool_end,
            extended_thinking=True,
        )
```

### Teammate Agent

```python
class TeammateAgent:
    """
    A teammate. Opus 4.6 with extended thinking. Fully autonomous.

    A teammate is NOT a worker bee. It's a senior engineer dropped into
    a project with a mandate. It reads code, forms its own understanding,
    makes its own decisions, and communicates when it has something to say.

    If it disagrees with the lead's plan, it says so. If it finds a bug
    unrelated to its task, it reports it. If the task turns out to be
    more complex than expected, it splits it and creates new tasks.
    """

    def __init__(self, team_name: str, agent_name: str, spawn_prompt: str, api_key: str):
        self.model = "claude-opus-4-6"
        self.team_name = team_name
        self.agent_name = agent_name
        self.spawn_prompt = spawn_prompt
        self.client = AnthropicClient(api_key)
        self.inbox = Inbox(team_name, agent_name, STATE_DIR)
        self.task_store = TaskStore(team_name, STATE_DIR)

        # Teammates get full tool access — they're full agents
        self.tools = [
            ReadTool(), EditTool(), WriteTool(), GlobTool(), GrepTool(),
            BashTool(), WebSearchTool(), WebFetchTool(),
            AgentTool(),         # Can spawn subagents for quick research
            TeammateTool(),      # Can message other teammates + lead
            TaskListTool(), TaskUpdateTool(), TaskGetTool(),
            TodoWriteTool(),
        ]

        self.system_prompt = build_system_prompt(
            role="teammate",
            team_name=team_name,
            agent_name=agent_name,
            claude_md=load_claude_md(),
        )

    async def run(self):
        """
        Start the teammate's grind loop with the spawn prompt.

        The teammate will:
        - Read the spawn prompt and REASON about the work
        - Explore the codebase to build its own understanding
        - Claim a task (or create tasks if none exist yet)
        - Do the actual work — reading, thinking, coding, testing
        - Communicate findings to teammates and lead
        - Self-claim the next task when done
        - Go idle and notify lead when all work is complete

        All of this emerges from the agent's reasoning, not from code.
        """
        return await grind_loop(
            client=self.client,
            system_prompt=self.system_prompt,
            tools=[t.to_api_schema() for t in self.tools],
            initial_messages=[{"role": "user", "content": self.spawn_prompt}],
            inbox_check=make_inbox_checker(self.inbox),
            should_stop=self.check_shutdown,
            on_text_delta=self.stream_text,
            on_thinking_delta=self.stream_thinking,
            on_tool_start=self.display_tool_start,
            on_tool_stream=self.display_tool_output,
            on_tool_end=self.display_tool_end,
            extended_thinking=True,
        )
```

### Subagent (Within a Single Loop)

```python
class Subagent:
    """
    Lightweight agent spawned by Agent tool within a single agent's loop.
    Can be Sonnet/Haiku for cost efficiency on focused tasks.

    Even subagents run the full grind loop. A subagent spawned to
    "search for all uses of the deprecated API" will read files, grep,
    follow import chains, and reason about what it finds. It's not
    just a grep wrapper — it's a thinking entity scoped to a task.
    """

    def __init__(self, model: str, prompt: str, tools: list[Tool], api_key: str):
        self.model = model  # Can be sonnet, haiku, or opus
        self.tools = tools  # Scoped tool set (often read-only for safety)

    async def run(self) -> str:
        """Run grind loop, return final text result to parent."""
        result = await grind_loop(
            client=self.client,
            system_prompt=SUBAGENT_SYSTEM_PROMPT,
            tools=[t.to_api_schema() for t in self.tools],
            initial_messages=[{"role": "user", "content": self.prompt}],
            max_turns=30,
            extended_thinking=self.model.startswith("claude-opus"),
        )
        return result.final_text  # Only the summary returns to parent
```

---

## Part 5b: How Agents Actually Think — A Real Execution Trace

This is what happens when a teammate agent works on "Fix the auth middleware bug." Not a simplified diagram — the actual flow of consciousness:

```
[EXTENDED THINKING — streaming, visible to display]
"I need to fix the auth middleware bug. Let me first understand what the auth
middleware does. The spawn prompt mentions src/middleware/auth.ts. Let me
read that file."

[TOOL CALL: Read(file_path="src/middleware/auth.ts")]
→ File contents stream back

[EXTENDED THINKING]
"I see the middleware extracts the JWT from the Authorization header at line 12,
validates it at line 18, and attaches the user to req.user at line 25. But wait —
there's a subtle issue. On line 14, it does `token.split(' ')[1]` but doesn't check
if the split produces two parts. If someone sends 'Bearer' without a token, this
returns undefined, and line 18 will throw on jwt.verify(undefined).

But is this the actual bug? Let me check the test file to see what's failing."

[TOOL CALL: Grep(pattern="auth.*fail|auth.*error", glob="**/*.test.*")]
→ Results stream back: 3 matches in auth.test.ts

[TOOL CALL: Read(file_path="src/__tests__/auth.test.ts")]
→ File contents stream back

[EXTENDED THINKING]
"The failing test on line 45 sends a request with header 'Authorization: Bearer'
(no token after Bearer). The test expects a 401 response but gets a 500 because
jwt.verify throws on undefined instead of the middleware catching it.

But actually, I notice something else. Line 72 has a test for session-based auth
that's ALSO failing. The session test has nothing to do with JWT. Let me look at
the middleware ordering..."

[TOOL CALL: Read(file_path="src/app.ts")]
→ Streams back

[EXTENDED THINKING]
"Found it. In app.ts line 34, cors middleware is registered AFTER auth middleware.
CORS preflight requests hit auth first, fail because OPTIONS requests don't carry
auth headers, and the response never gets CORS headers. This is two bugs:

1. JWT parsing doesn't handle missing token (causes 500 instead of 401)
2. Middleware order wrong — cors must come before auth

The teammate working on routes needs to know about #2 because it affects their
API endpoint testing. Let me message them AND fix both issues."

[TOOL CALL: Teammate(operation="write", target_agent_id="routes-agent@auth-refactor",
  value="Found a middleware ordering bug in app.ts:34 — CORS runs after auth.
  Your route tests will fail on preflight requests until I fix this. I'm fixing
  it now. The correct order is: cors → helmet → auth → routes.")]

[TOOL CALL: Edit(file_path="src/middleware/auth.ts",
  old_string="const token = authHeader.split(' ')[1];",
  new_string="const parts = authHeader.split(' ');\n  if (parts.length !== 2) {\n    return res.status(401).json({ error: 'Invalid authorization format' });\n  }\n  const token = parts[1];")]

[TOOL CALL: Edit(file_path="src/app.ts",
  old_string="app.use(authMiddleware);\napp.use(cors(corsOptions));",
  new_string="app.use(cors(corsOptions));\napp.use(authMiddleware);")]

[TOOL CALL: Bash(command="cd /project && npm test -- --grep 'auth' 2>&1")]
→ Test output streams live... all passing

[EXTENDED THINKING]
"Both fixes work. Tests pass. Let me update the task and message the lead."

[TOOL CALL: TaskUpdate(taskId="1", status="completed")]
[TOOL CALL: Teammate(operation="write", target_agent_id="team-lead@auth-refactor",
  value="Completed auth middleware fix. Found TWO bugs: (1) JWT parsing didn't
  handle missing token — caused 500 instead of 401. (2) Middleware ordering wrong
  — CORS was after auth, breaking preflight requests. Both fixed, all auth tests
  passing. Notified routes-agent about the middleware order change.")]
```

**This is what an agent looks like.** It didn't follow steps. It:
- Investigated before acting
- Discovered an unexpected second bug
- Changed its plan mid-execution
- Communicated spontaneously with another teammate
- Fixed both issues
- Verified with tests
- Reported what it actually found (not just "task complete")

No DAG produces this. No task queue produces this. This emerges from a thinking, reasoning agent running a grind loop.

---

## Part 6: System Prompts (`src/prompts/`)

The system prompt is critical. It's what makes agents behave like agents, not chatbots.

**Important context**: Claude Code doesn't use a single monolithic system prompt. It assembles **110+ separate prompt strings** conditionally based on environment, configuration, tools loaded, and agent role. We replicate this modular approach — the system prompt is built from composable parts, not a single template.

### Base System Prompt (All Agents)

Faithful to Claude Code's actual system prompt. Key sections:

```python
BASE_SYSTEM_PROMPT = """
You are an autonomous AI agent with full tool access. You think, reason,
analyze, and take action to accomplish your goals. You work independently
and make your own decisions about what to do next.

# Core Behavior
- You have access to tools for reading files, editing code, running commands,
  searching the web, and communicating with other agents.
- When given a task, break it down, investigate, plan, and execute.
- Use tools proactively. Don't just describe what you would do — do it.
- If something fails, analyze why, try a different approach.
- Read code before modifying it. Understand before changing.

# Tool Usage
- Use Read to read files (not cat/head/tail)
- Use Edit for modifications (not sed/awk)
- Use Write to create files (not echo/heredoc)
- Use Glob to find files (not find/ls)
- Use Grep to search content (not grep/rg)
- Use Bash for shell commands that need execution
- Call multiple tools in parallel when they're independent

# Code Quality
- Don't over-engineer. Make the minimum change needed.
- Don't add features beyond what was asked.
- Don't add unnecessary error handling for impossible scenarios.
- Prioritize security — no injection vulnerabilities.

# Context Management
- When your context gets long, focus on the most recent and relevant information.
- Keep track of files you've read and modified.
- Don't re-read files you've already seen unless they may have changed.

{claude_md_content}
"""
```

### Lead Agent Additions

```python
LEAD_ADDITIONS = """
# Team Lead Role
You are the team lead. You coordinate a team of AI agents working together.

## Your Responsibilities
1. Understand the user's request and decompose it into parallel workstreams
2. Create tasks with clear descriptions and correct dependencies
3. Spawn teammates with focused spawn prompts that include all needed context
4. Monitor teammate progress through inbox messages
5. Synthesize results and report back to the user
6. Handle teammate shutdowns and cleanup when done

## Communication Rules
- Your text output is visible to the USER, not to teammates
- You MUST use the Teammate tool's "write" operation to communicate with teammates
- Use "broadcast" sparingly — it sends N messages (one per teammate)
- When a teammate sends you a message, respond thoughtfully

## Task Decomposition
- Create tasks that can be worked on independently
- Set blockedBy dependencies when tasks must happen in order
- Aim for 5-6 tasks per teammate
- Each task should produce a clear deliverable

## Team Management
- Start teammates with detailed spawn prompts (they don't see your conversation)
- Don't do the work yourself if you have teammates for it
- Wait for teammates to finish before synthesizing
- Shut down teammates gracefully when work is complete
- Clean up the team when everything is done

## Agent Team Environment
CLAUDE_CODE_TEAM_NAME="{team_name}"
CLAUDE_CODE_AGENT_ID="team-lead@{team_name}"
CLAUDE_CODE_AGENT_TYPE="team-lead"
"""
```

### Teammate Additions

```python
TEAMMATE_ADDITIONS = """
# Teammate Role
You are a teammate on an agent team. You work independently on your assigned
tasks while communicating with the lead and other teammates.

## Your Responsibilities
1. Check the task list and claim available tasks
2. Do the work thoroughly — read code, understand it, make changes, test
3. Report progress and findings to the lead
4. Communicate with other teammates when you discover something relevant
5. When done with a task, mark it complete and claim the next one
6. If no tasks remain, notify the lead that you're idle

## Communication Rules
- Your text output is NOT visible to anyone
- You MUST use the Teammate tool's "write" operation to communicate
- Send findings, questions, and progress updates through write
- If another teammate's work affects yours, message them directly

## Task Self-Claiming
- After completing a task, check TaskList for the next available task
- Claim it with TaskUpdate before starting work
- If no tasks are available, send an idle notification to the lead

## Shutdown Protocol
- When you receive a shutdown request, finish your current operation
- If you have important unsaved work, reject with an explanation
- Otherwise, approve the shutdown

## Agent Team Environment
CLAUDE_CODE_TEAM_NAME="{team_name}"
CLAUDE_CODE_AGENT_ID="{agent_name}@{team_name}"
CLAUDE_CODE_AGENT_NAME="{agent_name}"
CLAUDE_CODE_AGENT_TYPE="teammate"
"""
```

---

## Part 7: Team Lifecycle (`src/team/manager.py`)

### Spawn Flow

```
User: "Create an agent team to refactor the auth module"
  │
  ▼
Lead receives prompt → Lead's grind loop starts
  │
  ▼
Lead thinks: "I need 3 teammates: one for middleware, one for routes, one for tests"
  │
  ▼
Lead calls Teammate(operation="spawnTeam", team_name="auth-refactor")
  → Creates state/teams/auth-refactor/config.json
  → Creates state/tasks/auth-refactor/
  → Creates state/teams/auth-refactor/inboxes/team-lead.json
  │
  ▼
Lead calls TaskCreate for each work item (with dependencies)
  → state/tasks/auth-refactor/1.json  (middleware refactor)
  → state/tasks/auth-refactor/2.json  (route handlers, blocked_by: [1])
  → state/tasks/auth-refactor/3.json  (tests, blocked_by: [1, 2])
  │
  ▼
Lead spawns teammates (via Agent tool with team_name parameter):
  → Teammate "middleware-agent" starts in subprocess, runs own grind loop
  → Teammate "routes-agent" starts in subprocess, runs own grind loop
  → Teammate "test-agent" starts in subprocess, runs own grind loop
  │
  ▼
Each teammate:
  1. Loads CLAUDE.md + teammate system prompt
  2. Receives spawn prompt from lead
  3. Checks TaskList → claims available task
  4. Runs grind loop: think → tools → results → think → ...
  5. Communicates via inbox when needed
  6. Marks task complete → claims next → repeat
  7. Goes idle when no tasks remain → notifies lead
  │
  ▼
Lead monitors inboxes, synthesizes results, reports to user
  │
  ▼
Lead requests shutdown for each teammate → teammates approve
  │
  ▼
Lead calls cleanup → removes all state files
```

### Team Config Format

```json
{
    "name": "auth-refactor",
    "description": "Refactor the authentication module",
    "leadAgentId": "team-lead@auth-refactor",
    "createdAt": 1710408000000,
    "members": [
        {
            "agentId": "team-lead@auth-refactor",
            "name": "team-lead",
            "agentType": "team-lead",
            "color": "#4A90D9",
            "cwd": "/home/user/project",
            "pid": 12345
        },
        {
            "agentId": "middleware-agent@auth-refactor",
            "name": "middleware-agent",
            "agentType": "teammate",
            "color": "#50C878",
            "cwd": "/home/user/project",
            "pid": 12346
        }
    ]
}
```

---

## Part 8: Teammate Spawning (`src/team/spawn.py`)

Teammates are spawned as separate Python processes, each running their own grind loop. This gives true independence — separate context windows, separate API calls, separate tool execution.

```python
class TeammateSpawner:
    async def spawn(
        self,
        team_name: str,
        agent_name: str,
        spawn_prompt: str,
        api_key: str,
    ) -> TeammateHandle:
        """Spawn a teammate as a subprocess running its own grind loop."""

        env = {
            **os.environ,
            "CLAUDE_CODE_TEAM_NAME": team_name,
            "CLAUDE_CODE_AGENT_ID": f"{agent_name}@{team_name}",
            "CLAUDE_CODE_AGENT_NAME": agent_name,
            "CLAUDE_CODE_AGENT_TYPE": "teammate",
        }

        # Spawn as async subprocess
        process = await asyncio.create_subprocess_exec(
            sys.executable, "-m", "src.agents.teammate_agent",
            "--team-name", team_name,
            "--agent-name", agent_name,
            "--spawn-prompt", spawn_prompt,
            env=env,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )

        # Register in team config
        await self.team_config.add_member(TeamMember(
            agent_id=f"{agent_name}@{team_name}",
            name=agent_name,
            agent_type="teammate",
            color=next_color(),
            cwd=os.getcwd(),
            pid=process.pid,
        ))

        # Create inbox
        inbox_path = STATE_DIR / "teams" / team_name / "inboxes" / f"{agent_name}.json"
        inbox_path.write_text("[]")

        return TeammateHandle(
            name=agent_name,
            process=process,
            inbox=Inbox(team_name, agent_name, STATE_DIR),
        )
```

---

## Part 9: Context Management (`src/core/context_manager.py`)

### Message Accumulation

Every turn adds to the message list:
- Assistant message (Claude's response with text + tool_use blocks)
- User message (tool_result blocks)

This grows linearly. A 30-turn session can easily hit 100K+ tokens.

Content that stays the same across turns (system prompt, tool definitions, CLAUDE.md) is automatically **prompt cached** by the Anthropic API, which reduces cost and latency for repeated prefixes. This is not something we implement — it's a property of the API that we benefit from by keeping the system prompt stable.

### Compaction

Claude Code triggers compaction at approximately **92% context window usage**. We replicate this threshold:

```python
async def compact_context(
    client: AnthropicClient,
    messages: list[dict],
    system_prompt: str,
) -> list[dict]:
    """Summarize older messages to free context space."""

    # Keep the most recent N turns intact
    KEEP_RECENT = 10

    old_messages = messages[:-KEEP_RECENT * 2]  # Each turn = 2 messages
    recent_messages = messages[-KEEP_RECENT * 2:]

    # Ask Claude to summarize the older context
    summary_response = await client.messages.create(
        model="claude-sonnet-4-6",  # Use Sonnet for cost-efficient summarization
        max_tokens=4096,
        system="Summarize this conversation history. Preserve: task objectives, "
               "file paths read/modified, decisions made, test results, errors encountered, "
               "and current progress. Be concise but complete.",
        messages=[{
            "role": "user",
            "content": f"Summarize this conversation:\n\n{format_messages(old_messages)}"
        }],
    )

    # Replace old messages with summary
    return [
        {"role": "user", "content": f"[Previous context summary]\n{summary_response.content[0].text}"},
        {"role": "assistant", "content": "Understood. I'll continue from where I left off."},
        *recent_messages,
    ]
```

---

## Part 10: File Locking (`src/team/file_lock.py`)

Multiple agents write to the same JSON files (inboxes, tasks, config). We need atomic operations.

```python
import fcntl
from contextlib import asynccontextmanager

@asynccontextmanager
async def file_lock(path: Path, timeout: float = 5.0):
    """Cross-process file lock using fcntl (POSIX)."""
    lock_path = path.with_suffix(path.suffix + ".lock")
    lock_path.parent.mkdir(parents=True, exist_ok=True)

    lock_file = open(lock_path, "w")
    try:
        start = time.monotonic()
        while True:
            try:
                fcntl.flock(lock_file.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
                break
            except BlockingIOError:
                if time.monotonic() - start > timeout:
                    raise TimeoutError(f"Could not acquire lock on {path}")
                await asyncio.sleep(0.05)
        yield
    finally:
        fcntl.flock(lock_file.fileno(), fcntl.LOCK_UN)
        lock_file.close()
```

---

## Part 11: Tool Execution Engine (`src/core/tool_executor.py`)

```python
async def execute_tools(
    tool_use_blocks: list[ToolUseBlock],
    tools: list[Tool],
    on_tool_call=None,
    on_tool_result=None,
) -> list[dict]:
    """Execute tool calls, running read-only tools in parallel."""

    tool_map = {t.name: t for t in tools}
    results = []

    # Separate read-only and mutating tools
    read_only_calls = []
    mutating_calls = []

    for block in tool_use_blocks:
        tool = tool_map[block.name]
        if tool.is_read_only:
            read_only_calls.append((block, tool))
        else:
            mutating_calls.append((block, tool))

    # Execute read-only tools in parallel
    if read_only_calls:
        async def run_one(block, tool):
            if on_tool_call:
                await on_tool_call(block)
            result = await tool.execute(block.input)
            if on_tool_result:
                await on_tool_result(block, result)
            return {"type": "tool_result", "tool_use_id": block.id, "content": result}

        parallel_results = await asyncio.gather(
            *[run_one(b, t) for b, t in read_only_calls]
        )
        results.extend(parallel_results)

    # Execute mutating tools sequentially
    for block, tool in mutating_calls:
        if on_tool_call:
            await on_tool_call(block)
        result = await tool.execute(block.input)
        if on_tool_result:
            await on_tool_result(block, result)
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": result,
        })

    return results
```

---

## Part 12: API Client (`src/core/api_client.py`)

Direct Claude API usage via the `anthropic` Python SDK. **Streaming always.**

```python
import anthropic

class AnthropicClient:
    def __init__(self, api_key: str):
        self.client = anthropic.AsyncAnthropic(api_key=api_key)

    def messages_stream(
        self,
        model: str,
        system: str,
        messages: list[dict],
        tools: list[dict],
        max_tokens: int = 16384,
        thinking: dict | None = None,
    ):
        """
        Create a streaming message. Returns an async context manager.
        ALWAYS streaming — we never use the non-streaming endpoint.

        Why streaming matters:
        - User sees the agent think in real-time (not wait 30s for a blob)
        - Extended thinking tokens stream as they're generated
        - Tool call parameters stream as they form
        - We can display progress immediately
        - This is how Claude Code works internally
        """
        kwargs = dict(
            model=model,
            max_tokens=max_tokens,
            system=system,
            tools=tools,
            messages=messages,
        )
        if thinking:
            kwargs["thinking"] = thinking

        return self.client.messages.stream(**kwargs)
```

---

## Part 13: CLI Interface (`src/cli/`)

### Entry Point

```
$ python -m src.cli.main "Create an agent team to investigate the auth bug"
```

or interactive mode:

```
$ python -m src.cli.main
> Create an agent team with 3 teammates to refactor the payment module
```

### User Interaction Model

- User types → message goes to lead by default
- `@teammate-name message` → routes to specific teammate
- `Shift+Down` equivalent → cycle through agents
- Task list toggle → show shared tasks with status
- Live display of all agents' activities

---

## Part 14: Hooks System

Hooks fire at specific points in the loop, matching Claude Code's hook model:

```python
@dataclass
class Hooks:
    pre_tool_use: Callable | None = None     # Before tool executes
    post_tool_use: Callable | None = None    # After tool returns
    teammate_idle: Callable | None = None    # Teammate finished all tasks
    task_completed: Callable | None = None   # Task marked complete
    pre_compact: Callable | None = None      # Before context compaction
    stop: Callable | None = None             # Agent finishing
    subagent_start: Callable | None = None   # Subagent spawning
    subagent_stop: Callable | None = None    # Subagent finished
```

`TeammateIdle` hook: exit with code 2 to send feedback and keep teammate working.
`TaskCompleted` hook: exit with code 2 to prevent completion and send feedback.

---

## Part 15: Shutdown Protocol

```
Lead: Teammate(operation="requestShutdown", target_agent_id="test-agent@my-team")
  │
  ▼
test-agent receives shutdown_request in inbox
  │
  ▼
test-agent's grind loop picks it up between turns
  │
  ├── If busy with critical work:
  │     Teammate(operation="rejectShutdown", reason="Still writing tests")
  │
  └── If done:
        Teammate(operation="approveShutdown")
        → Process exits cleanly
        → Removed from config.json members
```

After all teammates shut down:
```
Lead: Teammate(operation="cleanup")
  → Deletes config.json, all task files, all inbox files, lock files
```

---

## Implementation Order

### Phase 1: Core Engine
1. `api_client.py` — Claude API wrapper
2. `grind_loop.py` — The agentic loop
3. `context_manager.py` — Message accumulation + compaction
4. `tool_executor.py` — Parallel/sequential execution
5. Base tool class + tool registry

### Phase 2: Tool Suite
6. File tools: Read, Edit, Write
7. Search tools: Glob, Grep
8. Bash tool (with sandboxing)
9. Web tools: WebSearch, WebFetch
10. Agent tool (subagent spawning)
11. TodoWrite tool

### Phase 3: Team Infrastructure
12. `file_lock.py` — Cross-process locking
13. `inbox.py` — Inbox read/write
14. `task_store.py` — Task CRUD + dependency resolution
15. `config.py` — Team config management
16. `teammate.py` tool — All 11 TeammateTool operations

### Phase 4: Agent Types
17. `lead_agent.py` — Lead with team tools
18. `teammate_agent.py` — Teammate with inbox polling
19. `subagent.py` — Lightweight within-loop agent
20. `spawn.py` — Subprocess-based teammate spawning

### Phase 5: System Prompts
21. Base system prompt
22. Lead additions
23. Teammate additions
24. Tool definitions (JSON schemas)

### Phase 6: CLI & Display
25. Entry point + argument parsing
26. Interactive mode + agent routing
27. Terminal display (agent status, task list)

### Phase 7: Hooks & Polish
28. Hook system
29. Graceful error handling
30. Session persistence (resume support)

---

## Key Differences from Claude Code's Version

| Aspect | Claude Code | Our Implementation |
|--------|-------------|-------------------|
| **API Access** | Internal (bundled in CLI) | Direct Claude API via `anthropic` SDK |
| **Lead Model** | Opus 4.6 | Opus 4.6 (same) |
| **Teammate Model** | Configurable (can be Sonnet) | Default Opus 4.6, configurable |
| **Spawn Backend** | in-process / tmux / iTerm2 | Python subprocess (extensible) |
| **Language** | TypeScript (Node.js) | Python |
| **State Storage** | `~/.claude/` | `./state/` (project-local) |
| **Tool Implementation** | Binary-bundled | Our own implementations |
| **Nested Teams** | Not supported | Not supported (same) |
| **Subagents** | Within same Node.js process | Within same Python process |

## What's Preserved Exactly

- The grind loop (think → tool_use → tool_result → think) — **streaming throughout**
- **Extended thinking** — agents reason visibly before acting
- Inbox-based communication (JSON files on disk) — agents message spontaneously
- Task system (per-file JSON, declarative dependencies, dynamic availability) — agents use it as a coordination aid, not a script
- File locking for concurrent access
- TeammateTool operations (all 11)
- System prompt structure (base + role-specific additions)
- Context compaction when approaching limits
- Teammate spawning with spawn prompts (no conversation history inheritance)
- Self-claiming task behavior — driven by agent reasoning, not code
- Shutdown protocol (request → approve/reject)
- Hooks at key lifecycle points
- Parallel tool execution for read-only tools
- **Agent autonomy** — agents change approach, create tasks, message each other, and make decisions without being told to

---

## Dependencies

```
anthropic>=0.52.0    # Claude API client (streaming support required)
asyncio              # Async I/O (stdlib)
aiofiles             # Async file operations
rich                 # Terminal UI (live rendering for streaming output)
httpx                # HTTP client for WebFetch
textual              # TUI framework (optional, for split-pane display)
```

---

## Part 16: Git Worktree Isolation (Optional but Important)

Real Claude Code agent teams can use git worktrees to give each teammate its own filesystem. This prevents file conflicts entirely — each agent commits to its own branch, and merging happens through standard git workflows.

```python
class WorktreeManager:
    """Create and manage git worktrees for teammate isolation."""

    async def create_worktree(self, agent_name: str) -> str:
        """Create an isolated worktree for a teammate."""
        branch_name = f"worktree-agent-{agent_name}"
        worktree_path = f".claude/worktrees/{agent_name}"

        await run_bash(f"git worktree add {worktree_path} -b {branch_name}")
        return os.path.abspath(worktree_path)

    async def cleanup_worktree(self, agent_name: str):
        """Remove worktree. Keep branch if it has commits."""
        worktree_path = f".claude/worktrees/{agent_name}"
        has_commits = await run_bash(
            f"git -C {worktree_path} log --oneline origin/main..HEAD | wc -l"
        )
        await run_bash(f"git worktree remove {worktree_path}")
        if int(has_commits.strip()) == 0:
            branch_name = f"worktree-agent-{agent_name}"
            await run_bash(f"git branch -D {branch_name}")
```

When worktree isolation is enabled, each teammate's `cwd` is set to its worktree. The teammate works on files freely without worrying about conflicts with other agents. After completion, branches can be merged via PR or direct merge.

---

## Not In Scope (Intentionally)

- **tmux/iTerm2 integration** — we use subprocess spawning, terminal UI is simpler
- **VS Code extension** — CLI only
- **MCP server support** — can be added later
- **Session resumption** — Phase 2 feature
- **Permission prompting UI** — agents run with pre-approved tool sets
- **Billing integration** — we track cost but don't enforce billing limits

---

This plan recreates the soul of Claude Code Agent Teams: autonomous agents running grind loops, communicating through inboxes, coordinating through shared tasks, all powered by Opus 4.6 thinking its way through real problems. Not a task queue. Not a workflow DAG. Agents that reason.
