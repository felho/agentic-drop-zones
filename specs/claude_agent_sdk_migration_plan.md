# Claude Agent SDK Migration Plan
**Project:** Agentic Drop Zone
**Migration Goal:** Claude Code SDK → Claude Agent SDK
**Documentation:** https://docs.claude.com/en/docs/claude-code/sdk/migration-guide
**Created:** 2025-11-01

---

## Executive Summary

The Claude Code SDK has been renamed to **Claude Agent SDK** to reflect its expanded capabilities beyond coding tasks. This involves package name changes, import modifications, and several breaking changes that require code updates.

**Key Changes:**
- **3 Critical Updates Required:** Package name, imports, and type names must be updated
- **1 Additional Required Change:** Add `claude_code` preset to maintain backward compatibility with system prompt
- **1 Non-Applicable Change:** Settings sources (project uses custom configuration)

**Migration Strategy:** To ensure 100% backward compatibility, the migration will include enabling the **`claude_code` preset** by default:
```python
systemPrompt: {"type": "preset", "preset": "claude_code"}
```
This maintains the exact behavior of the current system and can be safely removed later if testing confirms it's unnecessary (though recommended to keep for stability).

---

## Current Project Configuration Status

**Active Drop Zones (drops.yaml):**
- ✅ Echo Drop Zone (line 2-11) - **ONLY ACTIVE ZONE**
  - Agent: `claude_code` (agent type, not SDK name - remains unchanged)
  - Model: `haiku`
  - Events: `["created"]` (fixed from `["created", "modified"]`)
  - MCP: Disabled (no `mcp_server_file` configured)

**Disabled Drop Zones (commented out):**
- 🔇 Gemini Echo Drop Zone (line 14-22)
- 🔇 Image Generation Drop Zone (line 24-33) - **MCP Required**
- 🔇 Gemini Image Generation Drop Zone (line 35-44) - **MCP Required**
- 🔇 Image Edit Drop Zone (line 46-55) - **MCP Required**
- 🔇 Training Data Generation Zone (line 57-65)
- 🔇 Morning Debrief Zone (line 67-75) - **Whisper Required**
- 🔇 Finance Categorizer Zone (line 77-85)

**Implications for Migration:**
- Minimal testing can focus on Echo Zone only
- MCP integration testing requires enabling Image Generation Zone
- Comprehensive testing requires uncommenting multiple zones

---

## Analysis of Relevant Changes

**Summary:**
- ✅ **3 Critical Changes:** Package (1), Imports (2), Type Names (3)
- ✅ **1 Additional Required Change:** System Prompt Preset (4)
- ⚠️ **1 Not Applicable:** Settings Sources (5)

### 1. ✅ Package Name Change (RELEVANT - CRITICAL)
**Affected file:** `sfs_agentic_drop_zone.py` (Line 5)

**Current state:**
```python
# dependencies = [
#     "claude-code-sdk",
```

**Required change:**
```python
# dependencies = [
#     "claude-agent-sdk",
```

**Rationale:** The Python package name has changed. Must be updated in uv script dependencies.

---

### 2. ✅ Import Statement Changes (RELEVANT - CRITICAL)
**Affected file:** `sfs_agentic_drop_zone.py` (Line 39)

**Current state:**
```python
from claude_code_sdk import ClaudeSDKClient, ClaudeCodeOptions
```

**Required change:**
```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions
```

**Rationale:**
- Python module name: `claude_code_sdk` → `claude_agent_sdk`
- Type name: `ClaudeCodeOptions` → `ClaudeAgentOptions`

---

### 3. ✅ ClaudeCodeOptions → ClaudeAgentOptions (RELEVANT - CRITICAL)
**Affected file:** `sfs_agentic_drop_zone.py` (Line 273)

**Current state:**
```python
options = ClaudeCodeOptions(**options_dict)
```

**Required change:**
```python
options = ClaudeAgentOptions(**options_dict)
```

**Rationale:** The Python SDK type name has changed to align with the new branding.

---

### 4. ⚠️ System Prompt No Longer Default (RELEVANT - ACTION REQUIRED)
**Status:** REQUIRES IMPLEMENTATION

**Migration guide states:**
> "The SDK no longer uses Claude Code's system prompt by default."

**Critical Question:**
In the old API, when calling `query(prompt)`, the `prompt` was sent as a **user message** and a **default Claude Code system prompt** was automatically added. In the new API, **no system prompt is added**.

