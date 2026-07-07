---
name: closure-phase-closure
description: End-to-end phase-closure workflow — quality gates, owned-gate producers, master_verify, closure-log update. User-triggered only (disable-model-invocation prevents Claude from running it autonomously).
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, Grep, Glob
---

# closure-phase-closure — end-to-end phase closure

## Precondition

```!
~/.local/share/telephone/bootstrap || { echo "ABORT: bootstrap_failed exit=$?"; exit 1; }
```

## Workflow (halt on first hard failure)

```bash
REPO=$(dirname $(readlink ~/.local/share/telephone))
cd "$REPO"
```

### 1. Carryforward quality gates

```bash
mix compile --warnings-as-errors
mix credo --strict
mix research.lint
mix dialyzer
mix test
mix research.telephone.e2e_render
```

If any fails: STOP. Report the diagnostic verbatim. Do not paper over.

### 2. Owned-gate producer runs (emits 4-record wire contracts)

```bash
mix research.gates                          # gates 1, 2
mix research.check.marker_scan              # gate 3
mix research.check.yoneda_manifests         # gate 6
mix research.check.linkml_32                # gate 10
mix research.check.hermit_grounding         # gate 11
mix research.check.o8                       # gate 12 (UDS roundtrip)
mix research.check.quorum_and_honeycomb     # gate 20
```

### 3. Master verify

```bash
mix research.master_verify
```

### 4. Wire-data freshness check (per `feedback_closure_detail_field_discipline`)

```bash
for n in 1 2 3 6 10 11 12 20; do
  ts=$(cheese_call --event "gate_${n}_passed" --last 1 --format json 2>/dev/null \
       | grep -oE '"time_iso":"[^"]*"' | head -1)
  echo "gate_${n}: $ts"
done
```

Every owned gate's `time_iso` must post-date the most recent code change. If stale, re-run that producer before quoting the master_verify summary.

### 5. Invariant verification

```bash
expected_schema=$(tr -d '[:space:]' < meta/schema_invariant.sha256)
actual_schema=$(sha256sum logs/research_render_event.avsc | cut -c1-64)
[ "$expected_schema" = "$actual_schema" ] || echo "ALARM: schema drift"

expected_cheese=$(tr -d '[:space:]' < meta/cheese_md_invariant.sha256)
actual_cheese=$(sha256sum working/cannon/cheese.md | cut -c1-64)
[ "$expected_cheese" = "$actual_cheese" ] || echo "ALARM: cheese.md drift"
```

### 6. Anti-shim audit

```bash
grep -rn -E "TODO|FIXME|XXX|stub|shim|UNIMPLEMENTED|placeholder|mock_only|not[_ ]implemented" \
  lib/research_render/observability/ lib/research_render/sou/ \
  lib/research_render/honeycomb/ lib/research_render/check/ \
  lib/mix/tasks/research.master_verify.ex \
  lib/mix/tasks/research.check.* bin/sou-* honeycomb/ 2>/dev/null \
  | grep -v "# rationale:" | grep -v "_test.exs:"
```

Expected: 0 unjustified matches.

### 7. Closure-log append

New dated section to `logs/closure/phase<NN>_<date>.md`:
- Updated matrix output
- Wire-freshness timestamps per owned gate
- Files-edited list (additive)
- Criterion ledger §41–58
- Any superseded prior claim, named explicitly with supersession reason

## Discipline (non-negotiable)

- Read every non-green detail field before claiming a green count
- Never relabel a fail as pass via a clever frame ("honest fail", "criterion-bug corrected", "by-design")
- No edits to `/working/***`, `logs/research_render_event.avsc`, or git commands
- The wire is the Yoneda witness — emit new records that name what happened, never retroactively pass-stamp
