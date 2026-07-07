---
name: wire-telephone-overload-verify
description: Verify Telephone overload and backpressure behavior. Use when testing queue lanes, high/low water marks, reject/drop/block/elastic policies, worker saturation, or mailbox growth.
---

# Wire Telephone Overload Verify

Primary files:

```bash
lib/research_render/telephone/port/queue_router.ex
lib/research_render/telephone/port/overflow_controller.ex
lib/research_render/telephone/port/worker_supervisor.ex
scripts/verify-telephone-storage-broker.sh
scripts/verify-telephone-performance.sh
test/research_render/telephone/port/queue_router_test.exs
test/research_render/telephone/port/overflow_controller_test.exs
test/research_render/telephone/port/worker_supervisor_test.exs
```

Verification discipline:

- Test both policy output and observable wire/readback behavior.
- High-water events should fire before unbounded queue or mailbox growth.
- `reject`, `drop`, `block`, and `elastic` must have distinct counters or
  structured details.
- JSON CLI checks must use `cheese_call --format json`; do not parse styled
  text for gates.
- Low-water resume behavior matters for `:block_acceptor` lanes; high-water
  evidence without resume evidence is partial.
- A16 overload/backpressure can pass as unit policy while A19 remains blocked
  because runtime counters are unavailable.

Run:

```bash
mix test test/research_render/telephone/port/queue_router_test.exs
mix test test/research_render/telephone/port/overflow_controller_test.exs
mix test test/research_render/telephone/port/worker_supervisor_test.exs
cheese_call --event hook_wire_overload --last 20 --format json
scripts/verify-telephone-storage-broker.sh
scripts/verify-telephone-performance.sh
```

Gate notes:

- A16 is covered by the storage/broker focused gate only when queue router,
  overflow controller, and worker supervisor tests are present and pass.
- A19 must stay non-green while `queue_depth`, `fallback_count`,
  `mailbox_summary`, or `ocf_bytes_per_sec` proof remains unavailable. Latency
  samples alone are not overload proof.
- JSON CLI checks remain mandatory; styled text output is not acceptable gate
  evidence.