**Analysis:**
- The current code uses **custom prompts** from `.claude/commands/*.md` files
- The `prompt_claude_code()` function uses the `full_prompt` variable (Line 277)
- This `full_prompt` is passed to `client.query()` as a **user message**

**Code reference:**
```python
# Line 240: Build full prompt using the build_prompt method
full_prompt = Agents.build_prompt(args.reusable_prompt, args.file_path)

# Line 277: Use the custom prompt
await client.query(full_prompt)
```

**Potential Issue:**
- **Old SDK:** `query(full_prompt)` → User message + automatic Claude Code system prompt (with tool usage instructions)
- **New SDK:** `query(full_prompt)` → User message only, **no system prompt**

**However, Migration Guide Rationale:**
> "Provides better control and isolation for SDK applications. You can now build agents with custom behavior without inheriting Claude Code's **CLI-focused instructions**."

This suggests:
1. The old default system prompt was **CLI-specific** (for terminal usage)
2. For **SDK usage** (not CLI), this default prompt may not be necessary
3. Tool access is managed through **`options`** configuration, not system prompts
4. The SDK handles tool integration independently

**Conclusion:** This change is **RELEVANT AND REQUIRES ACTION** because:
- The current codebase relies on the automatic system prompt for tool usage
- Custom prompts in `.claude/commands/*.md` assume tool access is already configured
- Removing the system prompt is a **behavioral change**, not just a rename
- Without the system prompt, agent behavior **will differ** from current implementation

**RECOMMENDED ACTION:** Enable the **`claude_code` preset** during migration to maintain existing behavior:
```python
options_dict = {
    "permission_mode": "bypassPermissions",
    "systemPrompt": {"type": "preset", "preset": "claude_code"}  # REQUIRED for backward compatibility
}
```

**Why this is necessary:**
- ✅ Maintains 100% backward compatibility with current behavior
- ✅ Ensures tool usage works exactly as before
- ✅ Prevents unexpected behavioral changes
- ✅ Follows migration guide best practices for existing applications
- ✅ Can be removed later if testing confirms it's not needed (but safer to keep)

**Implementation:** This will be added as part of **Phase 4** (Type Name Modifications), not as an optional fallback.

---

### 5. ⚠️ Settings Sources No Longer Default (NOT RELEVANT)
**Status:** NOT APPLICABLE

**Migration guide states:**
> "The SDK no longer automatically reads filesystem settings (CLAUDE.md, settings.json, slash commands)"

**Analysis:**
- The project uses **its own configuration system** (`drops.yaml`)
- **Does not use** filesystem-based Claude Code settings:
  - No CLAUDE.md reading
  - No settings.json reading
  - No slash command integration with the SDK
- Only 3 options are passed to the SDK (Line 258-270):
  - `permission_mode: "bypassPermissions"`
  - `model: <model_name>` (optional)
  - `mcp_servers: <file_path>` (optional)

**Code reference:**
```python
# Line 258-270
options_dict = {
    "permission_mode": "bypassPermissions"
}

if args.model:
    options_dict["model"] = args.model

if args.mcp_server_file:
    options_dict["mcp_servers"] = args.mcp_server_file

options = ClaudeCodeOptions(**options_dict)
```

**Conclusion:** This breaking change also does not affect the project. No need for `settingSources` configuration.

---

## Migration Steps

### Phase 1: Preparation
- [ ] **1.1** Create repository backup
- [ ] **1.2** Create git branch: `git checkout -b feature/migrate-to-claude-agent-sdk`
- [ ] **1.3** Test and document current functionality
- [ ] **1.4** List dependencies: `uv pip list | grep claude`

### Phase 2: Package Update
- [ ] **2.1** Remove old package (if manually installed):
  ```bash
  pip uninstall claude-code-sdk -y
  ```
- [ ] **2.2** Modify uv script dependencies in `sfs_agentic_drop_zone.py` file:
  - **Line:** 5
  - **Change:** `"claude-code-sdk"` → `"claude-agent-sdk"`

### Phase 3: Import Modifications
- [ ] **3.1** Update import statement in `sfs_agentic_drop_zone.py` file:
  - **Line:** 39
  - **Change:**
    ```python
    # BEFORE:
    from claude_code_sdk import ClaudeSDKClient, ClaudeCodeOptions

    # AFTER:
    from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions
    ```

