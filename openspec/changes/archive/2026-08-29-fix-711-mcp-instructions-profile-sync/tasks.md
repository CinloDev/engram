# Tasks: Dynamic MCP Server Instructions & Profile Sync

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | ~180-260 lines |
| 400-line budget risk | Low |
| Chained PRs recommended | No |
| Suggested split | Single PR (3 atomic work units) |
| Delivery strategy | ask-on-risk |
| Chain strategy | stacked-to-main |

Decision needed before apply: No
Chained PRs recommended: No
Chain strategy: stacked-to-main
400-line budget risk: Low

### Suggested Work Units

| Unit | Goal | Likely PR | Focused test command | Runtime harness | Rollback boundary |
|------|------|-----------|----------------------|-----------------|-------------------|
| 1 | Dynamic instruction builder & length bounds | PR 1 | `go test -v ./internal/mcp -run "TestServerInstructions"` | `engram mcp --tools=agent` (handshake instructions inspection) | `internal/mcp/mcp.go`, `internal/mcp/*test.go` |
| 2 | Plugin hooks and skill tool sync | PR 1 | `bash -n plugin/claude-code/scripts/*.sh && bash -n plugin/codex/scripts/*.sh` | N/A (static hook and skill scripts) | `plugin/claude-code/**`, `plugin/codex/**` |
| 3 | Documentation tool counts & full suite validation | PR 1 | `go test -cover ./...` | N/A (documentation alignment) | `docs/*.md`, `internal/mcp/mcp.go` comments |

## Phase 1: Dynamic Instructions Implementation (TDD)

- [x] 1.1 (RED) Add unit tests in `internal/mcp/mcp_test.go` verifying `buildServerInstructions` filters CORE/DEFERRED tools and conditionally toggles `## CONFLICT SURFACING`.
- [x] 1.2 (RED) Add bounds tests in `internal/mcp/serverinstructions_length_test.go` verifying `utf8.RuneCountInString < 2048` across `nil`, `agent`, `admin`, empty, and custom allowlists.
- [x] 1.3 (GREEN) Implement `buildServerInstructions` in `internal/mcp/mcp.go` and wire into `newServerWithActivity`.
- [x] 1.4 (VERIFY) Run `go test -v ./internal/mcp/...` ensuring dynamic generation, bounds, and regression tests pass.

## Phase 2: Plugin Hooks & Skills Parity

- [x] 2.1 Update `plugin/claude-code/scripts/session-start.sh` and `post-compaction.sh` to remove admin tools and include active agent tools.
- [x] 2.2 Update `plugin/codex/scripts/session-start.sh` and `post-compaction.sh` to align tool listings.
- [x] 2.3 Update `plugin/claude-code/skills/memory/SKILL.md` and `plugin/codex/skills/memory/SKILL.md` to reflect the 18 agent tools.

## Phase 3: Documentation & Verification

- [x] 3.1 Update tool counts (18 agent, 22 total) in `internal/mcp/mcp.go` header comments, `docs/AGENT-SETUP.md`, `docs/PLUGINS.md`, and `docs/DOCS.md`.
- [x] 3.2 Run full test suite (`go test ./...`) and check package coverage (`go test -cover ./internal/mcp/...`).
