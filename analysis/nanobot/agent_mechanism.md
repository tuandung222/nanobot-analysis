# NanoBot - Agent Mechanism Analysis

> **Repository:** [HKUDS/NanoBot](https://github.com/HKUDS/NanoBot)
> **Version analyzed:** v0.1.3.post4
> **Focus:** ReAct Loop, Function Calling, Tool Use, Planning, Prompt Engineering

---

## 1. Control Loop Architecture: Simplified ReAct Pattern

### 1.1 Overview

NanoBot implements a **streamlined ReAct (Reason + Act) loop** — but notably, it does **not** include an explicit reasoning/observation phase. Instead, it relies entirely on the LLM's **native function calling** (tool_calls) capability to alternate between thinking and acting. This is a **tool-call-driven loop** rather than a classical ReAct pattern with structured `Thought → Action → Observation` prompts.

### 1.2 Core Loop: `AgentLoop._process_message()`

**File:** `nanobot/agent/loop.py:143-247`

The control loop is deceptively simple:

```
┌─────────────────────────────────────────────────────┐
│                  AgentLoop.run()                     │
│                                                      │
│   while running:                                     │
│     msg = await bus.consume_inbound()                │
│     response = await _process_message(msg)           │
│     await bus.publish_outbound(response)             │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│            _process_message(msg)                     │
│                                                      │
│   1. Get/create session                              │
│   2. Set tool contexts (channel, chat_id)            │
│   3. Build messages (system + history + user msg)    │
│   4. ┌─── Agent Loop (max 20 iterations) ────────┐  │
│      │                                            │  │
│      │  response = await provider.chat(           │  │
│      │      messages, tools, model                │  │
│      │  )                                         │  │
│      │                                            │  │
│      │  if response.has_tool_calls:               │  │
│      │    ├─ Add assistant message + tool_calls   │  │
│      │    ├─ Execute each tool call               │  │
│      │    ├─ Add tool results to messages         │  │
│      │    └─ CONTINUE LOOP                        │  │
│      │  else:                                     │  │
│      │    └─ final_content = response.content     │  │
│      │       BREAK                                │  │
│      │                                            │  │
│      └────────────────────────────────────────────┘  │
│   5. Save user + assistant messages to session       │
│   6. Return OutboundMessage                          │
└─────────────────────────────────────────────────────┘
```

### 1.3 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Loop type | Iterative tool-call loop | Simple, relies on LLM's built-in tool selection |
| Max iterations | 20 (configurable) | Prevents infinite loops |
| Reasoning separation | None (implicit in LLM) | Minimizes code complexity |
| Parallel tool calls | Sequential execution | Simplicity over performance |
| Error handling | Return error as text to LLM | Let LLM decide retry strategy |
| Loop termination | When LLM returns no tool_calls | Natural completion signal |

### 1.4 Comparison with Classical ReAct

```
Classical ReAct:                    NanoBot's Approach:
┌──────────────────┐               ┌──────────────────┐
│ Thought: I need  │               │ [LLM internally  │
│ to search for... │               │  decides to call  │
├──────────────────┤               │  a tool]          │
│ Action: search   │               ├──────────────────┤
│ Action Input:... │               │ tool_call:        │
├──────────────────┤               │   web_search(q)   │
│ Observation:     │               ├──────────────────┤
│ <search results> │               │ tool result:      │
├──────────────────┤               │ <search results>  │
│ Thought: Based   │               ├──────────────────┤
│ on results...    │               │ [LLM continues    │
├──────────────────┤               │  with more calls  │
│ Final Answer:... │               │  or final answer] │
└──────────────────┘               └──────────────────┘

Explicit prompt template                Native function calling
More control, more tokens               Less code, relies on LLM
Works with any LLM                      Requires tool_calls support
```

**NanoBot does NOT use:**
- Explicit `Thought:` / `Action:` / `Observation:` prompt templates
- Chain-of-thought forcing
- Self-reflection or self-critique loops
- Structured output parsing (no regex extraction)

---

## 2. Function Calling / Tool Use System

### 2.1 Tool Architecture

The tool system follows a clean **Abstract Base Class → Registry → Execution** pattern:

```
Tool (ABC)                          ToolRegistry
├── name: str (abstract)            ├── _tools: dict[str, Tool]
├── description: str (abstract)     ├── register(tool)
├── parameters: dict (abstract)     ├── get_definitions() → OpenAI format
├── execute(**kwargs) → str         ├── execute(name, params) → str
├── validate_params(params)         └── tool_names: list[str]
└── to_schema() → OpenAI format
```

**File:** `nanobot/agent/tools/base.py`

### 2.2 Built-in Tools (10 tools)

| Tool | Class | File | Description |
|------|-------|------|-------------|
| `read_file` | `ReadFileTool` | `tools/filesystem.py` | Read file contents |
| `write_file` | `WriteFileTool` | `tools/filesystem.py` | Write content to file |
| `edit_file` | `EditFileTool` | `tools/filesystem.py` | Find-and-replace text editing |
| `list_dir` | `ListDirTool` | `tools/filesystem.py` | List directory contents |
| `exec` | `ExecTool` | `tools/shell.py` | Execute shell commands (with safety guards) |
| `web_search` | `WebSearchTool` | `tools/web.py` | Brave Search API |
| `web_fetch` | `WebFetchTool` | `tools/web.py` | Fetch and extract web content |
| `message` | `MessageTool` | `tools/message.py` | Send messages to chat channels |
| `spawn` | `SpawnTool` | `tools/spawn.py` | Spawn background subagents |
| `cron` | `CronTool` | `tools/cron.py` | Schedule reminders/recurring tasks |

### 2.3 Tool Schema Format

Tools are exposed to the LLM in **OpenAI function calling format**:

```python
# Tool.to_schema() output
{
    "type": "function",
    "function": {
        "name": "exec",
        "description": "Execute a shell command and return its output.",
        "parameters": {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The shell command to execute"
                },
                "working_dir": {
                    "type": "string",
                    "description": "Optional working directory"
                }
            },
            "required": ["command"]
        }
    }
}
```

### 2.4 Tool Execution Pipeline

```
LLM Response
    │
    ├── response.tool_calls = [ToolCallRequest(id, name, arguments)]
    │
    ▼
ToolRegistry.execute(name, params)
    │
    ├── 1. Find tool by name
    ├── 2. Validate parameters against JSON schema
    ├── 3. Call tool.execute(**params)
    ├── 4. Return result as string
    └── 5. On error: return "Error: ..." string (not exception)
    │
    ▼
Result added to messages as:
{
    "role": "tool",
    "tool_call_id": "<id>",
    "name": "<tool_name>",
    "content": "<result_string>"
}
```

### 2.5 Parameter Validation System

The `Tool` base class includes a **recursive JSON Schema validator** (`validate_params()`) that checks:
- Type matching (string, integer, number, boolean, array, object)
- Enum constraints
- Numeric range (minimum, maximum)
- String length (minLength, maxLength)
- Required fields
- Nested object/array validation

```python
# From base.py — type mapping
_TYPE_MAP = {
    "string": str,
    "integer": int,
    "number": (int, float),
    "boolean": bool,
    "array": list,
    "object": dict,
}
```

### 2.6 Safety Guards (Shell Execution)

The `ExecTool` includes a multi-layer safety system (`tools/shell.py:111-141`):

**Deny patterns** (blocked by default):
```python
deny_patterns = [
    r"\brm\s+-[rf]{1,2}\b",          # rm -r, rm -rf
    r"\bdel\s+/[fq]\b",              # Windows del /f
    r"\brmdir\s+/s\b",               # Windows rmdir /s
    r"\b(format|mkfs|diskpart)\b",   # Disk operations
    r"\bdd\s+if=",                   # dd
    r">\s*/dev/sd",                  # Direct disk write
    r"\b(shutdown|reboot|poweroff)\b", # System power
    r":\(\)\s*\{.*\};\s*:",          # Fork bomb
]
```

**Workspace restriction mode:**
- When `restrict_to_workspace = True`, all file/shell operations are sandboxed
- Path traversal (`../`) is blocked
- Absolute paths outside workspace are rejected

---

## 3. Planning & Task Decomposition

### 3.1 No Explicit Planner

NanoBot has **no dedicated Planner module**. There is no:
- Task decomposition engine
- Goal-state tracking
- Plan verification/refinement
- Hierarchical planning
- Plan-and-execute separation

Task planning is **entirely implicit** — the LLM naturally breaks down complex requests into tool calls within its reasoning. The `max_iterations=20` limit acts as the only structural constraint on plan execution depth.

### 3.2 Subagent Delegation (Pseudo-Planning)

The closest thing to task decomposition is the **Subagent system** (`nanobot/agent/subagent.py`), which allows the main agent to delegate independent tasks to background workers:

```
Main Agent
    │
    ├── User: "Research topic X and write a report"
    │
    ├── [LLM decides to spawn subagent]
    │
    ├── spawn(task="Research topic X and write report to report.md")
    │       │
    │       └── SubagentManager.spawn()
    │              │
    │              ├── Creates isolated tool set (no message, no spawn)
    │              ├── Builds focused system prompt
    │              ├── Runs independent agent loop (max 15 iterations)
    │              └── Announces result back to main agent via MessageBus
    │
    └── Main agent continues processing other requests
```

**Subagent constraints:**
- Cannot send messages directly to users
- Cannot spawn further subagents (no recursion)
- Has no access to main agent's conversation history
- Limited to 15 iterations (vs. 20 for main agent)
- Shares the same LLM provider and model

---

## 4. Prompt Engineering System

### 4.1 System Prompt Assembly

The system prompt is built dynamically by `ContextBuilder.build_system_prompt()` (`context.py:28-71`):

```
System Prompt = ─────────────────────────────────────────
│ 1. Core Identity                                       │
│    - Role: "You are nanobot, a helpful AI assistant"   │
│    - Current time, runtime, workspace path             │
│    - Available capabilities list                       │
│    - Behavioral instructions                           │
├────────────────────────────────────────────────────────┤
│ 2. Bootstrap Files (workspace/*.md)                    │
│    - AGENTS.md  → Agent behavioral guidelines          │
│    - SOUL.md    → Personality/values                   │
│    - USER.md    → User preferences                     │
│    - TOOLS.md   → Tool documentation                   │
│    - IDENTITY.md → Custom identity                     │
├────────────────────────────────────────────────────────┤
│ 3. Memory Context                                      │
│    - Long-term memory (MEMORY.md)                      │
│    - Today's notes (YYYY-MM-DD.md)                     │
├────────────────────────────────────────────────────────┤
│ 4. Active Skills (always-loaded)                       │
│    - Full SKILL.md content for auto-loaded skills      │
├────────────────────────────────────────────────────────┤
│ 5. Skills Summary (progressive loading)                │
│    - XML list of available skills with descriptions    │
│    - Agent uses read_file to load on demand            │
├────────────────────────────────────────────────────────┤
│ 6. Session Context                                     │
│    - Current channel and chat_id                       │
└────────────────────────────────────────────────────────┘
```

### 4.2 Progressive Skill Loading

Skills use a two-tier loading strategy:

**Tier 1: Always-loaded** — Skills with `always: true` in frontmatter metadata are fully included in every system prompt. This minimizes token usage for skills that are always needed.

**Tier 2: On-demand** — All other skills are summarized in an XML block. The agent can read full skill content using the `read_file` tool when needed:

```xml
<skills>
  <skill available="true">
    <name>github</name>
    <description>Interact with GitHub using the gh CLI</description>
    <location>/path/to/skills/github/SKILL.md</location>
  </skill>
  <skill available="false">
    <name>tmux</name>
    <description>Remote-control tmux sessions</description>
    <location>/path/to/skills/tmux/SKILL.md</location>
    <requires>CLI: tmux</requires>
  </skill>
</skills>
```

### 4.3 Skill Metadata & Requirements

Skills support dependency checking via frontmatter metadata:

```yaml
---
name: github
description: "Interact with GitHub using the gh CLI"
metadata: {"nanobot":{"emoji":"🐙","requires":{"bins":["gh"]}}}
---
```

The `SkillsLoader` checks:
- **Binary requirements** (`bins`): Verifies CLI tools are installed via `shutil.which()`
- **Environment variables** (`env`): Checks if required env vars are set
- Skills with unmet requirements are marked `available="false"` but still listed

### 4.4 Memory System

The memory system (`nanobot/agent/memory.py`) provides two persistence layers:

```
MemoryStore
├── Long-term Memory: workspace/memory/MEMORY.md
│   └── Persistent facts, preferences, important notes
├── Daily Notes: workspace/memory/YYYY-MM-DD.md
│   └── Day-specific information, auto-partitioned by date
└── get_memory_context()
    └── Combines long-term + today's notes for system prompt
```

**Key characteristics:**
- File-based storage (plain Markdown)
- No vector database, no embeddings, no semantic search
- Agent writes to memory via `write_file`/`edit_file` tools
- Last 7 days of memories accessible via `get_recent_memories()`

### 4.5 Session Management

Conversation history is managed by `SessionManager` (`nanobot/session/manager.py`):

- Sessions stored as **JSONL files** in `~/.nanobot/sessions/`
- Session key format: `{channel}:{chat_id}` (e.g., `telegram:12345678`)
- History window: last **50 messages** sent to LLM
- Messages include timestamps for temporal context
- In-memory cache for active sessions

---

## 5. Channel Communication Architecture

### 5.1 Message Bus Pattern

NanoBot uses an **async message bus** (`nanobot/bus/queue.py`) to decouple channels from agent logic:

```
                    ┌─────────────┐
    Telegram ──────▶│             │──────▶ Agent Loop
    Discord  ──────▶│  MessageBus │       (processes one
    WhatsApp ──────▶│  (asyncio   │        at a time)
    Feishu   ──────▶│   Queue)    │
    CLI      ──────▶│             │
    Cron     ──────▶│  Inbound    │
    Subagent ──────▶│             │
                    │             │
                    │  Outbound   │◀────── Agent Response
                    │             │
                    └──────┬──────┘
                           │
                    ChannelManager
                    .dispatch_outbound()
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Telegram     Discord     WhatsApp
```

### 5.2 Channel Abstraction

All channels extend `BaseChannel` (`nanobot/channels/base.py`):

```python
class BaseChannel(ABC):
    name: str = "base"
    
    async def start() -> None       # Connect and listen
    async def stop() -> None        # Disconnect
    async def send(msg) -> None     # Send outbound message
    def is_allowed(sender_id) -> bool  # Permission check
    async def _handle_message(...)  # Forward to bus
```

### 5.3 Media Handling Flow

```
Channel receives media (photo/voice/document)
    │
    ├── Download to ~/.nanobot/media/{file_id}.{ext}
    │
    ├── If voice/audio:
    │   └── GroqTranscriptionProvider.transcribe(file_path)
    │       └── Append "[transcription: ...]" to content
    │
    ├── If image:
    │   └── Store path in InboundMessage.media list
    │       └── ContextBuilder base64-encodes for LLM
    │
    └── Forward to MessageBus as InboundMessage
```

---

## 6. Heartbeat & Proactive Behavior

### 6.1 Heartbeat Service

The `HeartbeatService` (`nanobot/heartbeat/service.py`) enables **proactive agent behavior**:

```
Every 30 minutes:
    │
    ├── Read HEARTBEAT.md from workspace
    │
    ├── If has actionable content (not just headers/comments):
    │   └── Send HEARTBEAT_PROMPT to agent:
    │       "Read HEARTBEAT.md and follow any instructions"
    │
    ├── Agent processes tasks listed in HEARTBEAT.md
    │
    └── If agent replies "HEARTBEAT_OK":
        └── No action needed, skip
```

This enables recurring behaviors like:
- Periodic status checks
- Scheduled data collection
- Autonomous task execution

### 6.2 Cron Service

The `CronService` (`nanobot/cron/service.py`) provides fine-grained scheduling:

- **Three schedule types:** `at` (one-time), `every` (interval), `cron` (expression)
- **Persistent storage:** Jobs saved to `~/.nanobot/cron/jobs.json`
- **Agent integration:** Each cron job runs through `agent.process_direct()`
- **Delivery:** Results can be sent to specific channels/users

---

## 7. Comparison with Other Agent Frameworks

| Feature | NanoBot | LangChain Agent | AutoGen | CrewAI |
|---------|---------|-----------------|---------|--------|
| **Codebase size** | ~3,400 LOC | ~500K LOC | ~100K LOC | ~50K LOC |
| **Control loop** | Simple tool-call loop | Configurable (ReAct, Plan-Execute, etc.) | Multi-agent conversation | Role-based task delegation |
| **Planning** | None (implicit in LLM) | Optional planner | Conversation-driven | Task decomposition |
| **Tool system** | Custom ABC-based | Structured tools | Function decorators | Tool class |
| **Memory** | File-based (Markdown) | Vector store integration | In-memory + persistence | Various backends |
| **Multi-agent** | Subagent spawn | Agent chains | Multi-agent conversation | Crew of agents |
| **Prompt engineering** | Bootstrap files + skills | Prompt templates | System messages | Role prompts |
| **Channels** | 4 chat platforms | None built-in | None built-in | None built-in |

---

## 8. Key Source Files for Agent Mechanism

| File | Purpose | Lines |
|------|---------|-------|
| `nanobot/agent/loop.py` | Core agent loop (message processing, tool execution) | 375 |
| `nanobot/agent/context.py` | System prompt builder, multimodal content formatting | 230 |
| `nanobot/agent/memory.py` | File-based persistent memory system | 110 |
| `nanobot/agent/skills.py` | Skill loading, progressive loading, requirements checking | 229 |
| `nanobot/agent/subagent.py` | Background subagent spawning and management | 245 |
| `nanobot/agent/tools/base.py` | Tool abstract base class with validation | 103 |
| `nanobot/agent/tools/registry.py` | Dynamic tool registration and execution | 74 |
| `nanobot/agent/tools/shell.py` | Shell execution with safety guards | 142 |
| `nanobot/agent/tools/filesystem.py` | File read/write/edit/list operations | 212 |
| `nanobot/agent/tools/web.py` | Web search (Brave) and content fetch (Readability) | 164 |
| `nanobot/agent/tools/message.py` | Cross-channel message sending | 87 |
| `nanobot/agent/tools/spawn.py` | Subagent spawning interface | 66 |
| `nanobot/agent/tools/cron.py` | Cron job scheduling tool | 115 |

---

## 9. Summary

NanoBot's agent mechanism is characterized by its **radical simplicity**:

1. **No framework overhead** — direct tool-call loop, no intermediate abstractions
2. **LLM-native planning** — relies on the model's inherent reasoning capability
3. **File-based everything** — memory, skills, sessions, configuration
4. **Clean separation** — channels ↔ bus ↔ agent ↔ tools
5. **Extensible by design** — add tools via ABC, skills via Markdown files

The tradeoff is clear: NanoBot sacrifices advanced planning, structured reasoning, and sophisticated memory retrieval for **readability, simplicity, and a tiny footprint**. This makes it an excellent **teaching/research artifact** for understanding agent fundamentals, but it would need significant enhancement for production-grade autonomous task execution.
