# Design: Dynamic MCP Server Instructions & Profile Sync

## Technical Approach

Implement dynamic MCP server instruction generation via `buildServerInstructions(allowlist map[string]bool) string` in `internal/mcp/mcp.go`. The builder inspects the active tool allowlist using `shouldRegister(name, allowlist)` to partition registered tools into CORE and DEFERRED sections and conditionally attach the `## CONFLICT SURFACING` block when `mem_judge` is present. `newServerWithActivity` passes the dynamically generated string to `server.WithInstructions`. Plugin hook scripts, skills, and documentation are aligned with the 18 agent tools and 22 total tools.

## Architecture Decisions

| Option | Tradeoff | Decision |
|--------|----------|----------|
| **Dynamic Instruction Builder** | Small string generation cost on server startup | **Chosen**: Single source of truth, supports arbitrary custom allowlists and standard profiles (`agent`, `admin`, `all`). |
| Static per-profile constants | Rigid, cannot support custom tool allowlists (`--tools=mem_save,mem_search`) | **Rejected**: Inflexible and duplicates prose across constants. |
| Agent-only static string | Breaks full and admin profiles | **Rejected**: Violates profile contracts for admin/custom use cases. |

## Sequence Diagram

```
Client/CLI             MCP Initialization         buildServerInstructions         mcp-go Server
    │                           │                            │                          │
    ├── ResolveTools(flag) ────►│                            │                          │
    │   (generates allowlist)   │                            │                          │
    │                           ├── newServerWithActivity ───┼─────────────────────────►│
    │                           │   (allowlist)              │                          │
    │                           │                            │                          │
    │                           ├── buildServerInstructions(allowlist)                  │
    │                           │   (partitions tools, cond. conflict surfacing)       │
    │                           │◄───────────────────────────┘                          │
    │                           │                                                       │
    │                           ├── server.WithInstructions(instructions) ─────────────►│
    │                           │                                                       │
    │                           ├── registerTools(allowlist) ──────────────────────────►│
    │                           │                                                       │
    ◄── Handshake (instructions < 2048 runes) ──────────────────────────────────────────┘
```

## Data Flow & Profile Handling

| Profile / Allowlist | CORE Section | DEFERRED Section | CONFLICT SURFACING |
|---------------------|--------------|------------------|--------------------|
| `nil` / `all` (22 tools) | 7 core tools | 11 agent deferred + 4 admin tools | Included (`mem_judge` present) |
| `agent` (18 tools) | 7 core tools | 10 agent deferred tools | Included (`mem_judge` present) |
| `admin` (4 tools) | Omitted (0 core registered) | 4 admin tools (`mem_delete`, `mem_stats`, `mem_timeline`, `mem_merge_projects`) | Omitted (`mem_judge` absent) |
| Custom allowlist | Registered core tools only | Registered deferred/admin tools only | Included iff `mem_judge` present |

## Package Boundaries

All instruction generation and profile resolution logic resides strictly within `internal/mcp`. No exported types or cross-package couplings are introduced.

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `internal/mcp/mcp.go` | Modify | Add `buildServerInstructions`, update `newServerWithActivity`, retain `serverInstructions` variable, update doc comments (18 agent, 22 total) |
| `internal/mcp/serverinstructions_length_test.go` | Modify | Assert `utf8.RuneCountInString < 2048` across `nil`, `agent`, `admin`, empty, and custom allowlists |
| `internal/mcp/mcp_test.go` | Modify | Unit tests verifying core/deferred tool filtering and conditional conflict surfacing |
| `plugin/claude-code/scripts/session-start.sh` | Modify | Remove admin tools from deferred prompt; include active agent tools |
| `plugin/claude-code/scripts/post-compaction.sh` | Modify | Align deferred tool guidance |
| `plugin/codex/scripts/session-start.sh` | Modify | Remove admin tools from deferred prompt; include active agent tools |
| `plugin/codex/scripts/post-compaction.sh` | Modify | Align deferred tool guidance |
| `plugin/claude-code/skills/memory/SKILL.md` | Modify | Align tool lists with 18 agent tools |
| `plugin/codex/skills/memory/SKILL.md` | Modify | Align tool lists with 18 agent tools |
| `docs/{AGENT-SETUP,PLUGINS,DOCS}.md` | Modify | Align tool counts (18 agent tools, 22 total tools) |

## Implementation Details

```go
func buildServerInstructions(allowlist map[string]bool) string {
    var b strings.Builder
    b.WriteString("Engram provides persistent memory that survives across sessions and compactions.\n\n")

    // Filter CORE tools
    core := filterTools(coreToolList, allowlist)
    if len(core) > 0 {
        b.WriteString("CORE TOOLS (always available — use without ToolSearch):\n")
        b.WriteString("  " + strings.Join(coreDescriptions(core), "\n  ") + "\n\n")
    }

    // Filter DEFERRED tools
    deferred := filterTools(deferredToolList, allowlist)
    if len(deferred) > 0 {
        b.WriteString("DEFERRED TOOLS (use ToolSearch when needed):\n")
        b.WriteString("  " + strings.Join(deferred, ", ") + "\n\n")
    }

    if shouldRegister("mem_save", allowlist) {
        b.WriteString("PROACTIVE SAVE RULE: Call mem_save immediately after ANY decision, bug fix, discovery, or convention — not just when asked.\n\n")
    }

    if shouldRegister("mem_judge", allowlist) {
        b.WriteString("## CONFLICT SURFACING\n\nAfter mem_save: if judgment_required...")
    }

    return strings.TrimSpace(b.String())
}
```

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| Unit | `buildServerInstructions` per profile | Test output strings for `nil`, `ProfileAgent`, `ProfileAdmin`, empty map, and custom toolsets in `mcp_test.go` |
| Bounds | Rune length ceiling (< 2048 runes) | Test `utf8.RuneCountInString < 2048` across all profile permutations in `serverinstructions_length_test.go` |
| Regression | Backward compatibility | Verify `serverInstructions` variable retains baseline format and judgment assertions |

## Threat Matrix

`N/A — no routing, subprocess execution changes, VCS/PR automation, executable-file classification, or process-integration boundary changes (plugin script changes are static heredoc text updates).`

## Migration / Rollout

No migration required. Server handshake and plugin updates are backward-compatible.

## Open Questions

None.