### Phase 4: Type Name Modifications and System Prompt Configuration
- [ ] **4.1** Rename ClaudeCodeOptions in `sfs_agentic_drop_zone.py` file:
  - **Line:** 273
  - **Change:**
    ```python
    # BEFORE:
    options = ClaudeCodeOptions(**options_dict)

    # AFTER:
    options = ClaudeAgentOptions(**options_dict)
    ```

- [ ] **4.2** Add claude_code preset to maintain backward compatibility:
  - **Line:** 258-273 (in the `prompt_claude_code()` function)
  - **Action:** Add `systemPrompt` to `options_dict`
  - **Change:**
    ```python
    # BEFORE:
    options_dict = {
        "permission_mode": "bypassPermissions"  # Bypass all permission prompts
    }

    # AFTER:
    options_dict = {
        "permission_mode": "bypassPermissions",  # Bypass all permission prompts
        "systemPrompt": {"type": "preset", "preset": "claude_code"}  # Maintain backward compatibility
    }
    ```
  - **Rationale:** The current codebase relies on the automatic Claude Code system prompt. Adding this preset ensures 100% backward compatibility and prevents behavioral changes.

### Phase 5: Documentation Update
- [ ] **5.1** Update README.md:
  - **References:** "Claude Code SDK" → "Claude Agent SDK"
  - **Lines to check:** 39, 98-106, 139-152

- [ ] **5.2** Update comments in `sfs_agentic_drop_zone.py`:
  - **Line 14:** `"""Agentic Drop Zone - File monitoring and processing system."""`
  - **Line 52:** `"Required for Claude Code SDK authentication"` → `"Required for Claude Agent SDK authentication"`
  - **Line 242:** `"Processing prompt with Claude Code..."` → `"Processing prompt with Claude Agent..."`
  - **Line 275:** `"# Minimal Claude Code setup..."` → `"# Minimal Claude Agent setup..."`

- [ ] **5.3** Update console output messages (optional):
  - **Line 242:** `"Processing prompt with Claude Code"` can be kept or updated
  - **Line 296:** Panel titles: `"Claude Code • ..."` can be kept for consistency

### Phase 6: Testing

**IMPORTANT:** Based on current `drops.yaml` configuration:
- ✅ **Echo Drop Zone** - ENABLED (only active zone)
- 🔇 **Gemini Echo Drop Zone** - DISABLED (commented out)
- 🔇 **Image Generation Zones** - DISABLED (commented out)
- 🔇 **Training Data Zone** - DISABLED (commented out)
- 🔇 **Morning Debrief Zone** - DISABLED (commented out)
- 🔇 **Finance Categorizer Zone** - DISABLED (commented out)

**Testing Strategy:**
- **Minimum Required:** Test only Echo Drop Zone (6.1-6.3)
- **Recommended:** Enable and test Image Generation Zone for MCP validation (6.4)
- **Comprehensive:** Enable all zones for full validation (6.7)

---

- [ ] **6.1** Verify dependency installation:
  ```bash
  uv run --with claude-agent-sdk python -c "from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions; print('✅ Import successful')"
  ```

- [ ] **6.2** Echo Zone test (CRITICAL - Verifies system prompt change):
  ```bash
  # This is the ONLY active zone in current configuration
  uv run sfs_agentic_drop_zone.py &
  cp example_input_files/echo.txt agentic_drop_zone/echo_zone/
  ```
  **Verify:**
  - [ ] File is read successfully
  - [ ] Content is displayed correctly
  - [ ] File is moved to archive
  - [ ] Response quality matches previous behavior
  - [ ] **Tool usage works without default system prompt**

- [ ] **6.3** Tool usage validation test (uses active Echo Zone):
  ```bash
  # Create a test file that requires multiple tool operations
  echo "Test file for tool validation" > agentic_drop_zone/echo_zone/tool_test.txt
  ```
  **Verify agent can:**
  - [ ] Read files
  - [ ] Execute bash commands
  - [ ] Move/archive files
  - [ ] Stream responses correctly

---

**Optional Extended Testing** (requires enabling zones in drops.yaml):

- [ ] **6.4** MCP integration test (OPTIONAL - requires enabling zone):
  **Prerequisites:**
  1. Uncomment Image Generation Zone in `drops.yaml` (lines 24-33)
  2. Ensure `REPLICATE_API_TOKEN` is set in `.env`
  3. Restart the drop zone system

  ```bash
  # Edit drops.yaml to uncomment:
  # - name: "Image Generation Drop Zone"
  #   file_patterns: ["*.txt", "*.md"]
  #   ...

  # Then test:
  cp example_input_files/cats.txt agentic_drop_zone/generate_images_zone/
  ```
  **Verify:**
  - [ ] MCP tools are accessible
  - [ ] Image generation completes successfully
  - [ ] Files are saved correctly
  - [ ] **MCP tool usage works without default system prompt**

