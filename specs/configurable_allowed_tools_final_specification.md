# Configurable Allowed Tools - Final Specification

**Date:** 2025-11-01
**Status:** Final Specification - Ready for Implementation
**Priority:** High (Security Enhancement)
**Test Validation:** ✅ Proven by `test_sdk_allowed_tools.py`

---

## Executive Summary

Add per-drop-zone configurable `allowed_tools` and `disallowed_tools` to enable **non-interactive, fine-grained permission control** without requiring `bypassPermissions` mode.

**Key Finding:** SDK's `allowed_tools` parameter automatically enables whitelisted tools regardless of `permission_mode`, providing secure non-interactive operation.

---

## Problem Statement

### Current State

```python
# Hardcoded in sfs_agentic_drop_zone.py:259
options_dict = {
    "permission_mode": "bypassPermissions",  # All tools, all operations
    "system_prompt": {"type": "preset", "preset": "claude_code"}
}
```

**Issues:**

- ❌ All drop zones have unrestricted system access
- ❌ No per-zone permission differentiation
- ❌ Cannot apply least privilege principle
- ❌ Security risk for simple workflows

---

## Solution Overview

### Add Configurable Tool Whitelisting

```yaml
drop_zones:
  - name: "Echo Drop Zone"
    file_patterns: ["*.txt"]
    reusable_prompt: ".claude/commands/echo.md"
    zone_dirs: ["agentic_drop_zone/echo_zone"]
    agent: "claude_code"
    model: "haiku"

    # NEW: Permission configuration
    permission_mode: "default" # or "acceptEdits" - both work!
    allowed_tools:
      - "Read"
      - "Bash"
    disallowed_tools: # Optional extra protection
      - "Task"
      - "WebFetch"
```

---

## Technical Design

### 1. Data Model Changes

#### Update `DropZone` Pydantic Model

```python
from typing import Literal, Optional

# Type alias for permission modes
PermissionMode = Literal["default", "acceptEdits", "plan", "bypassPermissions"]

class DropZone(BaseModel):
    """Configuration for a single drop zone."""

    name: str = Field(description="Name of the drop zone")
    file_patterns: list[str] = Field(description="File patterns to watch")
    reusable_prompt: str = Field(description="Path to reusable prompt file")
    zone_dirs: list[str] = Field(description="Directories to monitor")
    events: list[EventType] = Field(default=[EventType.CREATED])
    agent: AgentType = Field(default=AgentType.CLAUDE_CODE)
    model: Optional[str] = Field(default="sonnet")

    # NEW: Permission configuration
    permission_mode: PermissionMode = Field(
        default="default",
        description="Permission mode for agent operations"
    )
    allowed_tools: Optional[list[str]] = Field(
        default=None,
        description="Whitelist of allowed tools (enables non-interactive operation)"
    )
    disallowed_tools: Optional[list[str]] = Field(
        default=None,
        description="Blacklist of disallowed tools (extra protection)"
    )

    mcp_server_file: Optional[str] = None
    color: Optional[str] = "cyan"
    create_zone_dir_if_not_exists: bool = False

    @field_validator("permission_mode")
    @classmethod
    def validate_permission_mode(cls, v: str) -> str:
        """Validate and warn about permission modes."""
        valid_modes = {"default", "acceptEdits", "plan", "bypassPermissions"}
        if v not in valid_modes:
            raise ValueError(
                f"Invalid permission_mode: {v}. "
                f"Must be one of: {', '.join(valid_modes)}"
            )

        # Warn if using bypassPermissions without allowed_tools
        if v == "bypassPermissions":
            logger.warning(
                f"⚠️ Using 'bypassPermissions' - consider using 'default' "
                f"with 'allowed_tools' for better security"
            )

        return v
```

#### Update `PromptArgs` Model

```python
class PromptArgs(BaseModel):
    """Arguments for prompt processing."""

    reusable_prompt: str
    file_path: str
    model: Optional[str] = None
    mcp_server_file: Optional[str] = None

    # NEW: Permission configuration
    permission_mode: PermissionMode = Field(default="default")
    allowed_tools: Optional[list[str]] = None
    disallowed_tools: Optional[list[str]] = None

    zone_name: Optional[str] = None
    zone_color: Optional[str] = "cyan"
```

### 2. Implementation Changes

#### Update `Agents.prompt_claude_code()` Method

