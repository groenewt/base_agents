---
name: telephone-beam-topology
description: Audit or modify Telephone BEAM supervision, listener, acceptor, router, and worker topology. Use when work touches socket ownership, acceptor handoff, process mailboxes, queue workers, or supervisor ordering.
---

# Telephone BEAM Topology

Start with these files:

```bash
lib/research_render/telephone/application.ex
lib/research_render/telephone/port.ex
lib/research_render/telephone/port/listener.ex
lib/research_render/telephone/port/acceptor_pool.ex
lib/research_render/telephone/port/queue_router.ex
lib/research_render/telephone/port/worker.ex
lib/research_render/telephone/port/worker_supervisor.ex
```

Required checks:

- Confirm application supervisor order: `Switchboard` before consumers,
  producers, `Port`, classifier/cross, `Dial`, `Archive.Auto`, and trace
  supervisor.
- Confirm Port boot order: listeners open first; internal supervisor starts
  `WorkerSupervisor`, `OverflowController`, `QueueRouter`, `AcceptorPool`;
  initial workers boot after router; acceptors boot last per listener.
- The default Port topology is sharded: `default`, `avro_in`, `avro_out`,
  `control`, `query`, `bulk_in`, and `audit_out`. `Producer.User` and `Dial`
  own separate `user` and `dial` sockets outside the Port shard list.
- Accepted sockets must have an explicit owner contract. The current JSON-line
  worker path is passive send/close without controlling-process handoff. Do not
  claim handoff safety unless a `:gen_tcp.controlling_process/2` path and
  failure test exist.
- Queue bounds are not mailbox bounds. Overload work must inspect GenServer call
  timeouts, mailbox growth, and worker concurrency, not only logical queue size.
- A02 found no direct mailbox bound assertion and no direct supervisor-order
  test; keep those caveats unless you add coverage.
- No topology change is complete without tests under
  `test/research_render/telephone/port/`.

Use these focused gates after topology changes:

```bash
mix test test/research_render/telephone/port
mix test test/research_render/telephone/port/listener_test.exs test/research_render/telephone/port/worker_supervisor_test.exs
```
