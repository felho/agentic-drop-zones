# AI Documentation Update Specification

**Related To:** Claude Agent SDK Migration
**Created:** 2025-11-01
**Updated:** 2025-11-01 (v2.0)
**Status:** Ready for Implementation

---

## Executive Summary

The `ai_docs/` directory contains project documentation and SDK reference materials that reference the old Claude Code SDK. The strategy is to:

1. Update project-specific documentation (`ai_docs.md`) with new type names
2. **Delete and replace** the SDK documentation with fresh content from official Claude Agent SDK docs

**Affected Files:**

- `ai_docs/ai_docs.md` - Project changelog and references (4 updates required)
- `ai_docs/claude-code-python-sdk.md` - **DELETE and download fresh Claude Agent SDK docs**
- `ai_docs/astral-uv-single-file-scripts.md` - No changes required (no SDK references)
- `ai_docs/watch-dog-python-docs.md` - No changes required (watchdog only)

---

## File-by-File Analysis

### 1. `ai_docs/ai_docs.md` (4 updates required)

**Purpose:** Project documentation index and changelog

**Required Changes:**

#### Change 1.1: SDK Documentation Link (Line 4)

**Current:**

```markdown
- https://docs.anthropic.com/en/docs/claude-code/sdk/sdk-python
```

**Action:** Verify link is still valid (it should be, as Claude Code is still the product name)
**Status:** No change required - URL remains valid

#### Change 1.2: ClaudeCodeOptions Reference (Line 25)

**Current:**

```markdown
The file path is passed directly to ClaudeCodeOptions, which handles loading the configuration.
```

**New:**

```markdown
The file path is passed directly to ClaudeAgentOptions, which handles loading the configuration.
```

**Rationale:** Type name changed from `ClaudeCodeOptions` to `ClaudeAgentOptions`

#### Change 1.3: ClaudeCodeOptions Reference (Line 52)

**Current:**

```markdown
### MCP Server File Format

MCP server files can be in JSON or YAML format. The ClaudeCodeOptions class handles loading these files automatically:
```

**New:**

```markdown
### MCP Server File Format

MCP server files can be in JSON or YAML format. The ClaudeAgentOptions class handles loading these files automatically:
```

**Rationale:** Type name changed

#### Change 1.4: ClaudeCodeOptions Reference (Line 65)

**Current:**

```markdown
- The `mcp_server_file` path is passed directly to `ClaudeCodeOptions` as the `mcp_servers` parameter
- ClaudeCodeOptions natively supports file paths (`dict | str | Path`) for MCP configurations
```

**New:**

```markdown
- The `mcp_server_file` path is passed directly to `ClaudeAgentOptions` as the `mcp_servers` parameter
- ClaudeAgentOptions natively supports file paths (`dict | str | Path`) for MCP configurations
```

**Rationale:** Type name changed

---

### 2. `ai_docs/claude-code-python-sdk.md` (DELETE and REPLACE)

**Current Status:** Contains outdated Claude Code SDK documentation

**New Approach:** Delete the entire file and download fresh Claude Agent SDK documentation

**Rationale:**

1. **Cleaner**: No need to manually update 100+ references
2. **Up-to-date**: Official docs are always current
3. **Simpler**: Single operation instead of complex find-replace
4. **Maintainable**: Can easily refresh by re-downloading in the future

**Implementation Steps:**

#### Step 2.1: Delete Old File

```bash
rm ai_docs/claude-code-python-sdk.md
```

#### Step 2.2: Download Fresh Claude Agent SDK Documentation

**Source URL:** `https://docs.claude.com/en/docs/claude-code/sdk/sdk-python`

**Method:** Use WebFetch to extract the complete Python SDK documentation

**Prompt for WebFetch:**
```
Extract the complete Python SDK documentation content. Include all sections:
- Installation
- Core concepts
- Configuration options
- Examples
- Best practices
- Error handling
- Migration guide

Format it as clean markdown suitable for saving as a local reference document.
Include code examples with proper syntax highlighting.
```

#### Step 2.3: Save as New File

**Target:** `ai_docs/claude-code-python-sdk.md`

**Additional Content to Add:**

At the top of the file, add metadata:

```markdown
# Python Agent SDK Documentation

**Source:** https://docs.claude.com/en/docs/claude-code/sdk/sdk-python
**Last Updated:** [current_date]
**SDK Version:** claude-agent-sdk (formerly claude-code-sdk)

---

[Downloaded content goes here]

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

**See `sfs_agentic_drop_zone.py` for complete implementation.**
```

---

### 3. `ai_docs/astral-uv-single-file-scripts.md` (No Changes Required)

**Analysis:** This file documents the Astral UV package manager and single-file script syntax. No SDK references.

**Action:** None

---

### 4. `ai_docs/watch-dog-python-docs.md` (No Changes Required)

**Analysis:** This file documents the Watchdog file monitoring library. No SDK references.