**File:** `sfs_agentic_drop_zone.py` (around line 237-280)

```python
@staticmethod
async def prompt_claude_code(args: PromptArgs) -> None:
    """Process a file using Claude Agent SDK."""
    full_prompt = Agents.build_prompt(args.reusable_prompt, args.file_path)

    console.print(f"[cyan]ℹ️  Processing prompt with Claude Agent...[/cyan]")

    # Display configuration
    if args.model:
        console.print(f"[dim]   Model: {args.model}[/dim]")
    if args.permission_mode:
        console.print(f"[dim]   Permission: {args.permission_mode}[/dim]")
    if args.allowed_tools:
        tools_display = ', '.join(args.allowed_tools[:3])
        if len(args.allowed_tools) > 3:
            tools_display += f" (+{len(args.allowed_tools) - 3} more)"
        console.print(f"[dim]   Allowed tools: {tools_display}[/dim]")
    if args.disallowed_tools:
        console.print(f"[dim]   Blocked tools: {', '.join(args.disallowed_tools)}[/dim]")

    cli_path = os.getenv("CLAUDE_CODE_PATH", "claude")

    if cli_path != "claude":
        cli_dir = os.path.dirname(cli_path) if os.path.dirname(cli_path) else "."
        current_path = os.environ.get("PATH", "")
        if cli_dir not in current_path:
            os.environ["PATH"] = f"{cli_dir}:{current_path}"

    # Build options with configurable permissions
    options_dict = {
        "permission_mode": args.permission_mode,  # CHANGED: Configurable
        "system_prompt": {"type": "preset", "preset": "claude_code"}
    }

    if args.model:
        options_dict["model"] = args.model

    if args.mcp_server_file:
        options_dict["mcp_servers"] = args.mcp_server_file
        console.print(f"[dim]   MCP config: {args.mcp_server_file}[/dim]")

    # NEW: Add tool restrictions
    if args.allowed_tools:
        options_dict["allowed_tools"] = args.allowed_tools

    if args.disallowed_tools:
        options_dict["disallowed_tools"] = args.disallowed_tools

    options = ClaudeAgentOptions(**options_dict)

    # Rest remains the same
    async with ClaudeSDKClient(options=options) as client:
        await client.query(full_prompt)

        file_name = Path(args.file_path).name
        has_response = False
        zone_workflow = (
            f"{args.zone_name} Workflow" if args.zone_name else "Workflow"
        )
        panel_color = args.zone_color or "cyan"

        async for message in client.receive_response():
            if hasattr(message, "content"):
                for block in message.content:
                    if hasattr(block, "text") and block.text.strip():
                        has_response = True
                        console.print(
                            Panel(
                                Text(block.text),
                                title=f"[bold {panel_color}]🤖 Claude Code • {zone_workflow}[/bold {panel_color}]",
                                subtitle=f"[dim]{file_name}[/dim]",
                                border_style=panel_color,
                                expand=False,
                                padding=(1, 2),
                            )
                        )

        if not has_response:
            console.print(
                Panel(
                    Text("[yellow]No response received[/yellow]"),
                    title=f"[bold yellow]🤖 Claude Code • {zone_workflow}[/bold yellow]",
                    subtitle=f"[dim]{file_name}[/dim]",
                    border_style="yellow",
                    expand=False,
                    padding=(1, 2),
                )
            )

        console.print()
```

#### Update `DropZoneHandler.process_file()` Method

**File:** `sfs_agentic_drop_zone.py` (around line 497-530)

