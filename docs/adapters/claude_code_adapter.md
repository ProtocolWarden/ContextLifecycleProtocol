# Claude Code Adapter

The Claude Code adapter implements the ContextGuard contract using Claude Code's PreToolUse and Stop hooks.

## Architecture

```
.claude/
  settings.json        ← hook registration
  hooks/
    pre_tool_use.sh    ← pre_action + pre_write + pre_spawn
    stop.sh            ← on_stop
```

Hooks receive JSON from Claude Code on stdin and communicate via exit codes and stdout/stderr.

## Hook Behavior

### pre_tool_use.sh

Runs before every tool call. Implements three checks:

**1. Capsule presence** (`pre_action`)  
If `guard.require_capsule: true` and no `.yaml` file exists in `guard.capsule_path` → blocks with reason.

**2. Lease expiry** (`pre_action`)  
If an active `WorkerHandoff` exists in `guard.handoff_path` with `lease.expires_at` set and now > expires_at → blocks with reason.

**3. Forbidden path** (`pre_write`)  
On Write/Edit tool calls: if target path matches any `worker_scope.forbidden_paths` prefix → blocks. If `mutation_policy: read_only` → blocks all writes.

**4. Subagent budget** (`pre_spawn`)  
On Agent tool calls: if `lease.max_subagents: 0` → blocks. If `context_risk.high_parallelism: true` in latest checkpoint → blocks.

**5. context_risk warnings** (`pre_action`)  
If `context_risk.long_lived_session: true` in latest checkpoint → warns (non-blocking).

### stop.sh

Runs when the Claude Code session ends. Implements `on_stop`:

- If no checkpoint file exists in `guard.checkpoint_path` and `loop.checkpoint_on_stop: true` → warns with instructions
- If active capsule status is still `active` → warns to update before terminating

## Limitations

- Stop hooks cannot hard-block session termination in all Claude Code configurations — warnings are surfaced prominently but may not prevent exit
- The adapter detects checkpoint presence by file count, not by session-specific timestamps
- `jq` and `python3` with `pyyaml` must be available in the shell environment
- Path matching uses bash prefix comparison — glob patterns in `forbidden_paths` are not supported in this version

## Testing

```bash
# Test pre_tool_use with a Write call
echo '{"tool_name": "Write", "tool_input": {"file_path": ".console/tmp/test.txt"}}' \
  | bash adapters/claude/hooks/pre_tool_use.sh

# Test with a forbidden path (requires active handoff with forbidden_paths set)
# Should exit 2 with block message

# Test stop hook
bash adapters/claude/hooks/stop.sh
# Should warn if no checkpoint exists
```