**Action:** None

---

## Implementation Steps

### Phase 1: Update ai_docs.md (5 minutes)

```bash
# Update 4 ClaudeCodeOptions references to ClaudeAgentOptions
# Lines: 25, 52, 65, 66
```

**Changes:**
1. Line 25: `ClaudeCodeOptions` → `ClaudeAgentOptions`
2. Line 52: `ClaudeCodeOptions class` → `ClaudeAgentOptions class`
3. Line 65: `ClaudeCodeOptions` → `ClaudeAgentOptions` (parameter name)
4. Line 66: `ClaudeCodeOptions` → `ClaudeAgentOptions` (Path support)

### Phase 2: Replace SDK Documentation (10 minutes)

#### Step 2.1: Delete Old File

```bash
rm ai_docs/claude-code-python-sdk.md
```

#### Step 2.2: Fetch Fresh Documentation

Use WebFetch tool:
```python
url = "https://docs.claude.com/en/docs/claude-code/sdk/sdk-python"
prompt = """Extract the complete Python SDK documentation content. Include all sections:
- Installation
- Core concepts
- Configuration options
- Examples
- Best practices
- Error handling
- Migration guide

Format it as clean markdown suitable for saving as a local reference document.
Include code examples with proper syntax highlighting."""
```

#### Step 2.3: Process and Save

1. Add metadata header (source, date, SDK version)
2. Insert downloaded content
3. Append project-specific implementation section
4. Save to `ai_docs/claude-code-python-sdk.md`

### Phase 3: Verification (5 minutes)

```bash
# Verify no old references remain
grep -r "ClaudeCodeOptions" ai_docs/
grep -r "claude-code-sdk" ai_docs/ | grep -v "formerly"
grep -r "claude_code_sdk" ai_docs/

# Verify new file exists and has correct content
ls -lh ai_docs/claude-code-python-sdk.md
head -20 ai_docs/claude-code-python-sdk.md
```

---

## Detailed Change List

### ai_docs.md - Line-by-Line Changes

```markdown
Line 25:
- OLD: "The file path is passed directly to ClaudeCodeOptions, which handles loading the configuration."
+ NEW: "The file path is passed directly to ClaudeAgentOptions, which handles loading the configuration."

Line 52:
- OLD: "MCP server files can be in JSON or YAML format. The ClaudeCodeOptions class handles loading these files automatically:"
+ NEW: "MCP server files can be in JSON or YAML format. The ClaudeAgentOptions class handles loading these files automatically:"

Line 65-66:
- OLD: "- The `mcp_server_file` path is passed directly to `ClaudeCodeOptions` as the `mcp_servers` parameter"
- OLD: "- ClaudeCodeOptions natively supports file paths (`dict | str | Path`) for MCP configurations"
+ NEW: "- The `mcp_server_file` path is passed directly to `ClaudeAgentOptions` as the `mcp_servers` parameter"
+ NEW: "- ClaudeAgentOptions natively supports file paths (`dict | str | Path`) for MCP configurations"
```

### claude-code-python-sdk.md - Complete Replacement

**Old:** 800+ lines of outdated Claude Code SDK documentation

**New:** Fresh Claude Agent SDK documentation with:
- Metadata header (source URL, date, version info)
- Complete official documentation (installation, usage, examples)
- Project-specific implementation notes
- Migration guide section
- All references use `claude-agent-sdk`, `claude_agent_sdk`, `ClaudeAgentOptions`

---

## Testing Strategy

### Phase 1: Update Verification

After updating `ai_docs.md`:

```bash
# Check for remaining old references
grep -n "ClaudeCodeOptions" ai_docs/ai_docs.md

# Expected: No results

# Verify new references
grep -n "ClaudeAgentOptions" ai_docs/ai_docs.md

# Expected: 4 results (lines 25, 52, 65, 66)
```

### Phase 2: SDK Documentation Verification

After replacing SDK documentation:

```bash
# Verify old file is deleted
ls ai_docs/claude-code-python-sdk.md

# Expected: File exists (new version)

# Verify new content uses correct SDK
grep -c "claude-agent-sdk" ai_docs/claude-code-python-sdk.md
grep -c "ClaudeAgentOptions" ai_docs/claude-code-python-sdk.md

# Expected: Multiple occurrences

# Verify no old references (except in migration examples)
grep "claude-code-sdk" ai_docs/claude-code-python-sdk.md | grep -v "formerly" | grep -v "OLD" | grep -v "uninstall"

# Expected: No results or only in migration guide
```

### Phase 3: Overall Consistency Check

```bash
# Check entire ai_docs directory
grep -r "ClaudeCodeOptions" ai_docs/

# Expected: Only in migration guide examples (showing OLD vs NEW)

# Verify consistency with codebase
diff <(grep -r "ClaudeAgentOptions" ai_docs/) <(grep -r "ClaudeAgentOptions" sfs_agentic_drop_zone.py)

# Expected: Consistent type names
```

