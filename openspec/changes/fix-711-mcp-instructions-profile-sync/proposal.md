# Proposal: Dynamic MCP Server Instructions & Profile Sync

## Intent

Eliminate false tool advertisements and discovery failures when clients connect with `--tools=agent` by generating MCP server instructions dynamically from active tool allowlists and aligning plugin hooks, skills, and documentation.

## Scope

### In Scope
- Dynamic instruction builder `buildServerInstructions(allowlist map[string]bool) string` in `internal/mcp/mcp.go`.
- Dynamic filtering of CORE vs DEFERRED tools and conditional `## CONFLICT SURFACING` inclusion.
- Enforcement and tests for 2048-rune client handshake limit across all profile permutations (`nil`/all, `agent`, `admin`, custom).
- Tool list updates in plugin hook scripts (`plugin/claude-code/scripts/`, `plugin/codex/scripts/`) and memory skills.
- Tool count documentation alignment (18 agent tools, 22 total tools) across package comments and docs.

### Out of Scope
- Modifying tool schemas, arguments, or execution logic.
- Adding new MCP tools or removing existing tools.
- Altering project resolution or storage mechanics.

## Capabilities

### New Capabilities
- `mcp-profile-instructions`: Dynamic generation and synchronization of MCP server instructions according to the active tool profile/allowlist.

### Modified Capabilities
None

## Approach

Implement `buildServerInstructions(allowlist map[string]bool) string` in `internal/mcp/mcp.go`:
- Filter CORE (`mem_save`, `mem_search`, `mem_context`, `mem_session_summary`, `mem_get_observation`, `mem_save_prompt`, `mem_current_project`) and DEFERRED (`mem_update`, `mem_review`, `mem_pin`, `mem_unpin`, `mem_suggest_topic_key`, `mem_session_start`, `mem_session_end`, `mem_doctor`, `mem_compare`, `mem_capture_passive`, plus admin tools `mem_stats`, `mem_delete`, `mem_timeline`, `mem_merge_projects`) using `shouldRegister`.
- Conditionally append `## CONFLICT SURFACING` block only when `mem_judge` is registered.
- Pass result to `server.WithInstructions` in `newServerWithActivity`.
- Retain package-level `serverInstructions` constant/variable for backward compatibility.

### Alternatives Considered
- **Static Template Selection**: Multiple profile-specific constants. Rejected for lack of flexibility with custom allowlists and tripled maintenance overhead.
- **Static Agent-Only String**: Hardcoding agent profile instructions. Rejected because it breaks full/admin profile tool advertising.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `internal/mcp/mcp.go` | Modified | Add `buildServerInstructions`, update server initialization, correct comments |
| `internal/mcp/serverinstructions_length_test.go` | Modified | Validate 2048-rune limit across all profiles |
| `internal/mcp/mcp_test.go` | Modified | Unit tests for instruction generation, section filtering, and conflict surfacing |
| `plugin/claude-code/scripts/*` | Modified | Update tool lists in `session-start.sh` and `post-compaction.sh` |
| `plugin/codex/scripts/*` | Modified | Update tool lists in `session-start.sh` and `post-compaction.sh` |
| `plugin/{claude-code,codex}/skills/memory/SKILL.md` | Modified | Sync deferred tool guidance |
| `docs/{AGENT-SETUP,PLUGINS,DOCS}.md` | Modified | Align tool counts (18 agent, 22 total) |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Client truncation at 2048 runes | Med | Test `utf8.RuneCountInString < 2048` for all profile configurations |
| Existing test regressions | Low | Maintain compatibility with existing instruction assertions |
| Shell script syntax regressions | Low | Verify bash and PowerShell scripts across platforms |

## Rollback Plan

Revert git commits modifying `internal/mcp/`, `plugin/`, and `docs/`. The static `serverInstructions` baseline remains safe to restore.

## Dependencies

- Repo skills: `engram-architecture-guardrails`, `engram-business-rules`, `engram-memory-protocol`, `engram-project-structure`, `engram-testing-coverage`.

## Success Criteria

- [ ] `buildServerInstructions` dynamically produces instructions advertising only registered tools.
- [ ] No admin tools advertised under `--tools=agent`.
- [ ] `utf8.RuneCountInString(instructions) < 2048` verified for all profile permutations.
- [ ] Plugin hook scripts and memory skills accurately reflect active agent tools.
- [ ] All tests pass (`go test ./...`) with 100% adherence to strict TDD.
