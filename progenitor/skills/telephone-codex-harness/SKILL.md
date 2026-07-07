---
name: telephone-codex-harness
description: Emit and verify Codex-shaped Telephone hook records through the provider-neutral `tool_hook_emit` wrapper, including native-hook install/trust checks and live readback for the nine Codex events.
---

# Telephone Codex Harness

Use this when Codex needs the same hook/harness behavior as other tool
surfaces, or when verifying Codex native hook coverage.

Commands:

```bash
~/.local/share/telephone/bootstrap
telephone_install_codex_project
scripts/verify-codex-native-hooks.sh

tool_hook_emit codex SurfaceSmoke repo <<'JSON'
{"session_id":"codex-skill","actor_kind":"agent","model":"codex","silmaril_python":"/home/tristan/anaconda3/envs/silmaril/bin/python"}
JSON

telephone_query --event hook_tool_surface_smoke --last 1 --format text
scripts/verify-codex-live-hooks.sh
```

The harness emits normalized records with `provider=codex`,
`surface=codex-cli`, `provider_event=<event>`, and the Silmaril Python
interpreter path. Normal hook observations omit `dunbar_layer` in both top-level
fields and `detail`; pass a fourth `tool_hook_emit` layer argument only for an
explicit Dunbar escalation. Use `/home/tristan/anaconda3/envs/silmaril/bin/python`
directly when Python is needed; do not rely on shell activation.

Provider-event mapping:

| Codex event | Normalized event | Scope used by live verifier |
|---|---|---|
| `SessionStart`, `SubagentStart` | `hook_tool_session_start` | `repo`, `subagent` |
| `PreToolUse`, `PostToolUse`, `PermissionRequest` | `hook_tool_use` | `repo` |
| `UserPromptSubmit`, `PreCompact`, `PostCompact` | `hook_context_surface` | `repo` |
| `Stop` | `hook_tool_turn_end` | `repo` |

Coverage is not complete until these are true:

- `.codex/hooks.json` contains all nine native Codex hooks: `SessionStart`,
  `SubagentStart`, `PreToolUse`, `PostToolUse`, `PermissionRequest`,
  `UserPromptSubmit`, `PreCompact`, `PostCompact`, and `Stop`.
- `$CODEX_HOME/config.toml` contains the managed
  `telephone-codex-native-hook-trust` block for the target hooks path.
- Live readback shows provider-neutral events such as `hook_tool_use`,
  `hook_tool_session_start`, `hook_tool_turn_end`,
  `hook_context_surface`, and `hook_tool_surface_smoke`.
- Live readback checks `schema_urn`, `writer_version`, provider/surface/event
  URNs, hook family/phase URNs, `actor_kind_urn`, `language_binding_urn`, and
  `taxonomy_path` in `detail`; command exit alone is not hook coverage.
- No normal observation uses `dunbar_layer`; escalation requires an explicit
  fourth argument and must be justified by a real alarm condition.

Gate notes:

- `scripts/verify-codex-native-hooks.sh` now reports
  `blocked:codex_native_incomplete` for missing tools, hooks config, fixtures,
  or fixture event names; those do not count as dry-run passes.
- `scripts/verify-codex-live-hooks.sh` now reports
  `blocked:codex_live_incomplete` when the daemon, hooks config, required
  commands, hook command, fixture, or Silmaril Python boundary is absent.
- If UDS or Mix is blocked by the sandbox, `tool_hook_emit` may fall back to
  direct OCF spool; that proves normalization/spool only. A18 live certification
  still requires daemon readback for the same correlation id.
- The Codex nodes are A01, A03, A10, and A11 in
  `research/specs/data/research/telephone_dag_manifest/manifest.csv`.