```python
def process_file(self, file_path: str) -> None:
    """Process a file that has been added or modified."""
    path = Path(file_path)

    if not any(path.match(pattern) for pattern in self.drop_zone.file_patterns):
        return

    zone_color = self.drop_zone.color or "green"
    console.print(
        f"\n[bold {zone_color}]📁 Drop Zone: {self.drop_zone.name}[/bold {zone_color}]"
    )
    console.print(f"[yellow]   File: {file_path}[/yellow]")
    console.print(f"[dim]   Agent: {self.drop_zone.agent}[/dim]")
    console.print(f"[dim]   Prompt: {self.drop_zone.reusable_prompt}[/dim]")

    if self.drop_zone.model:
        console.print(f"[dim]   Model: {self.drop_zone.model}[/dim]")

    # NEW: Display permission configuration
    if self.drop_zone.permission_mode:
        console.print(f"[dim]   Permission: {self.drop_zone.permission_mode}[/dim]")
    if self.drop_zone.allowed_tools:
        tools_str = ', '.join(self.drop_zone.allowed_tools[:3])
        if len(self.drop_zone.allowed_tools) > 3:
            tools_str += f" (+{len(self.drop_zone.allowed_tools) - 3} more)"
        console.print(f"[dim]   Allowed: {tools_str}[/dim]")

    if self.drop_zone.mcp_server_file:
        console.print(f"[dim]   MCP: {self.drop_zone.mcp_server_file}[/dim]")

    # Create PromptArgs with permission configuration
    prompt_args = PromptArgs(
        reusable_prompt=self.drop_zone.reusable_prompt,
        file_path=file_path,
        model=self.drop_zone.model,
        mcp_server_file=self.drop_zone.mcp_server_file,
        permission_mode=self.drop_zone.permission_mode,  # NEW
        allowed_tools=self.drop_zone.allowed_tools,      # NEW
        disallowed_tools=self.drop_zone.disallowed_tools,  # NEW
        zone_name=self.drop_zone.name,
        zone_color=self.drop_zone.color,
    )

    asyncio.run(Agents.process_with_agent(self.drop_zone.agent, prompt_args))
```

---

## Configuration Examples

### Example 1: Safe Echo Zone (Read + Basic Bash)

```yaml
- name: "Echo Drop Zone"
  file_patterns: ["*.txt"]
  reusable_prompt: ".claude/commands/echo.md"
  zone_dirs: ["agentic_drop_zone/echo_zone"]
  events: ["created"]
  agent: "claude_code"
  model: "haiku"

  # Safe permissions
  permission_mode: "default"
  allowed_tools:
    - "Read"
    - "Bash" # For mv to archive

  color: "cyan"
  create_zone_dir_if_not_exists: true
```

**Security Level:** 🟢 Low Risk
**Capabilities:** Read files, run basic bash commands (mv, wc, etc.)
**Blocked:** Write, Edit, network access, sub-agents

---

### Example 2: Image Generation Zone (API Workflow)

```yaml
- name: "Image Generation Drop Zone"
  file_patterns: ["*.txt", "*.md"]
  reusable_prompt: ".claude/commands/create_image.md"
  zone_dirs: ["agentic_drop_zone/generate_images_zone"]
  events: ["created"]
  agent: "claude_code"
  model: "sonnet"

  # Permissions for API + file operations
  permission_mode: "default"
  allowed_tools:
    - "Read"
    - "Write"
    - "Bash"
    - "mcp__replicate__create_models_predictions"
    - "mcp__replicate__get_predictions"
  disallowed_tools:
    - "Task"
    - "WebFetch"

  mcp_server_file: ".mcp.json"
  color: "blue"
  create_zone_dir_if_not_exists: true
```

**Security Level:** 🟡 Medium Risk
**Capabilities:** File operations, Replicate API, bash for downloads
**Blocked:** Sub-agents, general web access

---

### Example 3: Data Processing Zone (No Network)

```yaml
- name: "Training Data Generation Zone"
  file_patterns: ["*.csv", "*.jsonl"]
  reusable_prompt: ".claude/commands/more_training_data.md"
  zone_dirs: ["agentic_drop_zone/training_data_zone"]
  events: ["created"]
  agent: "claude_code"
  model: "sonnet"

  # File operations only
  permission_mode: "default"
  allowed_tools:
    - "Read"
    - "Write"
    - "Bash" # For data tools (awk, sed, head, tail)
  disallowed_tools:
    - "WebFetch"
    - "WebSearch"
    - "Task"

  color: "magenta"
  create_zone_dir_if_not_exists: true
```

**Security Level:** 🟢 Low-Medium Risk
**Capabilities:** Read/write data files, bash data processing
**Blocked:** Network access, sub-agents

---

### Example 4: Analysis Only Zone (Read-Only)

```yaml
- name: "Finance Analyzer Zone"
  file_patterns: ["*.csv"]
  reusable_prompt: ".claude/commands/finance_categorizer.md"
  zone_dirs: ["agentic_drop_zone/finance_zone"]
  events: ["created"]
  agent: "claude_code"
  model: "sonnet"

  # Read-only with write for output
  permission_mode: "default"
  allowed_tools:
    - "Read"
    - "Write" # Only for analysis output
    # NO Bash - no command execution

  color: "green"
  create_zone_dir_if_not_exists: true
```