- [ ] **6.5** Error handling test:
  ```bash
  # Test with invalid API key (edit .env temporarily)
  # Test with missing prompt file
  # Test with invalid model name in drops.yaml
  ```

- [ ] **6.6** Compare behavior with old SDK (if possible):
  - [ ] Record response quality before migration
  - [ ] Compare response quality after migration
  - [ ] Verify no degradation in agent capabilities
  - [ ] Document any differences

---

**Comprehensive Testing** (optional - all zones):

- [ ] **6.7** Enable and test additional drop zones:

  **To enable zones, uncomment in `drops.yaml`:**
  ```yaml
  # Gemini Echo (line 14-22) - Test Gemini CLI agent
  # Image Generation (line 24-33) - Test MCP with Claude
  # Gemini Image Gen (line 35-44) - Test MCP with Gemini
  # Image Edit (line 46-55) - Test image editing workflow
  # Training Data (line 57-65) - Test data generation
  # Morning Debrief (line 67-75) - Test audio processing (requires whisper)
  # Finance (line 77-85) - Test financial analysis
  ```

  **Test each enabled zone:**
  - [ ] Gemini Echo Drop Zone (Gemini CLI agent)
  - [ ] Image Generation Zone (MCP integration)
  - [ ] Image Edit Zone (MCP integration)
  - [ ] Training Data Zone (file operations)
  - [ ] Morning Debrief Zone (audio processing with whisper)
  - [ ] Finance Categorizer Zone (data analysis)

  **Quick test commands:**
  ```bash
  # Training Data Zone
  cp example_input_files/twitter_classification_dataset.csv agentic_drop_zone/training_data_zone/

  # Morning Debrief Zone
  cp example_input_files/yt_script_5_agent_interaction_patterns_4m.mp4 agentic_drop_zone/morning_debrief_zone/

  # Finance Zone
  cp example_input_files/bank_statement.csv agentic_drop_zone/finance_zone/
  ```

---

**Recommended Testing Path:**

**Minimal (Required):**
1. Steps 6.1-6.3: Basic functionality with Echo Zone
2. Estimated time: 15-20 minutes

**Standard (Recommended):**
1. Steps 6.1-6.3: Basic functionality
2. Step 6.4: MCP integration (enable Image Generation Zone)
3. Estimated time: 30-40 minutes

**Comprehensive (Thorough):**
1. Steps 6.1-6.6: All core tests
2. Step 6.7: Enable and test all zones
3. Estimated time: 60-90 minutes

### Phase 7: Version Control and Deploy
- [ ] **7.1** Commit changes:
  ```bash
  git add .
  git commit -m "Migrate from Claude Code SDK to Claude Agent SDK

  - Updated package dependency: claude-code-sdk → claude-agent-sdk
  - Updated imports: claude_code_sdk → claude_agent_sdk
  - Renamed ClaudeCodeOptions → ClaudeAgentOptions
  - Added claude_code preset for backward compatibility
  - Updated documentation and comments

  Breaking changes applied:
  - Package name change
  - Import path change
  - Type name change
  - System prompt now requires explicit configuration

  Compatibility measures:
  - Enabled claude_code preset to maintain exact previous behavior
  - Ensures 100% backward compatibility with current implementation
  - All existing workflows remain unchanged

  Non-applicable changes:
  - Settings sources (using drops.yaml config)
  "
  ```

- [ ] **7.2** Create pull request (if team project)

- [ ] **7.3** Create tag after merge to main:
  ```bash
  git tag -a v2.0.0-claude-agent-sdk -m "Migrated to Claude Agent SDK"
  git push origin v2.0.0-claude-agent-sdk
  ```

---

## System Prompt Configuration Strategy

The migration plan includes enabling the `claude_code` preset **by default** to ensure backward compatibility. This section explains the options and rationale.

### Default Implementation: Use Claude Code Preset (RECOMMENDED - INCLUDED IN MIGRATION)
The new SDK **officially supports** a "claude_code" preset for backward compatibility, and this is **included by default** in Phase 4 of the migration:

