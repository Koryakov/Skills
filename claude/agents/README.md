# Agent roles

Predefined subagents for orchestration patterns — pattern selection lives in
[.claude/skills/orchestrate/SKILL.md](../skills/orchestrate/SKILL.md).
Coordinator = the main session. No role below lists `Agent` in `tools:`, so none spawns
subagents — but the platform allows nesting (depth 3 by default, remotely gated),
`general-purpose` has no allowlist, and any role holding `Bash` can shell out to a fresh
session. Treat the limit as convention, not a spend barrier; keep coordination here.

| Role | Model / effort | Purpose | Returns |
|---|---|---|---|
| explorer | `claude-haiku-4-5` / low | fast repo recon | path-cited map |
| researcher | `claude-sonnet-5` / medium | web evidence for decisions | sourced findings + confidence |
| analyzer | `claude-opus-5` / high | deep single-question analysis | conclusion + evidence chain |
| hypothesizer | `claude-opus-5` / high | competing root-cause theories | ranked hypotheses + falsification tests |
| critic | `claude-opus-5` / high | adversarial review | severity-ranked issues or explicit "no findings" |
| planner | `claude-opus-5` / high | implementation strategy (read-only) | executable step plan |
| implementer | `claude-sonnet-5` / medium | scoped code writing (`acceptEdits`) | diff summary + self-check |
| verifier | `claude-sonnet-5` / low | run tests/app, report observed behavior | pass/fail + verbatim output |
| synthesizer | `claude-sonnet-5` / low | merge findings files (reduce step) | single deduplicated report |
| documenter | `claude-haiku-4-5` / low | project docs in terse table style | docs touched |

Models are pinned to exact IDs (not tier aliases) since v1.1 — see each role file's version
footer. verifier/synthesizer effort and documenter's model were stepped down after an
in-session A/B test (real task through old vs. new settings) showed no quality loss; opus-tier
roles (analyzer/critic/hypothesizer/planner) stayed at `high` — a same-day test found `medium`
held quality on analyzer/critic but did not reduce token spend, so the step-down wasn't applied.

Tuning: per-invocation Agent-tool params override these files — `model` (escalate to
`opus`/`fable` for a hard instance), `run_in_background`, `isolation: worktree`.
`model` overrides the model only — `effort` and `maxTurns` stay as pinned here and
cannot be set at spawn time, so an escalated `explorer` still runs at low effort with a
15-turn cap.
`fable` is never pinned in files; `inherit` covers it when the session runs fable.