**Security Level:** 🟢 Low Risk
**Capabilities:** Read CSVs, write analysis results
**Blocked:** Bash execution, network, sub-agents

---

## Recommended Permission Modes

### Recommendation by Workflow Type

| Workflow Type        | Permission Mode | Allowed Tools               | Rationale              |
| -------------------- | --------------- | --------------------------- | ---------------------- |
| Read-only analysis   | `default`       | `Read`                      | Minimal permissions    |
| File transformations | `default`       | `Read, Write, Bash`         | Safe file ops          |
| API integration      | `default`       | `Read, Write, Bash, mcp__*` | API + file handling    |
| Complex automation   | `acceptEdits`   | Custom whitelist            | Alternative to default |

### Why `default` is Recommended

1. **Explicit Control:** Only whitelisted tools work
2. **Non-Interactive:** `allowed_tools` bypasses prompts
3. **Semantic Clarity:** "default" mode with explicit whitelist is self-documenting
4. **Future-Proof:** SDK may add more granular controls to `default` mode

### Why NOT `bypassPermissions`

1. **Over-Permissive:** Without `allowed_tools`, all tools available
2. **Dangerous Default:** Easy to forget to restrict
3. **Poor Security Model:** All-or-nothing approach
4. **Maintenance Burden:** Harder to audit what's allowed

---

## Default Values Strategy

### Recommended Defaults

```python
class DropZone(BaseModel):
    permission_mode: PermissionMode = Field(default="default")
    allowed_tools: Optional[list[str]] = Field(default=None)
    disallowed_tools: Optional[list[str]] = Field(default=None)
```

**Behavior:**

- **`permission_mode="default"`** - Safe default
- **`allowed_tools=None`** - Must be explicitly configured
- **No tools specified** → Agent will ask for permission (safe failure mode)

### Migration Path

**For existing configurations without permission settings:**

1. System warns on startup: "No allowed_tools specified for zone X"
2. Defaults to asking permission (safe but non-automated)
3. User must add `allowed_tools` to restore automation

**This is intentional:**

- Forces conscious security decision
- Prevents accidental over-permissioning
- Makes migration visible and deliberate

---

## Security Best Practices

### 1. Principle of Least Privilege

Always specify minimum required tools:

```yaml
# ✅ Good: Minimal tools
allowed_tools:
  - "Read"
  - "Bash"

# ❌ Bad: Too many tools
allowed_tools:
  - "Read"
  - "Write"
  - "Edit"
  - "Bash"
  - "Task"
  - "WebFetch"
```

### 2. Use Disallowed Tools for Defense in Depth

Even with whitelist, add explicit blacklist:

```yaml
allowed_tools:
  - "Read"
  - "Write"
  - "Bash"
disallowed_tools:
  - "Task" # Prevent sub-agent spawning
  - "WebFetch" # Block arbitrary web access
  - "WebSearch" # Block search engines
```

### 3. Bash Command Considerations

**Warning:** `Bash` tool allows ANY bash command. Consider:

```yaml
# If only archiving needed
allowed_tools:
  - "Read"
  - "Bash" # Can run any command!

# Better: Limit bash usage in prompt template
# "Run ONLY: mv [[FILE_PATH]] archive/"
```

**Future Enhancement:** Hook-based bash command filtering (Solution 2 from archive)

### 4. MCP Tool Naming

MCP tools follow pattern: `mcp__<server>__<tool>`

```yaml
allowed_tools:
  - "mcp__replicate__create_models_predictions"
  - "mcp__replicate__get_predictions"
  # Specific tools only, not all Replicate tools
```

---

## Startup Validation

### Add Security Warnings

