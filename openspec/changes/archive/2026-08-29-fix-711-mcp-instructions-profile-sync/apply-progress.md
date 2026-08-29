# Apply Progress: fix-711-mcp-instructions-profile-sync

## Implementation Overview

- **Change**: `fix-711-mcp-instructions-profile-sync`
- **Mode**: Strict TDD
- **Delivery Strategy**: ask-on-risk (single PR, Low risk)
- **Status**: All 9 tasks across Phases 1-3 completed.

## Completed Tasks

### Phase 1: Dynamic Instructions Implementation (TDD)
- [x] 1.1 (RED) Add unit tests in `internal/mcp/mcp_test.go` verifying `buildServerInstructions` filters CORE/DEFERRED tools and conditionally toggles `## CONFLICT SURFACING`.
- [x] 1.2 (RED) Add bounds tests in `internal/mcp/serverinstructions_length_test.go` verifying `utf8.RuneCountInString < 2048` across `nil`, `agent`, `admin`, empty, and custom allowlists.
- [x] 1.3 (GREEN) Implement `buildServerInstructions` in `internal/mcp/mcp.go` and wire into `newServerWithActivity`.
- [x] 1.4 (VERIFY) Run `go test -v ./internal/mcp/...` ensuring dynamic generation, bounds, and regression tests pass.

### Phase 2: Plugin Hooks & Skills Parity
- [x] 2.1 Update `plugin/claude-code/scripts/session-start.sh` and `post-compaction.sh` to remove admin tools and include active agent tools.
- [x] 2.2 Update `plugin/codex/scripts/session-start.sh` and `post-compaction.sh` to align tool listings.
- [x] 2.3 Update `plugin/claude-code/skills/memory/SKILL.md` and `plugin/codex/skills/memory/SKILL.md` to reflect the 18 agent tools.

### Phase 3: Documentation & Verification
- [x] 3.1 Update tool counts (18 agent, 22 total) in `internal/mcp/mcp.go` header comments, `docs/AGENT-SETUP.md`, `docs/PLUGINS.md`, and `docs/DOCS.md`.
- [x] 3.2 Run full test suite (`go test ./...`) and check package coverage (`go test -cover ./internal/mcp/...`).

## TDD Cycle Evidence

| Task | Test File | Layer | Safety Net | RED | GREEN | TRIANGULATE | REFACTOR |
|------|-----------|-------|------------|-----|-------|-------------|----------|
| 1.1 | `internal/mcp/mcp_test.go` | Unit | ✅ Pass | ✅ Written | ✅ Passed | ✅ 4 profiles/cases | ✅ Clean |
| 1.2 | `internal/mcp/serverinstructions_length_test.go` | Bounds | ✅ Pass | ✅ Written | ✅ Passed | ✅ 5 profiles/cases | ✅ Clean |
| 1.3 | `internal/mcp/mcp.go` | Unit/Integration | ✅ Pass | ✅ Written | ✅ Passed | ✅ All suites pass | ✅ Clean |
| 2.1 | `plugin/claude-code/scripts/*.sh` | Hook/Script | ✅ bash -n | ✅ Tested | ✅ Passed | ✅ Full parity | ✅ Clean |
| 2.2 | `plugin/codex/scripts/*.sh` | Hook/Script | ✅ bash -n | ✅ Tested | ✅ Passed | ✅ Full parity | ✅ Clean |
| 2.3 | `plugin/*/skills/memory/SKILL.md` | Skill/Doc | N/A (doc) | ✅ Inspected | ✅ Aligned | ✅ 18 agent tools | ✅ Clean |
| 3.1 | `docs/*.md`, `cmd/engram/main.go` | Doc/CLI | N/A (doc) | ✅ Inspected | ✅ Aligned | ✅ 18/22 counts | ✅ Clean |
| 3.2 | Full test suite (`go test ./...`) | Regression | ✅ Full suite | N/A | ✅ Passed | ✅ 88.4% mcp cov | ✅ Clean |

## Work Unit Evidence

| Evidence | Required value |
|---|---|
| Focused test command and exact result | `go test -v ./internal/mcp -run "TestServerInstructions\|TestBuildServerInstructions"` → PASS (all tests pass) |
| Runtime harness command/scenario and exact result | `bash -n plugin/claude-code/scripts/*.sh && bash -n plugin/codex/scripts/*.sh` → exit 0 (clean syntax) |
| Rollback boundary | `git checkout -- internal/mcp/ plugin/ docs/ cmd/engram/main.go DOCS.md` |

## Test Summary
- **Total tests written**: 6 unit/bounds test functions (across multiple subtests and profile scenarios)
- **Total tests passing**: All tests passing
- **Layers used**: Unit (5), Bounds (1), Regression (full suite)
- **Approval tests**: None — no refactoring tasks
- **Pure functions created**: 1 (`buildServerInstructions`)