---

## Success Criteria

✅ `ai_docs/ai_docs.md` updated with 4 `ClaudeAgentOptions` references
✅ `ai_docs/claude-code-python-sdk.md` replaced with fresh Claude Agent SDK documentation
✅ New SDK doc includes metadata (source, date, version)
✅ New SDK doc includes project-specific implementation notes
✅ No old references remain (except in migration guide examples)
✅ All code examples use correct import paths and type names
✅ Documentation is consistent with actual codebase

---

## Estimated Timeline

### Recommended Approach (Delete and Replace)

- **ai_docs.md update:** 5 minutes
- **Delete old SDK doc:** 1 minute
- **Download new SDK doc:** 5 minutes
- **Format and save:** 4 minutes
- **Verification:** 5 minutes
- **Total:** ~20 minutes

**Advantages over manual update:**
- ✅ 75-85 minutes faster than manual update (100+ changes)
- ✅ Guaranteed accuracy (official documentation)
- ✅ Easier to refresh in the future
- ✅ Cleaner process (single operation instead of find-replace)

---

## Risk Assessment

### ai_docs.md Updates

- **Risk:** Very Low
- **Impact:** Documentation accuracy
- **Mitigation:** Simple find-replace with verification

### SDK Documentation Replacement

- **Risk:** Very Low
  - Official docs are authoritative source
  - Fresh content is always up-to-date
  - Can re-download if issues arise
- **Impact:** Positive (better documentation)
- **Mitigation:**
  - Keep backup of old file (git handles this)
  - Verify downloaded content before finalizing
  - Add project-specific notes for context

---

## Rollback Plan

If issues arise:

### Quick Rollback (Git)

```bash
# Restore both files from git
git restore ai_docs/ai_docs.md
git restore ai_docs/claude-code-python-sdk.md
```

### Manual Rollback

If needed to restore just the SDK doc:

```bash
# Get from git history
git show HEAD:ai_docs/claude-code-python-sdk.md > ai_docs/claude-code-python-sdk.md
```

---

## Post-Implementation Checklist

- [ ] `ai_docs/ai_docs.md` updated with `ClaudeAgentOptions` (4 references)
- [ ] Old `claude-code-python-sdk.md` deleted
- [ ] Fresh Claude Agent SDK documentation downloaded
- [ ] Metadata header added to new SDK doc
- [ ] Project-specific implementation section added
- [ ] Verification: No old references remain (except migration examples)
- [ ] Verification: grep confirms all updates applied
- [ ] Documentation consistent with `sfs_agentic_drop_zone.py`
- [ ] Migration plan updated to reflect ai_docs changes

---

## Appendix: Bash Commands for Implementation

### Complete Implementation Script

```bash
#!/bin/bash

echo "=== Phase 1: Update ai_docs.md ==="

# Update Line 25
sed -i '' 's/ClaudeCodeOptions, which handles/ClaudeAgentOptions, which handles/' ai_docs/ai_docs.md

# Update Line 52
sed -i '' 's/The ClaudeCodeOptions class handles/The ClaudeAgentOptions class handles/' ai_docs/ai_docs.md

# Update Lines 65-66
sed -i '' 's/to `ClaudeCodeOptions` as the/to `ClaudeAgentOptions` as the/' ai_docs/ai_docs.md
sed -i '' 's/- ClaudeCodeOptions natively supports/- ClaudeAgentOptions natively supports/' ai_docs/ai_docs.md

echo "✅ ai_docs.md updated"

echo ""
echo "=== Phase 2: Replace SDK Documentation ==="

# Delete old file
rm ai_docs/claude-code-python-sdk.md
echo "✅ Old SDK doc deleted"

# Download new documentation using WebFetch tool
# (This step requires using the WebFetch tool in Claude Code)

echo ""
echo "=== Phase 3: Verification ==="

# Verify updates
echo "Checking for old references in ai_docs.md:"
grep -n "ClaudeCodeOptions" ai_docs/ai_docs.md || echo "  ✅ No old references found"

echo ""
echo "Checking new references in ai_docs.md:"
grep -n "ClaudeAgentOptions" ai_docs/ai_docs.md

echo ""
echo "Checking SDK doc:"
ls -lh ai_docs/claude-code-python-sdk.md

echo ""
echo "Done!"
```

### Verification Commands

```bash
# Complete verification suite
grep -r "ClaudeCodeOptions" ai_docs/
grep -r "claude-code-sdk" ai_docs/ | grep -v "formerly" | grep -v "OLD"
grep -r "claude_code_sdk" ai_docs/

# All should return no results or only migration guide examples
```

---

**Last Updated:** 2025-11-01
**Version:** 2.0 (Complete Rewrite - Delete and Replace Strategy)
**Status:** Ready for Implementation
**Estimated Time:** 20 minutes
**Risk Level:** Very Low