```python
def check_environment_variables():
    """Check environment and display security warnings."""
    # ... existing checks ...

    # NEW: Check permission configurations
    if config and config.drop_zones:
        risky_zones = []
        unprotected_zones = []

        for zone in config.drop_zones:
            # Check for bypassPermissions
            if zone.permission_mode == "bypassPermissions":
                if not zone.allowed_tools:
                    risky_zones.append(zone.name)

            # Check for missing allowed_tools
            if not zone.allowed_tools:
                unprotected_zones.append(zone.name)

        if risky_zones:
            console.print("\n[bold red]⚠️  Security Warning: Unrestricted Zones[/bold red]")
            console.print("[red]The following zones use 'bypassPermissions' without tool restrictions:[/red]")
            for name in risky_zones:
                console.print(f"[dim]   - {name}[/dim]")
            console.print("[yellow]Consider using 'default' mode with 'allowed_tools'[/yellow]\n")

        if unprotected_zones:
            console.print("\n[bold yellow]⚠️  Warning: Zones Without Tool Whitelist[/bold yellow]")
            console.print("[yellow]The following zones have no 'allowed_tools' specified:[/yellow]")
            for name in unprotected_zones:
                console.print(f"[dim]   - {name}[/dim]")
            console.print("[dim]These zones may prompt for permissions (non-automated)[/dim]\n")
```

---

## Documentation Updates

### 1. Update README.md

Add new section after "Configuration (drops.yaml)":

````markdown
## Permission Configuration

Each drop zone can specify fine-grained permissions using `allowed_tools`:

### Basic Configuration

```yaml
drop_zones:
  - name: "Echo Zone"
    permission_mode: "default" # Recommended
    allowed_tools: # Whitelist - required for non-interactive
      - "Read"
      - "Bash"
    disallowed_tools: # Optional blacklist
      - "Task"
      - "WebFetch"
```
````

### Available Tools

**Core Tools:**

- `Read` - Read files
- `Write` - Create new files
- `Edit` - Modify existing files
- `Bash` - Run bash commands
- `Glob` - Find files by pattern
- `Grep` - Search file contents

**Advanced Tools:**

- `Task` - Spawn sub-agents
- `WebFetch` - Fetch web content
- `WebSearch` - Search the web
- `TodoWrite` - Manage todo lists
- `NotebookEdit` - Edit Jupyter notebooks

**MCP Tools:**

- `mcp__<server>__<tool>` - MCP server tools
- Example: `mcp__replicate__create_models_predictions`

### Security Recommendations

🟢 **Low Risk Workflows** (read-only, analysis):

```yaml
permission_mode: "default"
allowed_tools: ["Read"]
```

🟡 **Medium Risk Workflows** (file transformations):

```yaml
permission_mode: "default"
allowed_tools: ["Read", "Write", "Bash"]
disallowed_tools: ["Task", "WebFetch"]
```

🔴 **High Risk Workflows** (API integration, network):

```yaml
permission_mode: "default"
allowed_tools: ["Read", "Write", "Bash", "mcp__*"]
disallowed_tools: ["Task"]
```

### Permission Modes

- **`default`** ✅ Recommended - Works with allowed_tools for non-interactive
- **`acceptEdits`** ✅ Alternative - Also works with allowed_tools
- **`bypassPermissions`** ⚠️ Legacy - Use only if needed, add allowed_tools
- **`plan`** ℹ️ Analysis only - No execution

````

### 2. Update drops.yaml Comments

```yaml
drop_zones:
  - name: "Echo Drop Zone"
    file_patterns: ["*.txt"]
    reusable_prompt: ".claude/commands/echo.md"
    zone_dirs: ["agentic_drop_zone/echo_zone"]
    events: ["created"]
    agent: "claude_code"
    model: "haiku"

    # Permission configuration (enables non-interactive operation)
    permission_mode: "default"  # Recommended: default or acceptEdits

    # Whitelist of allowed tools (required for non-interactive)
    allowed_tools:
      - "Read"   # Read files
      - "Bash"   # Run bash commands (mv, wc, etc.)

    # Optional: Blacklist specific tools for extra protection
    # disallowed_tools:
    #   - "Task"      # Block sub-agent spawning
    #   - "WebFetch"  # Block web access

    color: "cyan"
    create_zone_dir_if_not_exists: true
````

---

## Testing Strategy

### 1. Unit Tests

```python
def test_allowed_tools_configuration():
    """Test allowed_tools is properly configured."""
    config_data = {
        "drop_zones": [{
            "name": "Test Zone",
            "file_patterns": ["*.txt"],
            "reusable_prompt": "test.md",
            "zone_dirs": ["test_dir"],
            "permission_mode": "default",
            "allowed_tools": ["Read", "Bash"]
        }]
    }

    config = DropsConfig(**config_data)
    assert config.drop_zones[0].permission_mode == "default"
    assert config.drop_zones[0].allowed_tools == ["Read", "Bash"]

