# ContextGuard Adapter Contract

Every ContextGuard adapter must implement the four lifecycle hooks defined here. The hooks may be shell scripts, language bindings, or runtime-specific integrations — the interface is behavioral, not syntactic.

---

## Hook: `pre_action`

**Trigger:** Before any tool call or action  
**Purpose:** Load active capsule state, check lease validity, inject capsule summary

### Required behavior

1. Locate the active capsule: read `guard.capsule_path` from `.context/config.yaml` (default: `.context/active/`)
2. If `guard.require_capsule: true` and no active capsule exists → **block** with message: `"No active capsule. Create or load an InvestigationCapsule before proceeding."`
3. If an active capsule exists and `lease.expires_at` is set and now > `expires_at` → **block** with message: `"Lease expired at <expires_at>. Checkpoint and escalate before continuing."`
4. If `context_risk.long_lived_session: true` in the latest checkpoint → **warn** and require compaction before proceeding
5. If `context_risk.checkpoint_stale: true` → **warn** and require checkpoint refresh before dispatch
6. If capsule loaded successfully → inject capsule summary as context (tool-dependent implementation)

### Inputs

The adapter receives from the runtime:
- `tool_name`: string — name of the tool being called
- `tool_input`: object — parameters passed to the tool

### Output

- Allow: proceed with tool call
- Block: halt tool call, surface reason to operator
- Warn: proceed but surface warning

---

## Hook: `pre_write`

**Trigger:** Before any Write, Edit, or mutation tool call (Write, Edit, Bash with write operations)  
**Purpose:** Enforce `worker_scope.forbidden_paths` and `worker_scope.allowed_paths`

### Required behavior

1. Identify the target path(s) from `tool_input`
2. Load `worker_scope.forbidden_paths` from the active `WorkerHandoff` (if one exists in `.context/handoffs/`)
3. If target path matches any `forbidden_paths` entry → **block** with message: `"Path <path> is forbidden by active worker scope."`
4. If `worker_scope.allowed_paths` is non-empty and target path does not match any entry → **block** with message: `"Path <path> is outside allowed scope."`
5. If `worker_scope.mutation_policy: read_only` → **block** all write operations

### Path matching

Use prefix matching: a forbidden path of `.console/tmp/` blocks writes to `.console/tmp/anything`.

---

## Hook: `pre_spawn`

**Trigger:** Before any subagent spawn or Agent tool call  
**Purpose:** Enforce subagent budget from active lease

### Required behavior

1. Load active `WorkerHandoff` from `.context/handoffs/` (if exists)
2. If `lease.max_subagents: 0` → **block** subagent spawn with message: `"Lease prohibits subagent spawning."`
3. If current subagent count >= `lease.max_subagents` → **block** with message: `"Subagent budget exhausted (<count>/<max>)."`
4. If `context_risk.high_parallelism: true` in latest checkpoint → **block** additional worker spawning
5. If `context_risk.subagent_heavy: true` → **warn** and reduce effective subagent budget by 50%

---

## Hook: `on_stop`

**Trigger:** On session or worker termination  
**Purpose:** Require checkpoint write and capsule update before the session ends

### Required behavior

1. Check if `.context/checkpoints/` has been written to during this session
2. If no checkpoint was written → **warn** (or block, depending on `loop.checkpoint_on_stop`) with message: `"Session ending without a checkpoint. Write a LoopCheckpoint before terminating."`
3. If an active capsule exists and `status` is still `active` → **warn**: `"Active capsule not updated. Update status or handoff_notes before terminating."`
4. If `loop.checkpoint_on_stop: true` and no checkpoint written → **block** termination

---

## Configuration

Adapters read configuration from `.context/config.yaml`. Required fields:

```yaml
guard:
  require_capsule: false    # bool — block tool calls with no active capsule
  enforce_lease: true       # bool — block actions after lease expires
  capsule_path: ".context/active/"
  checkpoint_path: ".context/checkpoints/"
  handoff_path: ".context/handoffs/"

loop:
  checkpoint_on_stop: true  # bool — require checkpoint before session ends
```

If `.context/config.yaml` does not exist, adapters should use the defaults above and emit a one-time warning.

---

## Exit Codes (shell adapters)

| Exit code | Meaning          |
| --------- | ---------------- |
| 0         | Allow            |
| 1         | Warn (non-fatal) |
| 2         | Block (fatal)    |

For runtimes that support structured output (e.g., Claude Code JSON hooks), use:

```json
{"decision": "block", "reason": "..."}
```

---

## Compliance

An adapter is compliant if:

- [ ] All four hooks are implemented
- [ ] `pre_action` reads `.context/config.yaml` on each invocation (not cached indefinitely)
- [ ] `pre_write` uses prefix path matching
- [ ] `on_stop` warns on missing checkpoint when `checkpoint_on_stop: true`
- [ ] No private paths, escalation tiers, or ecosystem-specific logic are hardcoded
- [ ] Adapter has a `README.md` describing installation and configuration
