---
name: orchestrate
description: Select a multi-agent architecture pattern (P0–P8) and role lineup for a complex task: state cost and expected time before spawning, have read-only roles return findings for the coordinator to persist, and collect every launched agent's result. Use proactively before starting any comprehensive investigation, audit, large feature, hard ambiguous bug, or research-backed decision — or when the user asks for subagents/multi-agent work. Not for small targeted changes (≤~3 files), quick questions, or interactive back-and-forth — do those solo.
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

## 2. Cost and time — state both before spawning

**Sum each planned agent's role median.** Treat the total as a **lower bound** (21% cost
sample, 89 of 427 runs; the unsampled ones are async launches, which skew long).

```
p50 tokens/run: planner 103.6k · implementer 88k (p90 197.6k) · analyzer 82k ·
critic 66.6k · documenter 65.5k · general-purpose 61.8k · synthesizer 52.2k ·
explorer 38.1k · verifier 32.6k
all-run p50 61.9k · p90 100.2k · max 197.6k · duration p50 ~3.1 min, max 17.3 min
Unmeasured — researcher, built-in Explore/Plan: use the all-run p50 as an explicitly
labelled placeholder; never invent a per-role figure.
The 21% sample CANNOT be improved from transcripts — don't retry. Reported
`subagent_tokens` is unrecoverable from the usage fields: for a run reporting 41,358,
output summed to 14,593, input+output 15,294, and +cache_creation 124,394. No formula
matches. Duration, by contrast, IS fully recoverable — see the measured table below.
```

P3 ≈ planner + implementer + verifier + critic ≈ 291k. The old S/M/L/XL bands are gone:
they topped out at "400k+" while one real session extrapolated to ~4M.

**Time is a separate quantity from cost.** A parallel wave costs linearly but lasts only as
long as its **slowest** member, so width costs **sub-linear** time — not zero, and the max
creeps up as you widen a right-tailed distribution. Sequential steps are what you wait for:

```
agent-compute ≈ Σ over sequential steps ( slowest agent in that step )
  a "step" = one barrier: agents that must finish before the next starts.
  P4's implementer→critic pair is TWO steps: 10.9 + 12.9, NOT max(10.9, 12.9).

MEASURED MINUTES — 113 subagent transcripts since 2026-07-30, full coverage (not a
sample), extracted twice with identical results. Prefer these over the p50/max on the
token line above, which came from a 21% parent-transcript sample and understates tails.
  role              n     p50    p90     max
  analyzer         49    10.6   16.5    23.2
  critic           27    12.9   21.3    52.2
  implementer      13    10.9   18.6    24.4
  general-purpose   8     6.7   11.5    11.5
  explorer          8     1.2    2.0     2.0
  verifier          5     5.1   36.5    36.5   <- n=5; p90 IS the max, one point. Weak.
  ALL ROLES       113    10.6   17.4    52.2

still unmeasured — planner, synthesizer, hypothesizer, documenter, built-in Explore/Plan:
  substitute the nearest measured role and label it a placeholder. Never invent a figure.
```

This is **agent-compute only**. P3–P5 and P8 end the turn at each stage (§4), so *elapsed*
time also includes however long the reply takes — don't present compute as elapsed.

Measured contrast: 3 analyzers in one wave = 198k in **8.2 min**; 2 sequential critic rounds
= 100k in **19.9 min** — half the tokens, 2.4× the wait. Where work is genuinely partitionable
(recon, audits, independent hypotheses) prefer **width**. This does **not** transfer to P4:
three critics on one unfixed artifact return triplicate findings, because a later round's
value comes from reviewing the *revised* artifact.

