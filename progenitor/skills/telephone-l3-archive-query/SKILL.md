---
name: telephone-l3-archive-query
description: Inspect or verify Telephone L3 archive behavior, Parquet/ORC export, archive manifests, readback completeness, and DuckDB query projections.
---

# Telephone L3 Archive Query

Use this when work touches archive files, `cheese_call` fallback queries, or
Parquet/ORC export.

Relevant files:

```bash
lib/research_render/telephone/archive.ex
lib/research_render/telephone/archive/auto.ex
lib/research_render/telephone/query.ex
bin/cheese_call
archive/
```

Checks:

- Parquet is the verified L3 primary. ORC is an optional sidecar export; ORC
  readback is still explicit as `blocked:orc_readback_not_supported`.
- Distinguish conversion smoke tests from retained archive artifacts.
- Verify row counts from source Avro to archive output and manifest fields
  `source_row_count`, `row_count`, and `parity_status`; never count CSV headers
  or emitted status lines as records.
- JSON query output must be ANSI-free and must fail loudly on query failures.
- Readback results need completeness metadata when spanning L0/L1/L2/L3.
- A12 can be green for L3 archive parity while A17 remains blocked by sequence
  ledger and L2 Arrow cache.

Useful commands:

```bash
cheese_call --last 20 --format json | jq '.records | length'
mix research.telephone.archive --format parquet
mix research.telephone.archive --format parquet --orc-sidecar
mix test test/research_render/telephone/archive_test.exs test/research_render/telephone/archive/auto_test.exs
scripts/verify-telephone-storage-broker.sh
```
