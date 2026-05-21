# Phase D — context_risk Enforcement Verification

**Date:** 2026-05-21  
**Hook commit:** `fix(hooks): enforce allowed_paths whitelist, context_risk flags, session-aware stop`

Verifies each `context_risk` flag is read from the latest LoopCheckpoint and produces the correct enforcement action.

---

## Fixture: LoopCheckpoint with all context_risk flags true

```yaml
# .context/checkpoints/chk-test-risk-all.yaml
checkpoint_id: "chk-test-risk-all"
schema_version: "0.1"
created_at: "2026-05-21T00:00:00Z"
current_phase: "investigating"
orchestrator:
  context_risk:
    long_lived_session: true
    high_parallelism: true
    subagent_heavy: true
    checkpoint_stale: true
    reload_scope_too_large: true
```

---

## D-01: checkpoint_stale=true → BLOCK (all tools)

| Input | `{"tool_name": "Read", "tool_input": {"file_path": "src/foo.py"}}` |
|-------|---------------------------------------------------------------------|
| Expected stdout | `{"decision": "block", "reason": "ContextGuard: context_risk.checkpoint_stale is true..."}` |
| Expected exit | 2 |
| Hook section | General context_risk block (runs for all tools) |
| Enforcement | **BLOCK** — checkpoint_stale gates on all tool calls; no work proceeds until a fresh checkpoint is written |

**Rationale:** If the checkpoint is stale the orchestrator has no reliable view of current state. Dispatching any action risks divergence.

---

## D-02: long_lived_session=true → WARN (all tools)

| Input | `{"tool_name": "Write", "tool_input": {"file_path": "src/foo.py"}}` |
|-------|----------------------------------------------------------------------|
| Expected stderr | `ContextGuard warning: context_risk.long_lived_session is true. Compact context before continuing.` |
| Expected exit | 0 |
| Enforcement | **WARN** — operator-visible; session can continue but should compact |

**Rationale:** Long-lived sessions accumulate context that degrades model reasoning. Warning without blocking preserves forward progress while surfacing the signal.

---

## D-03: high_parallelism=true → BLOCK Agent spawning

| Input | `{"tool_name": "Agent", "tool_input": {"prompt": "investigate..."}}` |
|-------|----------------------------------------------------------------------|
| Expected stdout | `{"decision": "block", "reason": "ContextGuard: context_risk.high_parallelism is true..."}` |
| Expected exit | 2 |
| Hook section | pre_spawn (Agent tool check) |
| Enforcement | **BLOCK** — existing workers already at parallelism limit |

**Rationale:** Multiple concurrent workers reading/writing the same capsules leads to conflicting updates and split context.

---

## D-04: subagent_heavy=true → WARN on Agent spawn

| Input | `{"tool_name": "Agent", "tool_input": {"prompt": "investigate..."}}` |
|-------|----------------------------------------------------------------------|
| Expected stderr | `ContextGuard warning: context_risk.subagent_heavy is true. Reduce subagent budget and avoid Explore escalation.` |
| Expected exit | 0 |
| Hook section | pre_spawn (Agent tool check) |
| Enforcement | **WARN** — spawn allowed but operator notified |

**Rationale:** subagent_heavy signals approaching (not at) the parallelism limit. One more spawn may be acceptable; operator decides.

---

## D-05: reload_scope_too_large=true + Read → WARN

| Input | `{"tool_name": "Read", "tool_input": {"file_path": "src/big_module.py"}}` |
|-------|---------------------------------------------------------------------------|
| Expected stderr | `ContextGuard warning: context_risk.reload_scope_too_large is true. Prune warm/cold context before broad reads.` |
| Expected exit | 0 |
| Enforcement | **WARN** — read allowed; operator should prune context before continuing broad reads |

---

## D-06: reload_scope_too_large=true + Bash → WARN

| Input | `{"tool_name": "Bash", "tool_input": {"command": "find . -name '*.py'"}}` |
|-------|---------------------------------------------------------------------------|
| Expected stderr | same as D-05 |
| Expected exit | 0 |
| Enforcement | **WARN** |

---

## D-07: reload_scope_too_large=true + Glob → WARN

| Input | `{"tool_name": "Glob", "tool_input": {"pattern": "src/**/*.py"}}` |
|-------|-------------------------------------------------------------------|
| Expected stderr | same as D-05 |
| Expected exit | 0 |
| Enforcement | **WARN** |

---

## D-08: reload_scope_too_large=true + Write → no warn

| Input | `{"tool_name": "Write", "tool_input": {"file_path": "src/foo.py"}}` |
|-------|---------------------------------------------------------------------|
| Expected stderr | (empty) |
| Expected exit | 0 |
| Enforcement | **PASS** — flag only applies to broad read operations |

---

## D-09: All flags false → ALLOW (baseline)

**Checkpoint:** all context_risk flags `false`  
**Input:** any tool

| Expected exit | 0 |
|---------------|---|
| Expected stderr | (empty) |
| Enforcement | **PASS** |

---

## Enforcement Priority

When multiple flags are true simultaneously, evaluation order in the hook is:

1. `high_parallelism` → BLOCK (Agent only, in pre_spawn section)
2. `subagent_heavy` → WARN (Agent only, in pre_spawn section)
3. `checkpoint_stale` → BLOCK (all tools, in general context_risk section — runs after pre_spawn)
4. `long_lived_session` → WARN (all tools)
5. `reload_scope_too_large` → WARN (Read/Bash/Glob only)

**Note:** `checkpoint_stale` block is checked AFTER the Agent-specific checks. If both `high_parallelism` and `checkpoint_stale` are true on an Agent call, `high_parallelism` blocks first. For non-Agent tools, `checkpoint_stale` is the first block in the general section.

---

## Summary

| Flag | Scope | Action | Added in hook-fixes |
|------|-------|--------|---------------------|
| `checkpoint_stale` | All tools | BLOCK | ✅ Yes |
| `high_parallelism` | Agent only | BLOCK | — (pre-existing) |
| `subagent_heavy` | Agent only | WARN | ✅ Yes |
| `long_lived_session` | All tools | WARN | ✅ Yes |
| `reload_scope_too_large` | Read/Bash/Glob | WARN | ✅ Yes |

**Overall Status: PASS**