**TypeScript Example (from migration guide):**
```typescript
const result = query({
  prompt: "Hello",
  options: {
    systemPrompt: { type: "preset", preset: "claude_code" }
  }
});
```

**Python Implementation:**
```python
# In prompt_claude_code() function, around Line 258-273
options_dict = {
    "permission_mode": "bypassPermissions",
    "systemPrompt": {"type": "preset", "preset": "claude_code"}
}

if args.model:
    options_dict["model"] = args.model

if args.mcp_server_file:
    options_dict["mcp_servers"] = args.mcp_server_file

options = ClaudeAgentOptions(**options_dict)
```

**Why this is the recommended default:**
- ✅ Official SDK feature (documented in migration guide)
- ✅ Maintains exact current behavior (100% backward compatibility)
- ✅ Minimal code change (one line addition in Phase 4.2)
- ✅ No need to write custom system prompt
- ✅ Automatically updated with SDK improvements
- ✅ Can be safely removed later if testing confirms unnecessary

**This is implemented in Phase 4.2 of the migration.**

---

### Alternative Option 1: Custom Explicit System Prompt (If preset doesn't work)
Only use if the `claude_code` preset is insufficient for some reason:

```python
# In prompt_claude_code() function, around Line 258-273
options_dict = {
    "permission_mode": "bypassPermissions",
    "systemPrompt": """You are Claude, an AI assistant with access to various tools.

You can:
- Read and write files using the Read and Write tools
- Execute bash commands using the Bash tool
- Use MCP servers for specialized tasks (image generation, etc.)

When the user requests actions:
1. Analyze the request carefully
2. Use the appropriate tools to complete the task
3. Provide clear feedback on what you're doing
4. Handle errors gracefully and report issues

Always execute commands and use tools as needed to complete the user's request."""
}
```

### Alternative Option 2: Enhance Custom Prompts
Add tool usage instructions directly to the `.claude/commands/*.md` files:

```markdown
# Echo Command

You have access to the following tools: Read, Write, Bash.
Use them as needed to complete this workflow.

Echo the contents of the file at DROPPED_FILE_PATH and provide a brief summary.
...
```

### Implementation Strategy
**Default Approach (Phase 4.2):**
- ✅ Enable `claude_code` preset during migration
- ✅ Test with preset enabled (Phase 6)
- ✅ Maintain backward compatibility
- ⚠️ Can optionally remove preset later if extensive testing confirms it's unnecessary

**Only If Needed (Alternatives):**
- **If preset behaves unexpectedly:** Try Alternative Option 1 (custom explicit prompt)
- **If system-wide solution doesn't work:** Try Alternative Option 2 (enhance individual prompts)
- **If complete failure:** Rollback and investigate

### Testing Strategy
- Test with `claude_code` preset **enabled** (as implemented in Phase 4.2)
- Verify all functionality works as expected
- Document any differences in behavior
- **After several weeks of stable operation:** Optionally test without preset to see if it's truly necessary
- Keep preset enabled unless proven unnecessary for maximum safety

---

## Rollback Plan

If problems arise during migration:

### Quick Rollback (Git)
```bash
# Revert branch
git checkout main
git branch -D feature/migrate-to-claude-agent-sdk

# Or revert commit
git revert <commit-hash>
```

### Manual Rollback
1. **Revert sfs_agentic_drop_zone.py** modifications:
   - Line 5: `"claude-agent-sdk"` → `"claude-code-sdk"`
   - Line 39: `from claude_agent_sdk` → `from claude_code_sdk`
   - Line 39: `ClaudeAgentOptions` → `ClaudeCodeOptions`
   - Line 273: `ClaudeAgentOptions` → `ClaudeCodeOptions`

2. **Reinstall old package:**
   ```bash
   pip uninstall claude-agent-sdk -y
   pip install claude-code-sdk
   ```

---

## Estimated Timeline

### Minimal Migration (Recommended for Initial Validation)

| Phase | Estimated Time | Risk |
|-------|----------------|------|
| 1. Preparation | 15 minutes | Low |
| 2. Package update | 5 minutes | Low |
| 3. Import modifications | 5 minutes | Low |
| 4. Type name & preset config | 10 minutes | Low |
| 4a. - Type name change | 3 minutes | Low |
| 4b. - Add claude_code preset | 7 minutes | Low |
| 5. Documentation update | 20 minutes | Low |
| 6. Testing (Minimal - Echo Zone only) | 15-20 minutes | Low |
| 6a. - Basic import test | 5 minutes | Low |
| 6b. - Echo Zone test | 10-15 minutes | Low |
| 7. Version control | 10 minutes | Low |
| **TOTAL (Minimal)** | **~80-85 minutes** | **Low** |

