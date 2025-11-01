# Python Agent SDK Documentation

**Source:** https://docs.claude.com/en/docs/claude-code/sdk/sdk-python
**Last Updated:** 2025-11-01
**SDK Version:** claude-agent-sdk (formerly claude-code-sdk)

---

## Installation

```bash
pip install claude-agent-sdk
```

## Core Concepts: `query()` vs `ClaudeSDKClient`

The SDK provides two interaction patterns:

### `query()` - Single Session
Creates a new session for each interaction. Best for one-off tasks without conversation history.

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    options = ClaudeAgentOptions(
        system_prompt="You are an expert Python developer",
        permission_mode='acceptEdits',
        cwd="/home/user/project"
    )

    async for message in query(
        prompt="Create a Python web server",
        options=options
    ):
        print(message)

asyncio.run(main())
```

### `ClaudeSDKClient` - Continuous Conversation
Maintains session state across multiple exchanges. Claude remembers previous context.

```python
from claude_agent_sdk import ClaudeSDKClient

async def main():
    async with ClaudeSDKClient() as client:
        # First question
        await client.query("What's the capital of France?")
        async for message in client.receive_response():
            print(message)

        # Follow-up - Claude remembers context
        await client.query("What's the population of that city?")
        async for message in client.receive_response():
            print(message)

asyncio.run(main())
```

## Configuration: ClaudeAgentOptions

Dataclass for query configuration:

```python
@dataclass
class ClaudeAgentOptions:
    allowed_tools: list[str] = []
    system_prompt: str | SystemPromptPreset | None = None
    mcp_servers: dict[str, McpServerConfig] | str | Path = {}
    permission_mode: PermissionMode | None = None
    continue_conversation: bool = False
    resume: str | None = None
    max_turns: int | None = None
    disallowed_tools: list[str] = []
    model: str | None = None
    cwd: str | Path | None = None
    can_use_tool: CanUseTool | None = None
    hooks: dict[HookEvent, list[HookMatcher]] | None = None
    setting_sources: list[SettingSource] | None = None
    agents: dict[str, AgentDefinition] | None = None
```

### Permission Modes

```python
PermissionMode = Literal[
    "default",           # Standard behavior
    "acceptEdits",       # Auto-accept file edits
    "plan",              # Planning mode - no execution
    "bypassPermissions"  # Bypass all checks
]
```

### Setting Sources

Control which filesystem settings to load:

```python
SettingSource = Literal["user", "project", "local"]

# Load all settings
options = ClaudeAgentOptions(
    setting_sources=["user", "project", "local"]
)

# Load only project settings
options = ClaudeAgentOptions(
    setting_sources=["project"]
)
```

## Custom Tools with Decorator

Define MCP tools using the `@tool` decorator:

```python
from claude_agent_sdk import tool, create_sdk_mcp_server
from typing import Any

@tool("add", "Add two numbers", {"a": float, "b": float})
async def add(args):
    return {
        "content": [{
            "type": "text",
            "text": f"Sum: {args['a'] + args['b']}"
        }]
    }

@tool("multiply", "Multiply two numbers", {"a": float, "b": float})
async def multiply(args):
    return {
        "content": [{
            "type": "text",
            "text": f"Product: {args['a'] * args['b']}"
        }]
    }

calculator = create_sdk_mcp_server(
    name="calculator",
    version="2.0.0",
    tools=[add, multiply]
)

options = ClaudeAgentOptions(
    mcp_servers={"calc": calculator},
    allowed_tools=["mcp__calc__add", "mcp__calc__multiply"]
)
```

## Message Types

```python
@dataclass
class UserMessage:
    content: str | list[ContentBlock]

@dataclass
class AssistantMessage:
    content: list[ContentBlock]
    model: str

@dataclass
class ResultMessage:
    subtype: str
    duration_ms: int
    is_error: bool
    num_turns: int
    session_id: str
    total_cost_usd: float | None = None
```

### Content Blocks

```python
@dataclass
class TextBlock:
    text: str