State agents, steps, expected compute minutes, and the token lower bound in **one line**,
every time. Get an explicit go/no-go when **either** the total exceeds **~30 min** or **any
single step** does — a lone critic reached **52.2 min**, so a one-step run clears a
sum-based gate and still overruns badly. Recomputed on measured figures: P3 ≈ 29 min +
planner, P4 ≈ 71 min (3 × [10.9 + 12.9]). A bar below ~30 min would fire on nearly every
pattern and decay into noise; the per-step check is what catches the long tail.

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
| Hand over what is collected | when a stage yields (§ Mid-loop checkpoints), hand its output to the user **labelled not-yet-reviewed** rather than holding it until the whole pipeline finishes — long silence is the complaint, and partial output ends it at no cost. Keep the label distinct from `unverified` above, which is a term of art meaning "confirm with a grep or git query". Do **not** pre-launch the next stage to overlap the user's reading: no fan-out pattern in §1 has a following critic (P1/P2 end at the coordinator, P6 at the synthesizer, P8 at Plan), and spending past an unreviewed stage is what § Mid-loop checkpoints exists to prevent |
| Scope round 2+, never round 1 | round 1 is a **full sweep**; its value is finding what nobody knew was load-bearing — it caught the two blind spots a real skill shipped with. Round 2+ may be narrowed to the corrected claims plus whatever they touch: re-reviewing a whole 96-line document cost 12.4 min when the live findings concerned four claims. Never scope round 1 to "what matters" — that set is only knowable after it runs |
| Later rounds — gate them | run round 2+ only if the previous round returned a **Critical or Major** (`critic.md` severities); Minor/style-only → stop. **Two traps.** (1) A clean report from a capped read-only role is an unverified negative (§ Distrust negatives): require the round to state what it examined, and treat a thin or near-empty report as **weak** — escalate it, never read it as clean. (2) The gate assumes a **static artifact** (a document, analysis, or frozen diff). In P4 the implementer edits between rounds, so a style-only round 1 still leaves those edits unreviewed — review the new diff instead of stopping. Round 1 is never skipped; P4's ≤3 rounds remains the ceiling |
| Escalation | weak result → re-run the **same role**, not more agents. `model` is the **only** per-invocation lever — `effort` and `maxTurns` come from the role file and cannot be set at spawn time (an `effort` key is silently dropped). **Read the role file's `model:` before overriding it** — an override to the value already pinned there does nothing, silently: 9 of 19 overrides since v2.2 were no-ops (`analyzer` ×5 and `critic` ×4 sent `model: opus`, both already `model: opus`). Pinned opus: analyzer, critic, planner, hypothesizer — for those use `model: fable`, narrow the subject, or edit the role file. Pinned sonnet: implementer, verifier — `model: opus` there is real escalation. Overriding `model` leaves the rest pinned: an escalated `explorer` still runs at `effort: low` with a 15-turn cap |
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
**v2.3** (2026-08-04) — time as well as cost. Agent-compute is stated alongside tokens, summed
over **sequential steps** (a P4 implementer→critic pair is two steps, not one); width is
sub-linear in time, so partitionable work buys thoroughness by widening — but that does not
transfer to P4. Go/no-go at ~30 min, firing per **step** as well as on the total, because one
critic alone hit 52.2 min. Per-role durations come from 113 subagent transcripts at full
coverage, extracted twice with identical results; the 21% token sample is recorded as
**unimprovable** from transcripts so it is not retried. `model` overrides must be checked
against the role file first — 9 of 19 were silent no-ops. Collected stages are handed over
labelled not-yet-reviewed; round 1 is a full sweep and only round 2+ may be scoped; later
rounds gated on Critical/Major, with a clean-but-thin report treated as weak rather than
clean, and P4 excepted because its artifact mutates between rounds.
Confirmed working, left alone: `brief.md` referencing rose to 58% of prompts; the coordinator
persisted findings 86 times.
Two designs **rejected**: named depth modes (quick/standard/deep) — `effort` is dropped at
spawn time so the dial has no lever, and unenforceable rules go unused here (escalation: 0 of
427 runs); and pre-launching the next stage to overlap the user's reading — no §1 fan-out
pattern has a following critic, and it would spend past an unreviewed stage.
Gap this file cannot fix: the skill loaded in only 3 of 32 sessions that ran agents — the
essential rules were mirrored into the always-loaded global instructions.
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