def test_disallowed_tools_configuration():
    """Test disallowed_tools is properly configured."""
    config_data = {
        "drop_zones": [{
            "name": "Test Zone",
            "file_patterns": ["*.txt"],
            "reusable_prompt": "test.md",
            "zone_dirs": ["test_dir"],
            "allowed_tools": ["Read"],
            "disallowed_tools": ["Task"]
        }]
    }

    config = DropsConfig(**config_data)
    assert config.drop_zones[0].disallowed_tools == ["Task"]

def test_permission_mode_validation():
    """Test invalid permission mode raises error."""
    config_data = {
        "drop_zones": [{
            "name": "Test Zone",
            "file_patterns": ["*.txt"],
            "reusable_prompt": "test.md",
            "zone_dirs": ["test_dir"],
            "permission_mode": "invalid"
        }]
    }

    with pytest.raises(ValueError):
        DropsConfig(**config_data)
```

### 2. Integration Tests

**Use existing:** `test_sdk_allowed_tools.py` validates the core functionality

### 3. Manual Testing Checklist

- [ ] Drop zone with `default` + `allowed_tools` works non-interactively
- [ ] Drop zone without `allowed_tools` prompts for permission
- [ ] `disallowed_tools` blocks specified tools
- [ ] Invalid `permission_mode` shows error
- [ ] Console displays permission configuration
- [ ] Security warnings appear for risky configurations
- [ ] Migration from old config works

---

## Implementation Checklist

### Phase 1: Core Implementation

- [ ] Update `DropZone` Pydantic model with new fields
- [ ] Add `PermissionMode` type alias
- [ ] Update `PromptArgs` model
- [ ] Modify `Agents.prompt_claude_code()` method
- [ ] Update `DropZoneHandler.process_file()` method
- [ ] Add `permission_mode` field validator

### Phase 2: User Experience

- [ ] Add console output for permissions
- [ ] Add startup security warnings
- [ ] Update `check_environment_variables()`
- [ ] Add permission summary on startup

### Phase 3: Documentation

- [ ] Update README.md with permission section
- [ ] Add inline comments to drops.yaml
- [ ] Update example configurations
- [ ] Create migration guide if needed

### Phase 4: Testing

- [ ] Run unit tests
- [ ] Run integration test (test_sdk_allowed_tools.py)
- [ ] Manual testing with real drop zones
- [ ] Validate security warnings

---

## Migration Guide

### For New Users

Simply add to your drop zone configuration:

```yaml
permission_mode: "default"
allowed_tools:
  - "Read"
  - "Bash"
```

### For Existing Users

**Option 1: Keep Current Behavior** (Not Recommended)

```yaml
# Keep bypassPermissions but add tool restriction
permission_mode: "bypassPermissions"
allowed_tools:
  - "Read"
  - "Write"
  - "Bash"
```

**Option 2: Migrate to Safer Mode** (Recommended)

```yaml
# Change to default + allowed_tools
permission_mode: "default"
allowed_tools:
  - "Read"
  - "Bash"
  # Add only tools your workflow needs
```

---

## Success Criteria

- [x] Empirically proven with `test_sdk_allowed_tools.py`
- [ ] `permission_mode` configurable per drop zone
- [ ] `allowed_tools` enables non-interactive operation
- [ ] `disallowed_tools` provides extra protection
- [ ] Clear documentation and examples
- [ ] Security warnings for risky configs
- [ ] No regression in existing functionality
- [ ] Default mode is safer than current

---

## Related Files

- **Test:** `test_sdk_allowed_tools.py` - Empirical validation
- **Archive:** `specs/archive_non_interactive_permission_solutions.md` - Alternative solutions
- **Implementation:** `sfs_agentic_drop_zone.py` - Main application file
- **Configuration:** `drops.yaml` - Drop zone configurations

---

## Conclusion

This specification provides a **proven, secure, and user-friendly** solution for fine-grained permission control in the Agentic Drop Zone system.

**Key Benefits:**

1. ✅ Non-interactive operation without `bypassPermissions`
2. ✅ Fine-grained per-zone permission control
3. ✅ Principle of least privilege
4. ✅ Simple YAML configuration
5. ✅ Empirically tested and validated

**Ready for implementation.**

---

_Specification Status: ✅ Final - Ready for Implementation_
_Test Validation: ✅ Passed - See test_sdk_allowed_tools.py_
_Estimated Implementation Time: 2-3 hours_
