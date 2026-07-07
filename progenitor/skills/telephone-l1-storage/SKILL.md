---
name: telephone-l1-storage
description: Work on Telephone L1 Avro OCF storage, segment naming, sync/rotate policy, schema fingerprinting, and append/readback verification. Use when files under logs/hooks, OCF writer, or L1 persistence are involved.
---

# Telephone L1 Storage

Primary files:

```bash
lib/research_render/telephone/ocf_writer.ex
lib/research_render/telephone/consumer/ocf_appender.ex
lib/research_render/telephone/producer/avro_spool.ex
lib/research_render/io/avro_codec.ex
logs/research_render_event.avsc
```

Storage rules:

- Preserve schema compatibility with `logs/research_render_event.avsc`.
- Segment naming must be deterministic and include enough context to avoid
  writer collisions.
- If a file descriptor, inode, device, checksum, or schema fingerprint is
  claimed, capture it from the actual write boundary and test it.
- Do not make runtime ACK durability claims unless the appender result is on the
  client response path.
- `OcfAppender` broadcast persistence must use committed append/readback
  behavior; unsynced append is not a storage pass.
- Per-file sequence ranges do not prove cross-file monotonicity. Global sequence
  claims require the A07 ledger/manifest.

Validation:

```bash
mix test test/research_render/telephone/ocf_writer_test.exs
mix test test/research_render/telephone/consumer/ocf_appender_test.exs
mix research.lint
mix compile --warnings-as-errors
```

Gate notes:

- `scripts/verify-telephone-storage-broker.sh` is A17. It may pass focused L0/L1,
  replay, overload, archive, and L2 smoke groups while still exiting nonzero as
  `blocked:storage_broker_incomplete` when sequence ledger or L2 cache remains
  absent.
- `scripts/verify-telephone-performance.sh` is A19. It must not report `ok` when
  queue depth, fallback count, mailbox summary, or OCF bytes/sec proof is
  unavailable; those are required measurement surfaces, not optional decoration.
- The current 24-node gate/blocker map lives at
  `research/specs/data/research/telephone_dag_manifest/gate_blocker_mapping.csv`.
