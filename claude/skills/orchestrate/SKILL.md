---
name: orchestrate
description: Select a multi-agent architecture pattern (P0–P7) and role lineup for a complex task. Use proactively before starting any comprehensive investigation, audit, large feature, hard ambiguous bug, or research-backed decision — or when the user asks for subagents/multi-agent work. Not for small targeted changes (≤~3 files), quick questions, or interactive back-and-forth — do those solo.
---

# Orchestrate — agent pattern selection

Coordinator = this session (subagents cannot spawn subagents; the `Agent` tool is
blocked inside them). Pick the **cheapest pattern that fits**; P0 is the default —
orchestration must earn its token cost. Roles defined in
[.claude/agents/README.md](../../agents/README.md).

## Selection table

| # | Pattern | Shape | Use when | Don't when | Cost |
|---|---|---|---|---|---|
| P0 | Solo | no agents | fits in context; ≤~3 files; iterative with user | — | S |
| P1 | Fan-out recon | N× explorer ∥ → coordinator synthesizes | scope unknown; several repo areas; "where/how does X work" | single known area → P0 | S–M |
| P2 | Research & decide | researcher(s) ∥ (+ analyzer for repo side) → coordinator | tech choice; lock-in decision; needs external evidence | facts already in repo/docs | M |
| P3 | Pipeline build | planner → implementer → verifier → critic | multi-file feature, clear spec | spec ambiguous → P2/clarify first; tiny change → P0 | L |
| P4 | Generator–critic loop | implementer ⇄ critic (≤3 rounds) → verifier | correctness-critical change; high blast radius | ordinary change (P3's single critic pass suffices) | L |
| P5 | Competing hypotheses | hypothesizer → N× analyzer ∥ (one per hypothesis) → critic ranks | hard bug; conflicting evidence; prior fixes failed | reproducible bug, obvious trail → P0/P3 | L–XL |
| P6 | Map-reduce audit | N× analyzer ∥ background (partitioned) → synthesizer | large-corpus review/audit/migration sweep | corpus fits one analyzer | XL |
| P7 | Team feature ownership | agent team: lead + teammates own layers | independent workstreams needing teammate↔teammate messaging | anything P1–P6 covers; experimental (setup below) | XL |

Cost tiers (subagent tokens, rough): S <50k · M 50–150k · L 150–400k · XL 400k+.
State the chosen pattern + expected tier to the user before spawning.

## Execution rules

| Rule | |
|---|---|
| Parallel spawns | independent agents → one message, multiple Agent calls |
| Reuse | follow-up for a finished agent → `SendMessage` (context intact), don't respawn |
| Background | verbose/long work (P6 partitions, long verifier runs) → `run_in_background: true` |
| Worktrees | parallel implementers only → `isolation: worktree` |
| Escalation | weak result → re-run same role with `model: opus`/`fable` or higher effort — not more agents |
| Built-in Explore | quick recon where project rules don't matter → built-in Explore (skips CLAUDE.md, cheaper); else custom explorer |
| Mid-loop checkpoints | P3–P5: report to user between stages; never burn XL tokens past a failed stage |

## Context-sharing protocol

Subagents start isolated — they share via files or coordinator relay.

```
.claude/orchestration/<task-slug>/     # gitignored scratch
  brief.md                # coordinator writes once: task, constraints, file pointers
  findings-<role>-<n>.md  # each parallel agent writes its findings (P5/P6)
  report.md               # synthesizer output
```

- Spawn prompts name `brief.md` instead of repeating context — cheaper, consistent.
- Results under ~1 page → relay via the agent's final message; skip files.
- Built-in Explore/Plan don't load CLAUDE.md — restate any binding project rule in
  their prompt.

## P7 setup — agent teams (experimental)

```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

- Teammates reference roles in `.claude/agents/` by name (their `tools`/`model` honored).
- Known rough edges: slow shutdown, task-status lag, one team per lead, no nested teams.
  Always clean up via the lead.
- Reach for P7 only when teammates must message each other directly; otherwise P6.
