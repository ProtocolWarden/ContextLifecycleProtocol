# Phase A — OperationsCenter Context Wiring

**Repo:** OperationsCenter  
**Date:** 2026-05-21  
**Branch:** feat/contextguard-hook-fixes (hooks synced from CLP#5)

---

## 1. Cognition Surface Presence

| Path | Present | Note |
|------|---------|------|
| `.context/config.yaml` | ✅ | OC-specific config |
| `.context/active/` | ✅ | Active capsule dir |
| `.context/checkpoints/` | ✅ | Checkpoint dir |
| `.context/handoffs/` | ✅ | Handoff dir |
| `.context/templates/` | ✅ | Templates present |
| `.context/schemas/` | ✅ | Schemas present |
| `.context/examples/` | ✅ | `oc_watchdog_checkpoint.yaml` |
| `.claude/hooks/pre_tool_use.sh` | ✅ | Synced from CLP adapter |
| `.claude/hooks/stop.sh` | ✅ | Synced from CLP adapter |
| `.claude/settings.json` | ✅ | Hook registration |

---

## 2. Hook Registration (`.claude/settings.json`)

```json
{
  "hooks": {
    "PreToolUse": [{ "matcher": ".*", "hooks": [{ "type": "command", "command": "bash .claude/hooks/pre_tool_use.sh" }] }],
    "Stop":       [{ "hooks": [{ "type": "command", "command": "bash .claude/hooks/stop.sh" }] }]
  }
}
```

- PreToolUse: matches all tools (`.*`)
- Stop: fires on session end

---

## 3. Config Verification (`.context/config.yaml`)

| Setting | Value | Expected |
|---------|-------|----------|
| `guard.require_capsule` | `false` | Capsule not required for all ops (correct — OC manages capsule lifecycle itself) |
| `guard.enforce_lease` | `true` | ✅ Lease expiry enforced |
| `loop.checkpoint_on_stop` | `true` | ✅ Stop hook will surface missing checkpoint |
| Worker `mutation_policy` | `write_allowed` | ✅ |
| Worker `allowed_paths` | `.context/`, `.console/`, `src/`, `tests/` | ✅ |
| Worker `forbidden_paths` | `.console/tmp/`, `.context/tmp/` | ✅ |
| Worker `allowed_subagents` | `1` | ✅ Budget enforced |

---

## 4. CLAUDE.md Wiring

File: `OperationsCenter/CLAUDE.md`

Confirms:
- `<!-- console-context -->` / `<!-- /console-context -->` fencing present
- "Cognition Lifecycle" section below the fence describes OC orchestrator lifecycle
- Instructs model to read `.context/checkpoints/<latest>.yaml` on wake
- Instructs model to write LoopCheckpoint on session end

---

## 5. Hook Sync Status

Hooks at `.claude/hooks/` are identical copies of CLP `adapters/claude/hooks/`.  
Synced at commit `fix(hooks): sync ContextGuard hook fixes from CLP adapter`.  
PR: ProtocolWarden/OperationsCenter#157

---

## Status: PASS
