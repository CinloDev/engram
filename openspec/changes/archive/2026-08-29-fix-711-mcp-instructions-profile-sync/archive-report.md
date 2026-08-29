# Archive Report: Dynamic MCP Server Instructions & Profile Sync

**Change**: `fix-711-mcp-instructions-profile-sync`  
**Status**: **COMPLETED & ARCHIVED**  
**Date**: 2026-08-29  
**Artifact Store**: Hybrid (Engram + OpenSpec)  
**Branch**: `fix/711-mcp-instructions-deferred-tools`  
**Base Commit**: `2732d68` (HEAD~3)  
**Commits**:
- `3e0da5e`: fix(mcp): generate server instructions dynamically from tool allowlist (#711)
- `231306b`: fix(plugins): align hook scripts and skills with agent tool profile (#711)
- `c3227e5`: docs: update tool counts to 18 agent and 22 total tools (#711)

---

## Executive Summary

Issue #711 addressed false tool advertisement under `--tools=agent` where static MCP server instructions listed 4 admin-only tools (`mem_stats`, `mem_delete`, `mem_timeline`, `mem_merge_projects`). This change replaced the static instruction constant with dynamic filtering via `buildServerInstructions(allowlist map[string]bool) string`, enforced the 2048-rune client truncation limit across all tool profiles, synchronized plugin hook scripts and memory skills for Claude Code and OpenAI Codex, and updated project documentation to reflect 18 agent tools and 22 total tools.

---

## Final-State Facts & Metrics

- **Tasks**: 9/9 complete (100%) across 3 phases.
- **Verification**: **PASS** (Zero criticals, zero warnings, 100% spec & scenario compliance).
- **RDD Audit**: Approved on lineage `review-75d88129d66050ad` (4 lenses completed: defect/regression, architecture/boundary, test/coverage, docs/alignment; burned, delivery unmanaged).
- **Test Suite & Coverage**:
  - `internal/mcp` unit and bounds test suite: 100% PASS.
  - Statement coverage in `internal/mcp`: 88.4%.
  - Full internal test suite (`go test ./internal/... ./cmd/...`): PASS.
  - Shell syntax validation (`bash -n` on all plugin scripts): PASS.
- **Handshake Bounds Safety**:
  - `all (nil)`: 1,487 runes (1,505 bytes) < 2,048 limit.
  - `agent`: 1,430 runes (1,448 bytes) < 2,048 limit.
  - `admin`: 184 runes (184 bytes) < 2,048 limit.
  - `empty`: 80 runes (80 bytes) < 2,048 limit.
  - `custom`: 854 runes (862 bytes) < 2,048 limit.
- **Changeset Size**: 22 files changed (~891 lines total including SDD artifacts, ~220 lines authored source/tests/docs).
- **Remaining Blockers**: None.

---

## Traceability & Memory Artifacts

| Phase Artifact | Engram Observation ID | Sync ID | Openspec Path |
|---|---|---|---|
| Exploration | #1570 | `obs-d1d50ec220829e72` | `explore.md` |
| Proposal | #1571 | `obs-23f520f514cfd07f` | `proposal.md` |
| Delta Spec | #1572 | `obs-62eafbc7ef2257ed` | `specs/mcp-profile-instructions/spec.md` |
| Design | #1573 | `obs-a3fdb502dbb4f5a3` | `design.md` |
| Tasks | #1574 | `obs-bc00745a972aa1ce` | `tasks.md` |
| Apply Progress | #1575 | `obs-147af541aa964874` | `apply-progress.md` |
| Verify Report | #1576 | `obs-b1540940c5943de5` | `verify-report.md` |

---

## Specs Synced

| Domain | Action | Details | Source of Truth |
|---|---|---|---|
| `mcp-profile-instructions` | Created Main Spec | Added 4 requirements (REQ-MPI-001 through REQ-MPI-004) covering dynamic instruction builder, profile partitioning, hook/skill tool parity, and documentation alignment. | `openspec/specs/mcp-profile-instructions/spec.md` |

---

## Mechanical Audit & Verification

- **Spec Copy Readback**: Mechanical shell copy via temporary file and `diff -r` validation returned exit status 0 (no diff).
- **Archive Move Readback**: Recursive pre-move snapshot and post-move comparison via `diff -r` returned exit status 0 (no diff).
- **Task Completion Gate**: 9/9 tasks checked in `tasks.md`.
- **Zero Critical Issues**: Clean verification verdict with no blocking defects.
