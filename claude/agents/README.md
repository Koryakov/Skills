# Agent roles

Predefined subagents for orchestration patterns — pattern selection lives in
[.claude/skills/orchestrate/SKILL.md](../skills/orchestrate/SKILL.md).
Coordinator = the main session (subagents cannot spawn subagents).

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
`fable` is never pinned in files; `inherit` covers it when the session runs fable.
