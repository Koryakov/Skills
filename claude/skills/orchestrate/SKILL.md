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

**Sum each planned agent's role median — then read the total by scope.** The medians describe
*substantial* tasks. For open-ended work the sum is a **lower bound** (21% cost sample, 89 of
427 runs; the unsampled ones are async launches, which skew long). For a **narrow, single-
question** prompt it is an **over**-estimate: a measured 2-explorer test projected ≥76k and
~2 min and actually cost 31k in 19 s — 41% of tokens, a tenth of the time. State which way
you are reading it.

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
agent-span ≈ Σ over sequential steps ( slowest agent in that step )
  a "step" = one barrier: agents that must finish before the next starts.
  P4's implementer→critic pair is TWO steps: 14.3 + 12.9, NOT max(14.3, 12.9).

MEASURED MINUTES — 212 subagent transcripts, 2026-07-30 to 2026-08-11, full coverage
(not a sample). Prefer these over the p50/max on the token line above, which came from a
21% parent-transcript sample.
  role              n     p50    p90     max
  analyzer         94    10.4   16.0    42.4
  critic           65    12.9   20.7   708.4
  implementer      22    14.3   21.6   105.2
  general-purpose  11     6.7   11.5    15.2
  explorer         10     0.9    2.0     2.0
  verifier          6     8.2   36.5    36.5   <- n=6; p90 IS the max, one point. Weak.
  claude            2    25.3      -       -   <- not a §1 role; listed only so the rows
                                                  reconcile to 212. Never plan with it.
  planner           1    14.9      -       -   <- n=1. Placeholder-grade.
  researcher        1     4.4      -       -   <- n=1. Placeholder-grade.
  ALL ROLES       212    11.3   18.6   708.4   (rows sum to 212)

EVERY figure above is a WALL SPAN — last timestamp minus first — so all of them, p50 and
p90 included, carry whatever idle time an agent spent paused. The tail carries most of it:
the 708-min critic is a pause, not 12 h of compute. Plan with p50, treat p90 as soft, and
read max only as "this role can stall" — never as a compute estimate.

Placeholder-grade rows (n<=2) are not medians. Any total that includes one must be stated
as placeholder-bearing — see P3 below.

still unmeasured — synthesizer, hypothesizer, documenter, built-in Explore/Plan:
  substitute the nearest measured role and label it a placeholder. Never invent a figure.
```

These are **agent-side spans**, not your wall-clock. P3–P5 and P8 end the turn at each stage
(§4), so the time you actually experience adds however long each reply takes. Quote the
agent-side figure and label it as such; never present it as total elapsed.

Measured contrast: 3 analyzers in one wave = 198k in **8.2 min**; 2 sequential critic rounds
= 100k in **19.9 min** — half the tokens, 2.4× the wait. Where work is genuinely partitionable
(recon, audits, independent hypotheses) prefer **width**. This does **not** transfer to P4:
three critics on one unfixed artifact return triplicate findings, because a later round's
value comes from reviewing the *revised* artifact.

**Log line — emit this before the first spawn, every time**, in exactly this shape so it is
greppable in transcripts later:

```
orchestrate v{VER} · P{N} · {lineup} · {k} steps · ~{m} min · ≥{t}k
   VER from the footer, N the pattern number, lineup e.g. "2x explorer parallel"
```

Note the placeholders are deliberately unfilled: a fully-formed example here would echo into
every transcript that loads this file and inflate any later grep for real emissions — the same
contamination that makes P0–P8 string counts worthless. Never put a realistic sample log line
in this file.

The version prefix is what makes log audits attributable — without it, which ruleset was in
force has to be guessed from prose, which is unreliable (P0–P8 string counts are worthless:
this file's own table echoes into transcripts). State agents, steps, expected agent-span
minutes, and the token lower bound in that one line, every time. Get an explicit go/no-go when **either** the total exceeds **~30 min** or **any
single step** does. Recomputed on measured spans: **P3 ≈ 50 min** (14.9 + 14.3 + 8.2 + 12.9
— placeholder-bearing, planner is n=1), **P4 ≈ 90 min** (3 × [14.3 + 12.9] + verifier 8.2).
Be honest about what follows: the **total** trigger fires on both computable pipeline
patterns, so for P3 and P4 a go/no-go is effectively mandatory rather than exceptional — say
so plainly instead of presenting it as a rare event. P5 cannot be totalled at all
(hypothesizer is unmeasured); nearest-role substitution puts it at 30–38 min, straddling the
bar, so treat it as gate-triggering and label the total placeholder-bearing. The **per-step**
trigger is the one that catches surprises, and its evidence is thin: only verifier's p90
(36.5, n=6, a single point) exceeds 30 min, so treat it as a stall guard, not a calibrated
threshold. P1 and P2 are single-wave and normally clear both; P6 is **two** steps — the
analyzer wave, then the synthesizer — and synthesizer is unmeasured, so total it with a
labelled placeholder rather than assuming it clears.

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
| Return contract — the reverse case | a global or session instruction rule can **override** a role's own output contract: an `explorer` whose role file says "final message = the map only" returned a single `// <reason>` annotation and nothing else, converting a paid run into zero. Restate the required final-message shape in **every** spawn prompt. Prevention is unproven — 9 spawns, 0 failures, against an ~18% base rate for `explorer`, so that is weak evidence; the §4 detector is what actually catches it. Restate it for **every** role, not recon only: thin results track spawn volume rather than role — the 8 observed were `analyzer` (3) and `critic` (5) simply because those were the roles being run, both at rates *below* explorer's base rate. Keep the restatement to one line so it costs nothing: name the required final-message shape, add "nothing else", and forbid a `//` annotation as the message's opening or its entirety, matching the §4 detector. Do not paste a fully-formed literal contract into this file — same contamination rule as the log line in §2 |
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
**v2.3.2** (2026-08-11)