### Standard Migration (Recommended for Production)

| Phase | Estimated Time | Risk |
|-------|----------------|------|
| 1-5. Code changes (inc. preset) | 55 minutes | Low |
| 6. Testing (Standard - Echo + MCP) | 30-40 minutes | Low-Medium |
| 6a. - Basic tests (6.1-6.3) | 15-20 minutes | Low |
| 6b. - MCP integration (6.4) | 15-20 minutes | Low-Medium |
| 7. Version control | 10 minutes | Low |
| **TOTAL (Standard)** | **~95-105 minutes** | **Low** |

### Comprehensive Migration (Full Validation)

| Phase | Estimated Time | Risk |
|-------|----------------|------|
| 1-5. Code changes (inc. preset) | 55 minutes | Low |
| 6. Testing (Comprehensive - All zones) | 60-90 minutes | Low-Medium |
| 6a. - Basic tests (6.1-6.3) | 15-20 minutes | Low |
| 6b. - MCP integration (6.4) | 15-20 minutes | Low-Medium |
| 6c. - Error handling (6.5) | 10 minutes | Low |
| 6d. - Behavior comparison (6.6) | 10-15 minutes | Low |
| 6e. - All zones (6.7) | 10-25 minutes | Low-Medium |
| 7. Version control | 10 minutes | Low |
| **TOTAL (Comprehensive)** | **~125-155 minutes** | **Low** |

**Notes:**
- **Minimal** path tests only the Echo Drop Zone (currently the only active zone)
- **Standard** path adds MCP integration testing (requires uncommenting Image Generation Zone)
- **Comprehensive** path tests all available drop zones (requires uncommenting all zones)
- Testing time varies based on API response times and file processing complexity
- **claude_code preset included by default** - ensures backward compatibility
- **Risk lowered to Low** across all paths due to preset maintaining exact current behavior

---

## Risks and Mitigation

### Identified Risks

1. **System Prompt Removal Impact** ⚠️ **RISK MITIGATED**
   - **Risk:** Tool usage and agent behavior may be affected by the removal of default Claude Code system prompt
   - **Likelihood:** Low (SDK should handle tools independently, official preset available if needed)
   - **Impact:** Low-Medium (can be immediately fixed with one line of code)
   - **Mitigation:**
     - Comprehensive testing in Phase 6.2, 6.3, 6.4
     - Compare behavior before/after migration (6.6)
     - **CONFIRMED:** Official `claude_code` preset available for instant rollback
   - **Fallback (One-line fix):**
     ```python
     options_dict["systemPrompt"] = {"type": "preset", "preset": "claude_code"}
     ```
   - **Risk Assessment:** **Significantly reduced** due to official preset availability

2. **API Breaking Changes**
   - **Risk:** The new SDK API may have changed beyond documented changes
   - **Likelihood:** Low (only naming changes documented)
   - **Mitigation:** Thorough testing in testing phase

3. **Dependency Conflicts**
   - **Risk:** Other packages may not be compatible with new SDK
   - **Likelihood:** Very low
   - **Mitigation:** Using uv as isolated environment

4. **MCP Server Compatibility**
   - **Risk:** MCP server integration may have changed
   - **Likelihood:** Low
   - **Mitigation:** MCP testing in step 6.4

5. **Production Downtime**
   - **Risk:** If this is a production system
   - **Likelihood:** N/A (development project)
   - **Mitigation:** Git branch usage, thorough testing

---

## Success Criteria

### Minimal Migration Success (Required)
The migration is considered minimally successful if:

- [ ] ✅ All imports work without errors (`from claude_agent_sdk import ...`)
- [ ] ✅ Echo Drop Zone runs successfully (only active zone)
- [ ] ✅ File reading works (Read tool)
- [ ] ✅ File archiving works (Bash tool with mv command)
- [ ] ✅ Console messages display correctly with Rich panels
- [ ] ✅ Streaming responses work
- [ ] ✅ Tool usage works without default system prompt
- [ ] ✅ Documentation is up-to-date

### Standard Migration Success (Recommended)
Additionally required for production use:

- [ ] ✅ MCP integration works (Image Generation Zone with Replicate)
- [ ] ✅ MCP tool calls succeed without default system prompt
- [ ] ✅ Error handling unchanged
- [ ] ✅ Response quality matches pre-migration behavior

