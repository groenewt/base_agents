---
name: wire-audit
description: Run the 7-tier global setup certification DAG — verifies daemon, PATH, hooks (global + repo), git observer, context files, and agent surface. Each tier emits its own hook_setup_tier_<n> wire record; orchestrator aggregates into hook_setup_summary. Use when the user asks to verify host setup, certify the wire, or audit the Codex/agent install.
context: fork
agent: Explore
allowed-tools: Bash, Read
---

# wire-audit — 7-tier setup certification DAG

## Precondition

```!
~/.local/share/telephone/bootstrap || { echo "ABORT: bootstrap_failed exit=$?"; exit 1; }
```

## Run the audit

The orchestrator script fires 7 tier inspections, each emitting its disjoint
surface verdict to the wire.

```bash
REPO=$(dirname $(readlink ~/.local/share/telephone))
"$REPO/scripts/audit-dag.sh"
```

## Tier matrix

| # | Surface | Pass criterion |
|---|---|---|
| 1 | Daemon + storage | daemon up, required UDS protocol surface live, ≥1 OCF, schema SHA matches manifest |
| 2 | PATH + binaries | symlink target correct, in PATH, Telephone/cheese command surface resolves + executable |
| 3 | Global hooks | UserPromptSubmit + Stop wired; **per-binary flag check** (cheese_surface accepts only --scope/--last; cheese_text accepts explicit --dunbar-layer only for escalation) |
| 4 | Repo hooks | SessionStart + UserPromptSubmit + PostToolUse + Stop wired; same per-binary criterion |
| 5 | Git pre-commit observer | executable, contains cheese_text call, emits hook_pre_commit_observed, layer-0 |
| 6 | Context + memory | CLAUDE.md + AGENTS.md present + non-empty; MEMORY.md has ≥1 entry with linked file |
| 7 | Agent + skill + command surface | ≥1 file in each of: global agents/skills, repo agents/skills; ~/.agents symlink; ~/.claude/AGENTS.md present |

## Critical discipline (avoid the criterion-bug pattern)

The 2026-05-19 session voided records at sequences 838+862 because the criterion required `--dunbar-layer` on `cheese_surface` invocations — but `cheese_surface`'s binary signature has no such flag. The corrected per-binary criterion is baked into `audit-tier-3.sh` and `audit-tier-4.sh`:

- `cheese_surface` is checked for `--scope` + `--last` only
- `cheese_text` is checked for `--scope` + `--event`; `--dunbar-layer` is valid only for explicit escalation records

If you find yourself wanting to apply a blanket flag check, you're regressing — read each command's binary signature first.

## Discipline (per `feedback_closure_detail_field_discipline`)

If a tier fails, read its wire detail field BEFORE relabeling. Concrete actionables in the `reasons=` field; reconcile against external ground truth. Don't relabel a fail as "honest" or "criterion-corrected" without producing a void+resubmit on the wire.

For Codex-native installs, also run `scripts/verify-codex-native-hooks.sh` and
`scripts/verify-codex-live-hooks.sh` before claiming hook coverage complete.
The setup audit proves host wiring; native dry-run proves normalization; live
hooks prove installed command execution, provider metadata, and daemon readback.

The current Telephone DAG also has named downstream gates outside this setup
audit:

```bash
scripts/verify-telephone-storage-broker.sh
scripts/verify-telephone-performance.sh
```

`wire-audit` is not green evidence for A17 storage/broker, A18 Codex E2E, A19
performance, or A20-A23 erasure readiness. Report those as separate gates with
their own blockers.

## Output

Wire records: `hook_setup_tier_{1..7}` (one per tier) + one `hook_setup_summary`. Read via:

```bash
cheese_call --last 50 --format text | grep -E "hook_setup_(tier|summary)"
```
