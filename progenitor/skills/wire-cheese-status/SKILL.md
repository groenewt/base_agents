---
name: wire-cheese-status
description: Daemon health snapshot — pid, uptime, socket state, OCF file count, total wire-record count, archive size. Use when the user asks "is the wire up?", "daemon status?", or wants a quick health check.
allowed-tools: Bash
---

# wire-cheese-status — daemon health snapshot

## Precondition

```!
~/.local/share/telephone/bootstrap || { echo "ABORT: bootstrap_failed exit=$?"; exit 1; }
```

## Run

```!
cheese_status
```

For gate scripts, use JSON and keep stderr separate:

```bash
cheese_status --format json | jq .
```

## Output shape

```
[telephone status]
  daemon:  up pid=<N> uptime=<MM:SS>
  socket:  up logs/.telephone.sock
  socket[default]: up logs/.telephone.sock codec=json_line direction=bidirectional probe=status
  socket[avro_in]: up logs/.telephone.avro_in.sock codec=avro_streaming direction=inbound probe=presence
  socket[avro_out]: up logs/.telephone.avro_out.sock codec=avro_streaming direction=outbound probe=presence
  socket[control]: up logs/.telephone.control.sock codec=json_line direction=bidirectional probe=status
  socket[query]: up logs/.telephone.query.sock codec=json_line direction=bidirectional probe=status
  socket[bulk_in]: up logs/.telephone.bulk_in.sock codec=avro_streaming direction=inbound probe=presence
  socket[audit_out]: up logs/.telephone.audit_out.sock codec=avro_streaming direction=outbound probe=presence
  socket[user]: up logs/.telephone.user.sock codec=json_line direction=inbound probe=presence
  socket[dial]: up logs/.telephone.dial.sock codec=json_line direction=bidirectional probe=presence
  L1 OCF:  <N> files, <size>
  records: <N> total, last=<ISO timestamp>
  archive: <N> parquet file(s)
```

JSON output preserves the legacy `socket` string and adds a structured
`sockets[]` matrix plus `paths.sockets` for gate consumers.

The daemon line may also report `up via socket_protocol path=<socket>
(pgrep_no_match)`. Treat that as usable live protocol evidence when the status
probe succeeds. A missing default socket is not automatically fatal if
`control` or `query` responds, but it is still a topology finding that must be
reported.

## Interpretation

- `daemon: down` -> wire is offline; bootstrap should have caught this, but if it didn't, the bootstrap script itself is broken
- `socket: up` but no recent records -> producers may be silent; check `cheese_call --last 10`
- Missing any of `socket[avro_in]`, `socket[avro_out]`,
  `socket[control]`, `socket[query]`, `socket[bulk_in]`, or
  `socket[audit_out]` → the Port shard matrix did not come up; inspect
  `ResearchRender.Telephone.Port` supervision before treating the default
  socket as complete
- Missing `socket[user]` or `socket[dial]` -> user producer or RPC server surface
  is incomplete even if the seven Port shards are up
- OCF count growing without bound -> archive sweep isn't running; check the `Telephone.Archive.Auto` supervisor child
- Schema SHA still locked: run `sha256sum logs/research_render_event.avsc | cut -c1-64` and compare to `<repo>/meta/schema_invariant.sha256` — these must match
- JSON output must decode cleanly and contain no ANSI escape sequences. Treat
  invalid `--format` as a command error, not as a text fallback.
- On very large hook stores, `CHEESE_STATUS_DUCKDB_TIMEOUT_SECONDS=<N>` bounds
  the record-count probe; timeout is reported in the `records` field rather
  than hanging the health check.