### Comprehensive Migration Success (Full Validation)
Additionally validated for complete confidence:

- [ ] ✅ All drop zones function correctly when enabled
- [ ] ✅ Gemini CLI agent integration works
- [ ] ✅ File operations across all zones (read, write, archive)
- [ ] ✅ Audio processing works (Morning Debrief with whisper)
- [ ] ✅ Data generation works (Training Data Zone)
- [ ] ✅ Financial analysis works (Finance Categorizer Zone)
- [ ] ✅ No performance degradation observed
- [ ] ✅ No behavioral changes in agent execution

---

## Post-Migration Steps

### Optional Enhancements

1. **Explore New SDK Capabilities**
   - Review Agent SDK documentation
   - Identify new features
   - Investigate custom tools opportunities

2. **Performance Monitoring**
   - Compare pre-migration vs. post-migration performance
   - Analyze token usage
   - Measure response times

3. **Documentation Enhancement**
   - Expand README.md with Agent SDK-specific examples
   - Update ai_docs/ directory
   - Expand example workflows

---

## Related Documentation

- **Migration Guide:** https://docs.claude.com/en/docs/claude-code/sdk/migration-guide
- **Agent SDK Overview:** https://docs.claude.com/en/docs/claude-code/sdk/sdk-overview
- **Python SDK Reference:** https://docs.claude.com/en/docs/claude-code/sdk/python
- **MCP Integration:** https://docs.claude.com/en/docs/claude-code/mcp-integration

---

## Changelog Template

```markdown
## [2.0.0] - 2025-11-01

### Changed
- Migrated from Claude Code SDK to Claude Agent SDK
- Updated package dependency: `claude-code-sdk` → `claude-agent-sdk`
- Updated Python imports: `claude_code_sdk` → `claude_agent_sdk`
- Renamed `ClaudeCodeOptions` → `ClaudeAgentOptions`

### Added
- **claude_code preset enabled by default** for backward compatibility
  - Ensures system prompt behavior remains identical to previous version
  - Prevents any behavioral changes during migration
  - Can be safely removed later if extensive testing confirms unnecessary

### Updated
- Documentation references to reflect new SDK name
- Code comments and console messages
- README.md with new package installation instructions

### Tested
- Verified tool usage works with `claude_code` preset enabled
- Confirmed MCP integration remains functional
- Validated all drop zone workflows operate correctly
- Ensured 100% backward compatibility with previous behavior

### Notes
- **No functional changes to drop zone behavior** - preset maintains exact previous behavior
- Custom prompts system unchanged
- MCP integration unchanged
- All existing workflows compatible
- System prompt preset ensures seamless migration without behavioral differences
```

---

## Approval and Sign-off

- [ ] **Technical Review:** _______________ (Date: _______)
- [ ] **Testing Approval:** _______________ (Date: _______)
- [ ] **Deployment Authorization:** _______________ (Date: _______)

---

**Last Updated:** 2025-11-01
**Version:** 2.0 (MAJOR REVISION)
**Created By:** Claude (Agentic Drop Zone Migration Analyzer)

---

## Summary of Migration Plan Updates

### Version 2.0 Changes (MAJOR REVISION - STRATEGY CHANGE)
- **🎯 CRITICAL DECISION:** `claude_code` preset now **REQUIRED BY DEFAULT** (not optional fallback)
- **Rationale:** Current codebase relies on automatic system prompt - removing it is a behavioral change, not just a rename
- **Updated:** Phase 4 now includes **4.2 - Add claude_code preset** as mandatory step
- **Updated:** Executive Summary - preset is default migration strategy
- **Updated:** "Contingency Plan" → "System Prompt Configuration Strategy" (preset is now default, not contingency)
- **Updated:** Analysis section 4 from "POTENTIALLY RELEVANT - REQUIRES TESTING" to "RELEVANT - ACTION REQUIRED"
- **Updated:** Implementation strategy - test WITH preset enabled, not without
- **Updated:** Changelog template to include preset addition
- **Impact:** Migration now ensures 100% backward compatibility by default

