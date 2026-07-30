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
(2.1.217 had disabled it; `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1` turns it off). That
default is remotely gated and has already flipped twice — don't treat it as fixed. What
stops custom roles is that no role's `tools:` list includes `Agent` — but
`general-purpose` has no allowlist and **can** nest, and any role holding `Bash` can
shell out to a fresh session, so the allowlist is convention, not a spend barrier. Don't
nest: it costs ~⅓ more than a flat fan-out, removes the user's veto (depth-2+ is
observable only via headless stream-json forwarding), and blurs the `file:line` evidence
that critic/verifier/analyzer exist to return. Coordination stays in this session.

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
<session scratch dir>/<task-slug>/   # DEFAULT — outside the repo
  brief.md               # coordinator writes once: task, constraints, file pointers
  findings-<subject>.md  # coordinator persists each agent's returned findings
  report.md              # synthesizer output, or the coordinator's own reduce step
```

`<scratch>` = the session scratch dir named in the environment; failing that, any temp dir
outside the working tree. Pass agents **absolute** paths — the 88/88 brief read-rate was
measured on in-repo paths. Use in-repo `.claude/orchestration/<task-slug>/` **only** when
artifacts must outlive the session or be shared: it is **not** gitignored, so add the
entry first — 12 scratch files were observed staged in a real repo. Create the dir only
when files are actually needed.

| Rule | |
|---|---|
| Read-only roles — analyzer, critic, planner, hypothesizer, explorer, verifier, researcher, built-in Explore/Plan | hold no `Write` (16/16 attempts blocked) → they return findings in their **final message**; the **coordinator persists** them to `findings-<subject>.md`, partitioned by subject, not by role |
| Write-capable — implementer, documenter, synthesizer, plus unrestricted roles like general-purpose | may persist directly |
| `brief.md` | reference it instead of repeating context — 88/88 read when named. It does not shorten prompts (median 2,444 vs 2,405); its value is consistency |
| Small results | under ~1 page → final message only; skip files |
| Built-in Explore/Plan | skip CLAUDE.md — restate any binding project rule in their prompt |
| Return contract — the reverse case | a global or session instruction rule can **override** a role's own output contract: an `explorer` whose role file says "final message = the map only" returned a single `// <reason>` annotation and nothing else, converting a paid run into zero. Restate the required final-message shape in **every** spawn prompt. Prevention is unproven — 9 spawns, 0 failures, against an ~18% base rate for `explorer`, so that is weak evidence; the §4 detector is what actually catches it |
| Cleanup | measured twice — 12 of 25, then 9 of 20 real scratch dirs held only an abandoned `brief.md`. Delete the dir once its report is collected |

## 4. Execution rules

| Rule | |
|---|---|
| Result collection | a backgrounded spawn returns a launch stub, not the result — **collect and act on every launched agent, then end the turn**. 299 of 427 results were stubs whose collection is unverifiable, and in one session the user had to ask for two finished designs the coordinator never gathered. Use `SendMessage` to pull an async or paused agent. An uncollected agent is 100% wasted spend |
| Empty result — detect, don't trust prevention | a **collected** final message under ~200 chars, or opening with `//`, is **absent**, not weak — escalation does not apply. (A launch stub is not this: that is uncollected, see above.) Re-prompt a fresh agent with an explicit output contract rather than resuming — a resume carries the corrupted context forward, and in the field case it resumed straight into the same wrong answer for a second full run. Cause may be an instruction conflict (§3) or a turn cap; the detector fires either way |
| Distrust negatives | "not found" / "not merged" / "not present" from a capped read-only role is **unverified** — confirm with one grep or git query before acting on it. A 15-turn haiku recon agent produced a confident false negative that a single `git merge-base --is-ancestor` disproved |
| Mid-loop checkpoints | P3–P5, P8: after collecting a stage, **end the turn and yield** — not report-and-continue. Never spend past a failed stage |
| Escalation | weak result → re-run the **same role**, not more agents. `model` is the **only** per-invocation lever — `effort` and `maxTurns` come from the role file and cannot be set at spawn time (an `effort` key is silently dropped). `model: opus` is a no-op on the opus-pinned roles (analyzer, critic, planner, hypothesizer): for those use `model: fable`, narrow the subject, or edit the role file. Overriding `model` leaves the rest pinned — an escalated `explorer` still runs at `effort: low` with a 15-turn cap |
| Built-in Explore vs custom explorer | built-in inherits the session model (opus 47/81 observed) and skips CLAUDE.md; custom `explorer` is haiku-pinned, `effort: low`, 15-turn cap. Neither is measured as cheaper overall — pick on those mechanics, not on a cost claim |
| Reuse / continuation | `SendMessage` resumes an agent paused mid-run or async-launched **and** follows up on a finished one — don't respawn, **except after an empty result** (below), where a fresh spawn beats carrying a corrupted context forward. A resume bills as a **full second run** (measured: 57k to collect after a 57k first run, for one wrong answer), so prompt correctly once rather than planning to resume |
| Background | agents background by **default** — pass `run_in_background: false` for a synchronous run when the result is needed before continuing. Collect explicitly either way |
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
**v2.2** (2026-07-30) — from field use: spawn prompts must restate the role's return
contract (a global rule can silently override it); empty/annotation-only results are
absent, not weak; negatives from capped recon roles are unverified; a `SendMessage`
resume bills a full second run; scratch defaults outside the repo.
**v2.1** (2026-07-30) — `effort` is not a spawn-time lever (was wrongly advised);
background is the default, not a hint; nesting note records the `Bash` escape route and
the remote gating of the depth default.
**v2.0** (2026-07-30) — nesting premise and Explore cost claim corrected; tiers replaced
by summed role medians; findings now coordinator-persisted; result collection and P8
added; escalation narrowed; checkpoints yield; scratch dir documented as not ignored.