@dataclass
class ThinkingBlock:
    thinking: str
    signature: str

@antml:parameter>
class ToolUseBlock:
    id: str
    name: str
    input: dict[str, Any]

@dataclass
class ToolResultBlock:
    tool_use_id: str
    content: str | list[dict[str, Any]] | None = None
    is_error: bool | None = None
```

## Error Handling

```python
from claude_agent_sdk import (
    CLINotFoundError,
    ProcessError,
    CLIJSONDecodeError
)

try:
    async for message in query(prompt="Hello"):
        print(message)
except CLINotFoundError:
    print("Claude Code CLI not installed")
except ProcessError as e:
    print(f"Exit code: {e.exit_code}")
except CLIJSONDecodeError as e:
    print(f"Parse error: {e}")
```

## Hooks for Behavior Control

Intercept and modify events:

```python
from claude_agent_sdk import HookMatcher, HookContext
from typing import Any

async def validate_bash(
    input_data: dict[str, Any],
    tool_use_id: str | None,
    context: HookContext
) -> dict[str, Any]:
    if "rm -rf /" in str(input_data.get('tool_input', {})):
        return {
            'hookSpecificOutput': {
                'permissionDecision': 'deny',
                'permissionDecisionReason': 'Dangerous command'
            }
        }
    return {}

options = ClaudeAgentOptions(
    hooks={
        'PreToolUse': [
            HookMatcher(matcher='Bash', hooks=[validate_bash])
        ]
    }
)
```

## System Prompt Presets

Use Claude Code's built-in system prompt:

```python
options = ClaudeAgentOptions(
    system_prompt={
        "type": "preset",
        "preset": "claude_code",
        "append": "Additional instructions here"
    }
)
```

## Advanced: Interactive Conversation Session

```python
class ConversationSession:
    def __init__(self, options: ClaudeAgentOptions = None):
        self.client = ClaudeSDKClient(options)
        self.turn_count = 0

    async def start(self):
        await self.client.connect()

        while True:
            user_input = input("\nYou: ")

            if user_input.lower() == 'exit':
                break
            elif user_input.lower() == 'interrupt':
                await self.client.interrupt()
                continue

            await self.client.query(user_input)
            self.turn_count += 1

            async for message in self.client.receive_response():
                print(f"Claude: {message}")

        await self.client.disconnect()
```

## Key Differences

| Feature | `query()` | `ClaudeSDKClient` |
|---------|-----------|------------------|
| Session | New each time | Persistent |
| Context | None | Full conversation |
| Interrupts | Not supported | Supported |
| Hooks | Not supported | Supported |
| Custom tools | Not supported | Supported |

## Important Notes

- Context manager support prevents asyncio cleanup issues
- Avoid using `break` when iterating messages; let iteration complete
- `setting_sources=None` (default) loads no filesystem settings
- Include `"project"` source to load CLAUDE.md files

---

## Project-Specific Implementation

**This project uses ClaudeSDKClient in the Agentic Drop Zone system:**

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

async def prompt_claude_agent(args: PromptArgs) -> None:
    """Process a file using Claude Agent SDK."""

    options_dict = {
        "permission_mode": "bypassPermissions",
        "system_prompt": {"type": "preset", "preset": "claude_code"}
    }

    if args.model:
        options_dict["model"] = args.model

    if args.mcp_server_file:
        options_dict["mcp_servers"] = args.mcp_server_file

    options = ClaudeAgentOptions(**options_dict)

    async with ClaudeSDKClient(options=options) as client:
        await client.query(full_prompt)

        async for message in client.receive_response():
            # Stream responses in Rich panels
            ...
```

**Key implementation details:**

- Uses `bypassPermissions` for automated file processing
- Leverages Claude Code's built-in system prompt preset
- Supports optional model specification
- Handles MCP server configuration via file paths
- Streams responses with Rich console panels for visual feedback

**See `sfs_agentic_drop_zone.py` for complete implementation.**
