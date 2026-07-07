---
name: setup-bootstrap
description: Establish telephone wire ready-state from any state (daemon up, daemon down, sockets stale, PATH symlink missing). Use when a fresh agent comes up, when daemon-up is uncertain, or before any downstream wire operation.
allowed-tools: Bash
---

# setup-bootstrap — wire readiness primitive

Run the bootstrap script — idempotent, cold-start safe. It's the precondition for every other telephone-wire skill and agent.

```!
~/.local/share/telephone/bootstrap
```

## Named exit codes

| code | reason | action |
|---|---|---|
| 0 | ready | proceed |
| 10 | `socket_never_appeared` | daemon failed to bind UDS within 10s — check `/tmp/telephone-bootstrap-*.log` |
| 11 | `binary_missing` | a cheese_* binary not on PATH — symlink broken |
| 12 | `schema_invariant_mismatch` | avsc SHA drift; REFUSE TO PROCEED, surface immediately |
| 13 | `repo_missing` / `cannot_cd` | fundamental host setup gone |
| 15 | `telephone_home_unset` | symlink + `$TELEPHONE_HOME` both absent — needs `make -f Makefile.setup install` |

## Discovery (no hardcoded paths)

`REPO=$(dirname $(readlink ~/.local/share/telephone))` — the symlink target IS the truth.

If the symlink is absent, set `$TELEPHONE_HOME` to the repo path before running bootstrap, OR run `make -f Makefile.setup install` to create the symlink.

## What "ready" guarantees

After exit 0:
- Daemon process alive (pgrep finds it)
- UDS socket `<repo>/logs/.telephone.sock` listening
- All 6 cheese_* binaries resolvable from PATH
- Avro schema invariant matches `<repo>/meta/schema_invariant.sha256`
- A `hook_setup_bootstrap` wire record was emitted (warm or cold)
