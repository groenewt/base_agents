---
name: closure-master-verify
description: Run the Phase 14 master_verify aggregator (20-gate × 4-dim matrix, pure wire reader). Use when the user asks to run master verify, get the green/fail/pending matrix, or check phase certification state.
allowed-tools: Bash(mix research.master_verify *), Bash(cheese_call *)
---

# closure-master-verify — Phase 14 master verification aggregator

## Precondition

```!
~/.local/share/telephone/bootstrap || { echo "ABORT: bootstrap_failed exit=$?"; exit 1; }
```

## Run

```!
cd $(dirname $(readlink ~/.local/share/telephone))/.. && mix research.master_verify
```

## Output

20 × 4 matrix (gate × dim ∈ {pass, profile, trace, drift}). Final line:

```
summary: fully_green=N pending=N fail=N drift=N empty_trace=N malformed=N error=N
```

Exit 0 iff all 80 cells are `ok`. Non-zero exit on pending/fail/drift is by design — that's the model telling the truth about what producers emitted onto the wire.

## Owned gates (8)

| # | Gate | Producer |
|---|---|---|
| 1, 2 | compile + test | `mix research.gates` |
| 3 | marker scan | `mix research.check.marker_scan` |
| 6 | yoneda manifests | `mix research.check.yoneda_manifests` |
| 10 | LinkML 32-generator | `mix research.check.linkml_32` |
| 11 | Hermit grounding | `mix research.check.hermit_grounding` |
| 12 | O8 supervision (UDS roundtrip) | `mix research.check.o8` |
| 20 | Sou quorum + honeycomb | `mix research.check.quorum_and_honeycomb` |

## Upstream gates (12)

Gates 4, 5, 7–9, 13–19. Pending until `/working/scripts/*.sh` emits its wire contract. Honest pending; not a defect.

## Interpreting fails

When a gate is failing, read the wire detail field BEFORE assuming wiring vs world state:

```bash
cheese_call --event gate_<n>_passed --last 1 --format json | jq -r '.records[0].detail'
```

Cross-check against external ground truth (`pgrep`, file presence, cross-tool validation). Don't label a fail "honest" without that reconciliation step.

## Discipline

Per `feedback_closure_detail_field_discipline`: never claim `fully_green=N` without reading every non-green detail field. Wiring bugs disguised as honest fails are this session's most-repeated pattern; the only break is reading detail and reconciling.
