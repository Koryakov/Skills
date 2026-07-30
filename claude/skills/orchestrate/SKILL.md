---
name: orchestrate
description: Select a multi-agent architecture pattern (P0–P8) and role lineup for a complex task: cost it before spawning, have read-only roles return findings for the coordinator to persist, and collect every launched agent's result. Use proactively before starting any comprehensive investigation, audit, large feature, hard ambiguous bug, or research-backed decision — or when the user asks for subagents/multi-agent work. Not for small targeted changes (≤~3 files), quick questions, or interactive back-and-forth — do those solo.
---

# Orchestrate — agent pattern selection

Coordinator = this session. Roles: [.claude/agents/README.md](../../agents/README.md).
`∥` = in parallel.

## 0. Is orchestration warranted?

P0 (solo) is the default — orchestration must earn its token cost. Move past P0 only
when the task matches a row below.

Nesting (corrected): the platform **permits** it — depth 3 by default since CLI 2.1.219
(2.1.217 had disabled it; `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1` turns it off). What
stops custom roles is that no role's `tools:` list includes `Agent` — but
`general-purpose` has no allowlist and **can** nest. Don't: nesting multiplies spend and
is untested here. Coordination stays in this session.

## 1. Which pattern

| # | Pattern | Shape | Use when | Don't when |
|---|---|---|---|---|
| P0 | Solo | no agents | fits in context; ≤~3 files; iterative with user | — |
| P1 | Fan-out recon | N× explorer ∥ → coordinator synthesizes | scope unknown; several repo areas; "where/how does X work" | single known area → P0 |
| P2 | Research & decide | researcher(s) ∥ (+ analyzer for repo side) → coordinator | tech choice; lock-in decision; needs external evidence | facts already in repo/docs |
| P3 | Pipeline build | planner → implementer → verifier → critic | multi-file feature, clear spec | spec ambiguous → P2/clarify first; tiny change → P0 |
| P4 | Generator–critic loop | implementer ⇄ critic (≤3 rounds) → verifier | correctness-critical change; high blast radius | ordinary change (P3's single critic pass suffices) |
| P5 | Competing hypotheses | hypothesizer → N× analyzer ∥ (one per hypothesis) → critic ranks | hard bug; conflicting evidence; prior fixes failed | reproducible bug, obvious trail → P0/P3 |
| P6 | Map-reduce audit | N× analyzer ∥ (partitioned) → synthesizer | large-corpus review/audit/migration sweep | corpus fits one analyzer |
| P7 | Team feature ownership | agent team: lead + teammates own layers | independent workstreams needing teammate↔teammate messaging | anything P1–P6 covers; experimental (setup below) |
| P8 | Recon-then-multi-plan | built-in Explore ×3 recon → built-in Plan ×2–3, each prompted for a deliberately different design bias (e.g. security-first / minimal-change pragmatic / event-driven) → coordinator picks or merges | several genuinely different designs needed before committing | one approach clearly suffices → P3 |

P5–P7 stay listed despite rare use. P8: built-in `Plan` emits ~15.5k chars, the largest
of any role, and cannot write — the coordinator persists it (§3).

## 2. Cost — state it before spawning

**Sum each planned agent's role median.** Treat the total as a **lower bound** (21% cost
sample, 89 of 427 runs; the unsampled ones are async launches, which skew long).

```
p50 tokens/run: planner 103.6k · implementer 88k (p90 197.6k) · analyzer 82k ·
critic 66.6k · documenter 65.5k · general-purpose 61.8k · synthesizer 52.2k ·
explorer 38.1k · verifier 32.6k
all-run p50 61.9k · p90 100.2k · max 197.6k · duration p50 ~3.1 min, max 17.3 min
Unmeasured — researcher, built-in Explore/Plan: use the all-run p50 as an explicitly
labelled placeholder; never invent a per-role figure.
```

P3 ≈ planner + implementer + verifier + critic ≈ 291k. The old S/M/L/XL bands are gone:
they topped out at "400k+" while one real session extrapolated to ~4M.

## 3. How agents share context

Subagents start isolated — they share via files or coordinator relay.

```
.claude/orchestration/<task-slug>/
  brief.md               # coordinator writes once: task, constraints, file pointers
  findings-<subject>.md  # coordinator persists each agent's returned findings
  report.md              # synthesizer output, or the coordinator's own reduce step
```

That dir is **not** gitignored by default — add the `.gitignore` entry or these files
land in commits.

| Rule | |
|---|---|
| Read-only roles — analyzer, critic, planner, hypothesizer, explorer, verifier, researcher, built-in Explore/Plan | hold no `Write` (16/16 attempts blocked) → they return findings in their **final message**; the **coordinator persists** them to `findings-<subject>.md`, partitioned by subject, not by role |
| Write-capable — implementer, documenter, synthesizer, plus unrestricted roles like general-purpose | may persist directly |
| `brief.md` | reference it instead of repeating context — 88/88 read when named. It does not shorten prompts (median 2,444 vs 2,405); its value is consistency |
| Small results | under ~1 page → final message only; skip files |
| Built-in Explore/Plan | skip CLAUDE.md — restate any binding project rule in their prompt |
| Cleanup | 12 of 25 real scratch dirs held only an abandoned `brief.md` — delete the dir once its report is collected |

## 4. Execution rules

| Rule | |
|---|---|
| Result collection | every spawn returns an async launch stub — **collect and act on every launched agent, then end the turn**. 299 of 427 results were stubs whose collection is unverifiable, and in one session the user had to ask for two finished designs the coordinator never gathered. Use `SendMessage` to pull an async or paused agent. An uncollected agent is 100% wasted spend |
| Mid-loop checkpoints | P3–P5, P8: after collecting a stage, **end the turn and yield** — not report-and-continue. Never spend past a failed stage |
| Escalation | weak result → re-run the **same role**, not more agents. `model: opus` is a no-op on the opus-pinned roles (analyzer, critic, planner, hypothesizer) — for those raise effort or narrow the subject; the model override only bites on haiku/sonnet roles |
| Built-in Explore vs custom explorer | built-in inherits the session model (opus 47/81 observed) and skips CLAUDE.md; custom `explorer` is haiku-pinned, `effort: low`, 15-turn cap. Neither is measured as cheaper overall — pick on those mechanics, not on a cost claim |
| Reuse / continuation | `SendMessage` resumes an agent paused mid-run or async-launched **and** follows up on a finished one — don't respawn |
| Background | `run_in_background` is a hint, not a guarantee — long work backgrounds regardless, so rely on explicit collection rather than the flag |
| Worktrees | `isolation: worktree` — mainly parallel implementers; a single implementer may also use it |

## 5. P7 setup — agent teams (experimental)

```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

- Teammates reference roles in `.claude/agents/` by name (their `tools`/`model` honored).
- Known rough edges: slow shutdown, task-status lag, one team per lead, no nested teams.
  Always clean up via the lead.
- Reach for P7 only when teammates must message each other directly; otherwise P6.

---
**v2.0** (2026-07-30) — nesting premise and Explore cost claim corrected; tiers replaced
by summed role medians; findings now coordinator-persisted; result collection and P8
added; escalation narrowed; checkpoints yield; scratch dir documented as not ignored.
