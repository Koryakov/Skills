# Orchestrate — agent pattern selection

A Claude Code skill that reads a task, picks the **cheapest orchestration
pattern that fits** (P0–P8), and runs only the roles and resources actually
needed. The coordinator is the main session; specialized subagents do the work
in isolated context windows and return only their findings.

Skill definition: [SKILL.md](SKILL.md) · Role definitions: [../../agents/README.md](../../agents/README.md)

![Orchestration in one flow](images/overview.png)

> **Diagram reflects skill v1.0.** Predates P8, the cost arithmetic, and the
> result-collection rule. See the revision history in [SKILL.md](SKILL.md).

## How it manages context

The main session stays clean. A single `brief.md` (task, constraints, file
pointers) is written once and read by every agent. Each agent works in its own
context window, so dead ends and intermediate reasoning die there — only
curated findings return to the main session.

![How the skill manages context](images/context-management.png)

> **Diagram reflects skill v1.0.** Shows each agent writing its own findings file;
> since v2.0 the read-only roles return findings in their final message and the
> coordinator persists them.

## How it picks model and effort

Cheap settings for supporting work; stronger models and higher effort only
where depth matters (analysis, architecture, criticism, verification). Both are
pinned in each role's own file. If a result is weak, the same role is rerun —
not padded with new agents — but only the **model** can be changed when starting
an agent. Effort and turn limits come from the role file and cannot be
overridden at spawn time, so an upgraded model still runs under the role's
original limits.

![Model and effort selection by role](images/model-and-effort.png)

> **Diagram reflects skill v1.0.** Effort cannot be set when spawning an agent —
> it comes from the role file. Since v2.1 only `model` is a spawn-time lever.

## The patterns

### P0 · Solo
No subagents. Fits in context, ≤ ~3 files, iterative with the user. The default.

![Solo](images/p0-solo.png)

### P1 · Map-Explore (fan-out recon)
Parallel explorers map different areas; the coordinator synthesizes. Use when
scope is unknown or spread across several repo areas.

![Map-Explore](images/p1-map-explore.png)

### P2 · Research & Decide
Researchers explore options in parallel, an analyzer checks repo fit, and the
coordinator compares. Use for tech choices and lock-in decisions needing evidence.

![Research and Decide](images/p2-research-and-decide.png)

### P3 · Pipeline Build
planner → implementer → verifier → critic. Use for a multi-file feature with a
clear spec.

![Pipeline Build](images/p3-pipeline-build.png)

### P4 · Generator–Critic Loop
implementer ⇄ critic (≤3 rounds) → verifier. Use for correctness-critical
changes with high blast radius.

![Generator-Critic Loop](images/p4-generator-critic-loop.png)

### P5 · Competing Hypotheses
A hypothesizer proposes theories, analyzers test them in parallel, a critic
ranks. Use for hard bugs with conflicting evidence or failed prior fixes.

![Competing Hypotheses](images/p5-competing-hypotheses.png)

### P6 · Map-Reduce Audit
The coordinator partitions the corpus across parallel analyzers; a synthesizer
merges findings. Use for large-corpus reviews, audits, or migration sweeps.

![Map-Reduce Audit](images/p6-map-reduce-audit.png)

### P7 · Team Feature Ownership
A lead plus teammates own layers (API, UI, tests) and message each other
directly. Use for independent workstreams that need teammate↔teammate coordination.

![Team Feature Ownership](images/p7-team-feature-ownership.png)

### P8 · Recon-then-Multi-Plan
Three built-in Explore agents map the ground in parallel, then two or three
built-in Plan agents each design a solution under a **deliberately different
bias** — security-first, minimal-change pragmatic, event-driven — and the
coordinator picks one or merges them. Use when the design space is genuinely
open and you want real alternatives rather than one plan and its variations.
Don't use it when a single approach obviously suffices: that is P3.

Plan output is long (~15.5k characters, the largest of any role) and Plan
cannot write files, so the coordinator persists each variant.

> No rendered diagram yet — schema below.

```mermaid
flowchart TD
    C1([Coordinator]) --> E1[Explore · area A]
    C1 --> E2[Explore · area B]
    C1 --> E3[Explore · area C]
    E1 --> C2([Coordinator · merges recon])
    E2 --> C2
    E3 --> C2
    C2 --> P1[Plan · security-first]
    C2 --> P2[Plan · minimal-change]
    C2 --> P3[Plan · event-driven]
    P1 --> C3([Coordinator · picks or merges])
    P2 --> C3
    P3 --> C3
```

## Operating rules the diagrams don't show

- **Cost is stated before spawning** — the sum of each planned agent's measured
  median, so the work can be vetoed before it runs. Treat it as a middle
  estimate: observed runs land roughly fourfold either side of it.
- **Every launched agent is collected** before the turn ends. An uncollected
  agent is spend with nothing to show for it.
- **Read-only roles return findings in their final message** and the coordinator
  writes the files — most roles have no ability to write.
- **A returned "not found" is unverified** until the coordinator confirms it, and
  a near-empty final message counts as no answer at all.
- **Scratch files live outside the repository by default**, so they cannot be
  committed by accident.
