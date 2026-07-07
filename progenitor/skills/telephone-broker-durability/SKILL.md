---
name: telephone-broker-durability
description: Verify or design Telephone durability semantics across L0 ring, L1 Avro, sequence ledger, ACK/NACK, replay, and readback. Use when the user asks whether an emit is durable, crash-safe, replayable, or complete.
---

# Telephone Broker Durability

Treat L0 and L1 separately:

- L0 `Switchboard` is volatile fanout and recent cache. It must not be described
  as durable storage.
- L1 Avro OCF append is the first durability boundary. ACK semantics may claim
  durability only when the append/sync result is on the client response path and
  write-failure/NACK behavior is tested.
- Current sequence values are writer-local unless a global ledger/manifest is
  present. Do not infer cross-file monotonicity from per-file tests.
- Replay must state whether it replays original records or synthetic
  `hook_replayed` records.
- A17 is currently non-green while `sequence_ledger:global_ledger_absent` and
  `l2_arrow_cache:arrow_ipc_cache_absent` remain blocked, even when focused
  L0/L1/replay/overload/archive tests pass.

Useful files:

```bash
lib/research_render/telephone/switchboard.ex
lib/research_render/telephone/consumer/ocf_appender.ex
lib/research_render/telephone/ocf_writer.ex
lib/research_render/telephone/replay.ex
lib/research_render/telephone/port/dispatcher.ex
```

Minimum gates for durability claims:

```bash
mix test test/research_render/telephone/switchboard_test.exs
mix test test/research_render/telephone/ocf_writer_test.exs
mix test test/research_render/telephone/replay_test.exs
scripts/verify-telephone-storage-broker.sh
```

If ACK/NACK is changed, add a crash/write-failure test proving the client does
not receive `ok` before the claimed boundary. If `cheese_text` or another CLI
falls back to direct write after a daemon NACK, that is masking the NACK and is
not a durable broker pass.

Gate notes:

- A17 storage/broker status is not green while `sequence_ledger` or
  `l2_arrow_cache` remains blocked. Passing focused tests is only local support
  for those tested modules, not a durability certification.
- Missing tests are blockers, not skipped green rows. The gate detail should
  include `blocked_items` and emit a non-`ok` status.
- Parquet L3 row parity can be locally green while global broker durability is
  still blocked by sequence/L2.
- External citation atoms for ACK and sequence caveats are in
  `research/specs/data/research/telephone_dag_manifest/source.md`.
