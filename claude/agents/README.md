# Agent roles

Predefined subagents for orchestration patterns — pattern selection lives in
[.claude/skills/orchestrate/SKILL.md](../skills/orchestrate/SKILL.md).
Coordinator = the main session. No role below lists `Agent` in `tools:`, so none spawns
subagents — but the platform allows nesting (depth 3 by default, remotely gated),
`general-purpose` has no allowlist, and any role holding `Bash` can shell out to a fresh
session. Treat the limit as convention, not a spend barrier; keep coordination here.

| Role | Model / effort | Purpose | Returns |
|---|---|---|---|
| explorer | `claude-haiku-4-5` | fast repo recon | path-cited map |
| researcher | `claude-sonnet-4-6` / medium | web evidence for decisions | sourced findings + confidence |
| analyzer | `claude-opus-4-8` / high | deep single-question analysis | conclusion + evidence chain |
| hypothesizer | `claude-opus-4-8` / high | competing root-cause theories | ranked hypotheses + falsification tests |
| critic | `claude-opus-4-8` / high | adversarial review | severity-ranked issues or explicit "no findings" |
| planner | `claude-opus-4-8` / high | implementation strategy (read-only) | executable step plan |
| implementer | `claude-sonnet-4-6` / medium | scoped code writing (`acceptEdits`) | diff summary + self-check |
| verifier | `claude-sonnet-4-6` / medium | run tests/app, report observed behavior | pass/fail + verbatim output |
| synthesizer | `claude-sonnet-4-6` / medium | merge findings files (reduce step) | single deduplicated report |
| documenter | `claude-haiku-4-5` | project docs in terse table style | docs touched |

Models are pinned to exact IDs (not tier aliases). v1.1 pinned to the 5.x line; **v1.2 rolls
back to the proven 4.x models** — Opus 4.8 for the reasoning roles, Sonnet 4.6 for the
sonnet-tier roles — a conservative choice favoring predictable token spend over the 5.x
generation's larger, less bounded reasoning (Opus 5 defaults thinking on; a same-day test
showed its `medium` effort held quality but did **not** cut tokens on evidence-bound tasks).
Haiku 4.5 does not support the `effort` parameter, so explorer and documenter carry no
`effort:` field (it was a silent no-op before).

Tuning: per-invocation Agent-tool params override these files — `model` (escalate to
`opus`/`fable` for a hard instance), `run_in_background`, `isolation: worktree`.
`model` overrides the model only — `effort` and `maxTurns` stay as pinned here and
cannot be set at spawn time, so an escalated `explorer` still runs at low effort with a
15-turn cap.
`fable` is never pinned in files; `inherit` covers it when the session runs fable.
