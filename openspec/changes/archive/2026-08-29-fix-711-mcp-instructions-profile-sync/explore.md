# Exploration: fix-711-mcp-instructions-profile-sync

## Current State

Engram exposes tools to AI agents and clients via the Model Context Protocol (MCP). Tools are partitioned into profiles (`ProfileAgent` with 18 tools and `ProfileAdmin` with 4 tools, totaling 22 tools in `internal/mcp/mcp.go`).

When clients connect with `--tools=agent` (which is configured by default in `plugin/claude-code/.mcp.json`, `plugin/codex/.mcp.json`, and setup generators), `NewServerWithConfig` filters tool registration using `shouldRegister` against the `ResolveTools("agent")` allowlist.

However, the server instructions attached via `server.WithInstructions(serverInstructions)` at `internal/mcp/mcp.go:225` use a single hardcoded string constant `serverInstructions`. This constant:
1. Hardcodes `DEFERRED TOOLS` to include 4 admin tools (`mem_stats`, `mem_delete`, `mem_timeline`, `mem_merge_projects`) from `ProfileAdmin`, which are NOT registered when running `--tools=agent`.
2. Omits newer agent-profile tools (`mem_judge`, `mem_compare`, `mem_doctor`) from the deferred tools list.
3. Is passed unconditionally to every MCP server instance regardless of the allowlist/profile.

Consequently, any client running with `--tools=agent` receives instructions advertising admin tools that do not exist on the server, causing confusing tool discovery failures when agents attempt to invoke them.

Furthermore:
- Plugin hook scripts (`plugin/claude-code/scripts/session-start.sh:153`, `post-compaction.sh:60`, `plugin/codex/scripts/session-start.sh:142`, `post-compaction.sh:46`) and memory skills (`plugin/claude-code/skills/memory/SKILL.md:26`, `plugin/codex/skills/memory/SKILL.md:25`) duplicate this discrepancy by directing agents to use ToolSearch for `mem_stats`, `mem_delete`, `mem_timeline`.
- Package and setup comments/documentation across `internal/mcp/mcp.go`, `docs/AGENT-SETUP.md`, and `docs/PLUGINS.md` cite stale counts ("15 agent tools", "19 total tools") instead of current counts (18 agent tools, 22 total tools).

## Affected Areas

- `internal/mcp/mcp.go` — Replace static `serverInstructions` constant attachment with a dynamic instruction generator `buildServerInstructions(allowlist map[string]bool) string` in `newServerWithActivity`; correct comments and tool counts.
- `internal/mcp/serverinstructions_length_test.go` — Test that dynamic instructions for `nil` (all), `ProfileAgent`, `ProfileAdmin`, and custom sets strictly stay under the 2048-rune client truncation limit.
- `internal/mcp/mcp_test.go` — Add unit tests for `buildServerInstructions` ensuring accurate filtering, correct sections, and preservation of conflict surfacing rules.
- `plugin/claude-code/scripts/session-start.sh` & `post-compaction.sh` — Update tool lists to accurately reflect active agent tools and remove admin tools.
- `plugin/codex/scripts/session-start.sh` & `post-compaction.sh` — Update tool lists to reflect active agent tools.
- `plugin/claude-code/skills/memory/SKILL.md` & `plugin/codex/skills/memory/SKILL.md` — Remove false admin tool pointers from agent-scoped memory skills.
- `docs/AGENT-SETUP.md` & `docs/PLUGINS.md` & `DOCS.md` — Update stale tool count references (15 → 18 agent tools, 19 → 22 total tools).

## Approaches

### 1. Dynamic Instruction Builder (`buildServerInstructions(allowlist map[string]bool)`)
Generate the instruction string dynamically during server initialization based on the active `allowlist`.
- **Pros**:
  - 100% accurate advertising for any profile combination (`agent`, `admin`, `all`) or custom tool subset.
  - Automatically omits un-registered tools from both CORE and DEFERRED sections.
  - Automatically conditionally includes `## CONFLICT SURFACING` block only when `mem_judge` is registered.
  - Clean separation of concerns and single source of truth for tool listings.
- **Cons**:
  - Requires string builder / formatting logic in `internal/mcp/mcp.go`.
- **Effort**: Low-Medium

### 2. Static Template Selection by Named Profile
Maintain static string constants for `all`, `agent`, and `admin`, and select the constant based on the profile name.
- **Pros**:
  - Avoids string generation logic at runtime.
- **Cons**:
  - Fragile: Does not support custom allowlists or mixed combinations (e.g. `--tools=agent,admin` or `--tools=mem_save,mem_search`).
  - Triples static constant maintenance overhead.
- **Effort**: Medium

### 3. Static Agent-Only Instructions
Replace `serverInstructions` with a static string strictly tailored to `ProfileAgent`.
- **Pros**:
  - Minimal code change in `mcp.go`.
- **Cons**:
  - Incomplete: When running in full/admin mode (`engram mcp` or `--tools=admin`), instructions will omit admin tools.
- **Effort**: Low

## Recommendation

**Approach 1 (Dynamic Instruction Builder)** is recommended.
- Implement `buildServerInstructions(allowlist map[string]bool) string` (or `BuildServerInstructions`) in `internal/mcp/mcp.go`.
- Structure the instructions into:
  1. Header (persistent memory overview).
  2. `CORE TOOLS (always available — use without ToolSearch)` listing registered core tools (`mem_save`, `mem_search`, `mem_context`, `mem_session_summary`, `mem_get_observation`, `mem_save_prompt`, `mem_current_project`).
  3. `DEFERRED TOOLS (use ToolSearch when needed)` listing registered deferred tools (e.g. `mem_update`, `mem_review`, `mem_pin`, `mem_unpin`, `mem_suggest_topic_key`, `mem_session_start`, `mem_session_end`, `mem_capture_passive`, `mem_doctor`, `mem_compare`, and admin tools only if allowed).
  4. `PROACTIVE SAVE RULE`.
  5. `## CONFLICT SURFACING` (included whenever `mem_judge` is registered).
- Retain a package-level constant or variable `serverInstructions` for backward compatibility in tests while making `newServerWithActivity` pass `buildServerInstructions(allowlist)`.
- Update plugin scripts, skills, and documentation to align with the agent profile.

## Risks

- **2048 Rune Truncation Limit**: The Claude Code MCP client truncates handshake instructions at 2048 runes. All permutations of generated instructions (`nil`, `agent`, `admin`, custom) must be tested to ensure `utf8.RuneCountInString(instructions) < 2048`.
- **Test Invariants**: Existing tests (`TestServerInstructions_ConflictSurfacingBlock`, `TestServerInstructionsUsesCandidateJudgmentIDs`, `TestServerInstructionsConstantIsNonEmpty`) must continue to pass across all default and agent configurations.
- **Hook Scripts Parity**: Changes to bash and ps1 scripts must maintain exact quoting, syntax, and behavior across Unix and Windows environments.

## Ready for Proposal

Yes. The root cause, architectural boundaries, blast radius, and approach are verified against the codebase. The orchestrator can proceed with the `propose` phase for `fix-711-mcp-instructions-profile-sync`.