### Version 1.2 Changes (Preset Discovery)
- **🎯 BREAKTHROUGH:** Confirmed official `claude_code` preset availability via screenshot evidence
- **Updated:** Contingency Plan Option 2 now marked as "CONFIRMED AVAILABLE - RECOMMENDED"
- **Updated:** Risk assessment from "Medium-High" to "Low" for system prompt impact
- **Added:** TypeScript and Python code examples for preset usage
- **Updated:** Decision criteria to prioritize preset over custom system prompt
- **Added:** Recommended fallback order (preset → custom → enhance prompts)
- **Updated:** Executive Summary to reflect mitigated risk
- **Updated:** All risk mitigation sections with preset solution
- **Impact:** Migration confidence significantly increased, risk substantially reduced

### Version 1.1 Changes
- **Added:** Current project configuration status showing active/disabled zones
- **Updated:** Testing strategy to reflect only Echo Zone is currently active
- **Added:** Three-tiered testing approach (Minimal/Standard/Comprehensive)
- **Updated:** Timeline estimates for realistic testing scenarios (75-150 minutes)
- **Enhanced:** System prompt removal analysis from "NOT RELEVANT" to "POTENTIALLY RELEVANT"
- **Added:** Contingency plan with 3 options if system prompt issues arise
- **CONFIRMED:** Official `claude_code` preset available as one-line fallback solution
- **Updated:** Risk assessment from Medium to Low due to preset availability
- **Added:** Success criteria split into three levels (Minimal/Standard/Comprehensive)
- **Clarified:** MCP integration testing requires manually enabling zones
- **Added:** Specific instructions for enabling disabled zones for testing
- **Added:** Screenshot evidence from migration guide showing preset usage

### Key Decision Points
1. **Minimal Migration:** 80-85 minutes - Test only Echo Zone (currently active) with preset
2. **Standard Migration:** 95-105 minutes - Add MCP testing (requires enabling zone) with preset
3. **Comprehensive Migration:** 125-155 minutes - Test all zones (requires enabling all) with preset

**All paths include claude_code preset by default for 100% backward compatibility.**

### Migration Approach & Risk Mitigation
**Approach:** The migration includes the `claude_code` preset **by default** (Phase 4.2) to ensure 100% backward compatibility.

**Preset Implementation:**
```python
systemPrompt: {"type": "preset", "preset": "claude_code"}
```
This is **included in the migration** (not added after if issues arise).

**Rationale:**
- ✅ Current codebase relies on automatic system prompt
- ✅ Removing it is a behavioral change, not just a rename
- ✅ Preset ensures exact current behavior is maintained
- ✅ No uncertainty - migration is conservative and safe
- ✅ Can be removed later if extensive testing confirms unnecessary (but recommended to keep)

### Overall Migration Risk Assessment
- **Code Changes:** ✅ **Low Risk** (4 simple modifications - package, imports, type, preset)
- **Testing Effort:** ✅ **Minimal Path Available** (80-85 minutes with Echo Zone only)
- **System Prompt Impact:** ✅ **No Risk** (preset included by default ensures backward compatibility)
- **Backward Compatibility:** ✅ **100%** (preset maintains exact current behavior)
- **Rollback Capability:** ✅ **Excellent** (git branch + straightforward code changes)
- **Overall Risk:** ✅ **LOW** - Safe to proceed with high confidence

---

## Quick Reference: Claude Code Preset (INCLUDED BY DEFAULT)

The migration includes the `claude_code` preset **by default** (Phase 4.2) to ensure 100% backward compatibility.

**Location:** `sfs_agentic_drop_zone.py`, around line 258-273, in the `prompt_claude_code()` function

**After migration (preset enabled by default):**
```python
options_dict = {
    "permission_mode": "bypassPermissions",
    "systemPrompt": {"type": "preset", "preset": "claude_code"}  # ← ADDED IN PHASE 4.2
}

if args.model:
    options_dict["model"] = args.model

if args.mcp_server_file:
    options_dict["mcp_servers"] = args.mcp_server_file

options = ClaudeAgentOptions(**options_dict)
```

**Why it's included by default:**
- ✅ Maintains exact current behavior (100% backward compatibility)
- ✅ Prevents unexpected behavioral changes
- ✅ Ensures tool usage works identically to current implementation
- ✅ Official SDK feature, maintained by Anthropic
- ✅ Can be safely removed later if extensive testing confirms unnecessary (but recommended to keep)

**Optional: Removing the preset (only after extensive testing):**
If after several weeks of stable operation you want to test without the preset:
```python
options_dict = {
    "permission_mode": "bypassPermissions"
    # systemPrompt removed - testing without preset
}
```

**Recommendation:** Keep the preset enabled for maximum stability and compatibility.
