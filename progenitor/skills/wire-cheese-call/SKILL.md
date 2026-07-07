---
name: wire-cheese-call
description: Query recent wire records by event name and/or count. Use when the user asks what fired recently, wants to inspect a specific event, or needs to trace what producers emitted in a time window.
allowed-tools: Bash
arguments: [event_name, last_n]
argument-hint: "[event_name] [last_n]"
---

# wire-cheese-call — query wire records

## Precondition

```!
~/.local/share/telephone/bootstrap || { echo "ABORT: bootstrap_failed exit=$?"; exit 1; }
```

## Run

Query last N records for an event:

```bash
cheese_call --event "$0" --last "${1:-10}" --format text
```

With no arguments, default to 10 most recent records across all events:

```bash
cheese_call --last 10 --format text
```

Query one scope from the live L0 socket path:

```bash
cheese_call --scope repo --last 10 --format json
```

Routing contract:

- With no historical/event/handle/correlation/Dunbar/alarm filters, `cheese_call`
  prefers L0 over the live UDS and returns an envelope with `route_selected`,
  `tiers_consulted`, `tiers_available`, `completeness_status`, `count`,
  `records`, and `client_filters`.
- L0 socket discovery tries the requested socket, then `control`, then `query`
  when the requested socket is the default `logs/.telephone.sock`.
- Historical and filtered reads use the Mix query path first, then direct DuckDB
  over `logs/hooks/*.avro` if Mix is unavailable.
- Direct DuckDB output is a projection; payload may be reported as an absent
  union placeholder when older Avro rows drift from the current schema.

## Useful patterns

```bash
# Confirm provider-neutral tool hooks alive
cheese_call --event hook_tool_use --last 5

# Read a gate's most recent record (detail field has the actionable)
cheese_call --event gate_12_passed --last 1 --format json | jq -r '.records[0].detail'

# Watch the wire live (Ctrl-C to stop)
cheese_call --watch --format text

# Filter by dunbar layer
cheese_call --dunbar-layer-leq 1 --last 20

# Alarm-immediate records only (layer 2 bypass)
cheese_call --alarm-immediate-only --last 50

# Machine-readable output; must be ANSI-free and fail on invalid formats
cheese_call --last 10 --format json | jq '.records | length'

# Query by correlation id or handle; filters also inspect payload attributes and detail KV
cheese_call --correlation-id "$CORR" --last 200 --format json
cheese_call --handle "$HANDLE" --last 50 --format json

# Fan out per-handle client reads and merge by time_iso desc
cheese_call --batch --handles alpha,beta --last 25 --format json
```

## Discipline (per `feedback_closure_detail_field_discipline`)

When investigating a failed gate or anomalous event: **read the detail field** before classifying. A reason atom like `:supervisor_not_running` while the daemon is demonstrably alive is a wiring bug, not honest world state. Reconcile against external ground truth (`pgrep`, file presence, cross-tool validation) before labeling.

If `cheese_call --format json` exits nonzero, report the failure. Do not
replace it with `{"records":[]}`; an empty `records` array is only valid when
the command itself succeeds.

`--scope` is part of the live L0 `op=call` frame and is also applied by the CLI
before JSON/text emission, so scoped reads must not leak records from another
stage. Malformed L0 responses are command errors; do not treat a missing
`records` array as an empty result.

For gate work, preserve the envelope metadata. `hot_partial` from L0 is not an
archive-complete claim, and a direct DuckDB route is not evidence that the live
daemon path was reachable.
