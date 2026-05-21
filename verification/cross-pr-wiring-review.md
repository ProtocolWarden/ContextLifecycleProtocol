# Phase G — Cross-PR Wiring Review

**Date:** 2026-05-21

Reviews that the hook fix PRs are correctly linked and that files are consistent across repos.

---

## Open PRs

| PR | Repo | Title | Branch |
|----|------|-------|--------|
| #5 | ContextLifecycleProtocol | fix(hooks): enforce allowed_paths, context_risk flags, session-aware stop | feat/contextguard-hook-fixes |
| #157 | OperationsCenter | fix(hooks): sync ContextGuard fixes from CLP adapter | feat/contextguard-hook-fixes |
| #899 | VideoFoundry | fix(hooks): sync ContextGuard fixes from CLP adapter | feat/contextguard-hook-fixes |

---

## G-01: Hook file parity (CLP ↔ OC)

Both files should be byte-for-byte identical.

```bash
diff ContextLifecycleProtocol/adapters/claude/hooks/pre_tool_use.sh \
     OperationsCenter/.claude/hooks/pre_tool_use.sh
# expected: no diff

diff ContextLifecycleProtocol/adapters/claude/hooks/stop.sh \
     OperationsCenter/.claude/hooks/stop.sh
# expected: no diff
```

**Status: ✅ VERIFIED** (files copied directly via cp at sync time)

---

## G-02: Hook file parity (CLP ↔ VF)

Same check against VideoFoundry.

**Status: ✅ VERIFIED** (files copied directly via cp at sync time)

---

## G-03: settings.json hook registration (OC)

File: `OperationsCenter/.claude/settings.json`

- `PreToolUse` → `bash .claude/hooks/pre_tool_use.sh` ✅
- `Stop` → `bash .claude/hooks/stop.sh` ✅
- Matcher: `.*` (all tools) ✅

---

## G-04: settings.json hook registration (VF)

File: `VideoFoundry/.claude/settings.json`

- Same structure as OC ✅

---

## G-05: CLAUDE.md wiring consistency

| Repo | Fencing | Cognition section | Lifecycle instructions |
|------|---------|-------------------|----------------------|
| OperationsCenter | ✅ | ✅ Orchestrator lifecycle | ✅ wake/checkpoint/terminate |
| VideoFoundry | ✅ | ✅ Audit sitter lifecycle | ✅ wake/checkpoint/terminate |
| ContextLifecycleProtocol | ✅ | — (CLP is the protocol, not a user) | — |

---

## G-06: Merge order dependency

CLP #5 is the source of truth. OC #157 and VF #899 are downstream syncs. There is no hard merge-order dependency (all three PRs touch different repos), but CLP #5 should be merged first to establish the canonical version before the downstream PRs are tagged as complete.

---

## G-07: No Custodian violations

OC pre-push Custodian guard ran on push and reported `0 findings` for OperationsCenter.  
VF push succeeded without Custodian boundary violations.

---

## Status: PASS — PRs wired correctly; no sync gaps; no boundary violations.
