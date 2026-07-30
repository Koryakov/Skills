# Agent roles

Predefined subagents for orchestration patterns — pattern selection lives in
[.claude/skills/orchestrate/SKILL.md](../skills/orchestrate/SKILL.md).
Coordinator = the main session. No role below lists `Agent` in `tools:`, so none spawns
subagents — but the platform allows nesting (depth 3 by default, remotely gated),
`general-purpose` has no allowlist, and any role holding `Bash` can shell out to a fresh
session. Treat the limit as convention, not a spend barrier; keep coordination here.

| Role | Model / effort | Purpose | Returns |
|---|---|---|---|
| explorer | haiku / low | fast repo recon | path-cited map |
| researcher | sonnet / medium | web evidence for decisions | sourced findings + confidence |
| analyzer | opus / high | deep single-question analysis | conclusion + evidence chain |
| hypothesizer | opus / high | competing root-cause theories | ranked hypotheses + falsification tests |
| critic | opus / high | adversarial review | severity-ranked issues or explicit "no findings" |
| planner | opus / high | implementation strategy (read-only) | executable step plan |
| implementer | sonnet / medium | scoped code writing (`acceptEdits`) | diff summary + self-check |
| verifier | sonnet / medium | run tests/app, report observed behavior | pass/fail + verbatim output |
| synthesizer | sonnet / medium | merge findings files (reduce step) | single deduplicated report |
| documenter | sonnet / low | project docs in terse table style | docs touched |

Tuning: per-invocation Agent-tool params override these files — `model` (escalate to
`opus`/`fable` for a hard instance), `run_in_background`, `isolation: worktree`.
`model` overrides the model only — `effort` and `maxTurns` stay as pinned here and
cannot be set at spawn time, so an escalated `explorer` still runs at low effort with a
15-turn cap.
`fable` is never pinned in files; `inherit` covers it when the session runs fable.
