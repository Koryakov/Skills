# Example — `CLAUDE.md` fragment for agent work

Merge this into your own `~/.claude/CLAUDE.md` (user-level) or a project `CLAUDE.md`.
**Do not copy this file over an existing one** — it is a fragment, not a whole config.

Why it exists: `CLAUDE.md` is always loaded; a skill is not. The `orchestrate` skill
loaded in only 3 of 32 sessions that actually ran agents, so the rules that must never be
missed are mirrored here. Everything else — patterns, measured timings, cost arithmetic —
stays in the skill, which is where the detail belongs.

---

## Working with agents

- **Solo by default.** Using agents must earn its cost.
- **State cost and expected minutes before spawning.** Ask for a go/no-go if the total, or
  any single step, exceeds ~30 min.
- **Collect every agent you launch.** An abandoned agent is 100% wasted spend.
- **Read-only roles cannot write files** (analyzer, critic, planner, hypothesizer,
  explorer, verifier, researcher, built-in Explore/Plan). Take their findings from the
  reply and persist them yourself.
- **The first review round is always a full sweep.** Never skip or narrow it to save time.
- **Check a role's `model:` before overriding it** — overriding to the value already pinned
  does nothing, silently.

Full detail — patterns, measured timings, cost arithmetic: the `orchestrate` skill.
