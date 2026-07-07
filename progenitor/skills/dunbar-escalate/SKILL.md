---
name: dunbar-escalate
description: Promote a critical event to dunbar layer 2 (alarm-immediate). Use when schema drift, gate fail with hard-blocking conditions, protected-path write attempted, or daemon failure detected. The layer-2 surface bypasses Dunbar filtering and reaches every subscriber.
allowed-tools: Bash(cheese_text *)
---

# dunbar-escalate — layer-2 alarm producer

## Precondition

```!
~/.local/share/telephone/bootstrap || { echo "ABORT: bootstrap_failed exit=$?"; exit 1; }
```

## When to escalate (layer 2 / alarm-immediate)

The layer-2 surface exists in `cheese_text` (the `--alarm-immediate` flag) but historically had no producer. This skill is that producer. Use for **genuine alarms**, not informational events:

1. **Schema drift** — `sha256sum <repo>/logs/research_render_event.avsc` ≠ contents of `<repo>/meta/schema_invariant.sha256`
2. **Cheese.md drift** — same check against `cheese_md_invariant.sha256`
3. **Protected-path write attempted** — Edit/Write tool call targeting `working/***` or `*.avsc`
4. **Hard gate fail** — predicate fail under conditions that should NEVER happen on a healthy host (not e.g. "Hermit absent" which is expected world state at layer 0)
5. **Daemon failure with retry expected** — `pgrep` returns nothing AND `cheese_call --last 1` timeout-fails AND bootstrap also fails

## How to emit

```bash
cheese_text "alarm_${TRIGGER_KIND}_${TARGET}" \
  --scope alarm \
  --event hook_dunbar_layer2_alarm \
  --dunbar-layer 2 \
  --alarm-immediate \
  --detail "trigger=${TRIGGER_KIND};target=${TARGET};at=$(date -Iseconds);correlation=${CLAUDE_SESSION_ID:-unknown}"
```

The `--alarm-immediate` flag bypasses dunbar-layer filtering and propagates to ALL subscribers regardless of their layer subscription.

## Discipline

- **Use sparingly.** Every layer-2 emission consumes attention budget across all consumers. Reserve for genuine alarms.
- **Predicate-fail with contract-emission is NOT a layer-2 alarm.** That's expected world state at layer 0 (e.g. gate 11 failing because Hermit isn't installed — that's a known actionable, not a host alarm).
- **One layer-2 record per discrete trigger.** Don't loop or batch. Schema-drift detected once → one alarm. Subsequent re-detection of the same drift → no new emission unless the value changed.
- **Cross-check against ground truth before escalating.** If you're about to emit `:supervisor_not_running` but `pgrep` shows the daemon alive, you have a wiring bug, not a host alarm. Investigate the wiring before consuming the layer-2 budget.
