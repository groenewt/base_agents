---
name: telephone-fountain-erasure-readiness
description: Plan or audit future Telephone erasure-coding readiness. Use for profiles disabled|host|factory|global|edge, GF(2^8), ramp/fountain simulation, block/share manifests, or native ISA-L/io_uring boundaries.
---

# Telephone Fountain Erasure Readiness

Erasure work is future-gated. It must not affect runtime ACK semantics until
L0-L3 durability, sequence ledgers, and manifests are green.

Required boundaries:

- Profiles are `disabled`, `host`, `factory`, `global`, and `edge`; default must
  be `disabled`.
- Pure simulation comes before native acceleration.
- Block/share manifests must include segment identity plus fd/inode/device
  provenance when those facts are claimed.
- Block/share manifests are data-only readiness records. They must carry an
  explicit future gate and `runtime_ack_effect=none`, and no dispatcher or ACK
  path may consume them before the durability gates are green.
- A23 native work is forbidden until pure tests pass and performance data proves
  the need.
- Existing manifest builders are pure data builders under
  `lib/telephone/storage/erasure_manifest.ex`; they validate profile names,
  threshold shape, checksums, byte sizes, and provenance fields, but they do not
  prove boundary-captured fd/inode/device facts.

Suggested artifacts:

```bash
research/specs/data/collectors/telephone_erasure_readiness_inventory.md
research/specs/data/research/telephone_dag_manifest/manifest.csv
priv/telephone/schemas/erasure_block_manifest.avsc
priv/telephone/schemas/erasure_share_manifest.avsc
test/telephone/codec/erasure_simulation_property_test.exs
test/telephone/storage/erasure_manifest_test.exs
```

Do not wire erasure into `Port.Dispatcher` ACK behavior during readiness work.

Gate notes:

- A20-A23 are research/deferred nodes in the 24-node manifest. They remain
  blocked from runtime until A17 storage/broker and A19 performance blockers are
  resolved.
- A20 is profile/ontology readiness only, A21 is pure GF simulation, A22 is
  block/share manifest provenance, and A23 is native acceleration boundary. Do
  not merge these into one green claim.
- External citations for Avro block structure, deferred-sync caveats, and
  io_uring development/fallback status are recorded in
  `research/specs/data/collectors/telephone_erasure_readiness_inventory.md`.
