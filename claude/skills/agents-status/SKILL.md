---
name: agents-status
description: Status of everything running in the background — subagents, long-running shell jobs, and monitors. Use when the user asks what is running, whether work is alive, stuck or looping, or how far a background job has got. Combines task enumeration, log growth, and disk activity into one table with a verdict per job.
user-invocable: true
allowed-tools:
  - Bash
  - PowerShell
  - Read
---

# agents-status

One-pass report on **every background job in this session** — subagents, backgrounded shell commands, and monitors. The user invokes this because the transcript does not say whether work is progressing, finished, or wedged.

Report all kinds. Long-running shell jobs are usually the most valuable rows: they run for hours and their logs prove progress, while subagents announce themselves on completion. **Never suppress the table because no subagent happens to be running**, and never drop a row just because you cannot identify it — hiding live work is this skill's worst failure mode, worse than an unlabelled row.

## Steps

1. **Enumerate — scan every session directory, not just this one.** Task files live at `<temp>/claude/<project-slug>/<session-id>/tasks/<taskId>.output`. The session id in the scratchpad path is **not reliably the one holding live files** (verified: a session's own agents wrote to a sibling directory while the scratchpad's parent held only hours-old files). So glob them all and let step 2 filter:
   ```
   ls -la "$LOCALAPPDATA/../Local/Temp/claude/<project-slug>"/*/tasks/ 2>/dev/null
   ```
   In bash the temp root is normally `/c/Users/<user>/AppData/Local/Temp/claude/<project-slug>/*/tasks/`. Widening the scan is free — the ID cross-reference in step 2 is the real gate, so extra directories cannot produce false rows. Do **not** look under `~/.claude/projects/`; that tree holds transcripts, not a `tasks/` directory.

2. **Classify** each entry by cross-referencing its task ID against this session's launches. The directory is not a registry — it also holds a file per *foreground* shell call, including the ones this skill just made:

   | ID matches | Kind | Report it? |
   |---|---|---|
   | an `Agent` result (`agentId`, or `taskId` for a remote agent) | subagent | yes |
   | a `Bash` call with `run_in_background` | shell job | yes — usually the most useful row |
   | a `Monitor` call (`taskId`) | monitor | yes |
   | nothing in this session, **but file is non-empty and its mtime is within ~5 min** | unknown — still moving | **yes**, provenance `launch not in context` |
   | nothing in this session, file empty or stale | transient foreground shell (often this skill's own) | discard |

   The unknown-but-moving row is not optional. A background job's ID appears in exactly one place in the transcript, so it is lost after compaction, and a job launched by a subagent or via `Ctrl+B` was never in this context at all. Growth is what distinguishes real live work from this skill's own finished calls. Note a large *foreground* result file can also sit here (hardlinked into `tool-results/`), so staleness, not size, is the discriminator.

   Print `nothing running in the background` only when no entry matches any reportable kind.

3. **Get the right signal per kind** — reliability differs sharply and the report must not blur them:

   | Kind | Read its output file? | Liveness signal | Reliability |
   |---|---|---|---|
   | shell job / monitor | **yes** — the documented way | size + mtime, then a bounded `tail -20` | good, but see the buffering caveat |
   | async / remote subagent | **only if sanctioned** (below) | `outputFile` if offered, else notification + project-file writes | weak |
   | local (synchronous) subagent | **no** | cannot be mid-run — the parent is blocked awaiting it | n/a |

   Shell jobs: check size first, then tail a bounded slice — never `cat` a whole log; these reach tens of megabytes. **Buffering caveat:** an unflushed writer (`python` without `-u`, a pipe, .NET without flush) means a perfectly healthy job's file does not grow. Absence of growth is therefore never proof of a stall.

   Subagents: the `TaskOutput` tool is deprecated and its description warns that a local agent's `.output` is a symlink to the full conversation JSONL that will overflow context. But a backgrounded agent's result carries `outputFile` ("for checking agent progress") together with `canReadOutputFile`. Reconcile them this way: **tail `outputFile` only when the launch result offered it and `canReadOutputFile` is true, and only ever a bounded tail.** Otherwise do not touch it. These files are often 0 bytes — empty is not failure.

4. **Disk heartbeat** — for work that writes files. Fall back to cwd when `src/` is absent, or an empty result reads as "no activity" and manufactures a false stall. PowerShell (wrap as `powershell -c "…"` if invoking through Bash):
   ```
   $p = if (Test-Path src) { 'src' } else { '.' }
   Get-ChildItem -Path $p -Recurse -File -ErrorAction SilentlyContinue | Where-Object { $_.FullName -notmatch '\\(\.git|node_modules|bin|obj)\\' -and $_.LastWriteTime -gt (Get-Date).AddMinutes(-10) } | Sort-Object LastWriteTime -Descending | Select-Object -First 15 LastWriteTime, FullName
   ```
   Unix (portable — avoid GNU-only `-printf`):
   ```
   d=src; [ -d "$d" ] || d=.; find "$d" -type f -mmin -10 -not -path '*/.git/*' -not -path '*/node_modules/*' -exec ls -ld {} + 2>/dev/null | tail -15
   ```
   Then, guarded so a non-repo does not error: `git rev-parse --git-dir >/dev/null 2>&1 && git status --short`

5. **Render one narrow table**, longest-running first. **Five columns, hard maximum** — seven wrapped and garbled in real use, so keep free text out of cells:
   ```
   task | job | status | elapsed | last activity
   ```
   - `job` carries a short kind prefix plus an abbreviated role: `shell: range 3 (ids 953706+)`, `agent: explorer`, `monitor: fill progress`. Truncate to ~30 chars.
   - `status` is the verdict icon plus its short label — don't spend a column on both.
   - **Latest output goes below the table, not in a cell** — one bullet per job, full width, so a log line never has to wrap:
     ```
     - br45mbdt0 — "Loaded 18,986 confidential sources; streaming 730 attributes over 18 tie hops"
     ```
   - **shell job** — the file grows, so its mtime *is* last activity.
   - **subagent** — the task-file mtime marks **spawn time** and does not advance while it works (measured). It is age, not activity: leave `last activity` blank rather than misreport it.
   - **monitor** — whether its task file grows is **unverified**; treat its mtime as unreliable and prefer the events it already emitted into the conversation.
   - **remote subagent** — work runs off-machine, so no local file or disk signal applies; its handle is a `sessionUrl`.

6. **Verdict per job** — pick exactly one; the list is exhaustive and applies to **all kinds**:
   - ✅ **alive** — the output file grew, or a project file was written, in the last 5 min.
   - 🏁 **finished** — a notification or result arrived, or the process exited. Summarize a subagent from its notification (a backgrounded launch result carries no content to summarize). Completed jobs stay listed until cleanup, so a stale-looking entry is usually this.
   - 🛑 **failed** — failure reported, or the log tail shows an error / traceback / OOM.
   - ❓ **running, nothing new since `<time>`** — still going, no fresh signal. Applies to a thinking subagent *and* to a quiet shell job or monitor: a load job logging one line per range, a bulk insert, or an event-waiting monitor can be silent for a long time by design. Print elapsed and say plainly that working and wedged cannot be told apart from outside. Do **not** call it stuck.

   Escalate ❓ to a *suspected* stall only when elapsed time is far past what the work warranted — and label that as a judgement about elapsed time, not as evidence.

7. **Actionable tail** — at most 3 short recommendations, e.g. "two ranges still unstarted — they begin when this pair exits", or "this one has run 4× longer than its siblings with no output; consider restarting it narrower".

## Rules

- **Never pull a full conversation transcript into context.** No `TaskOutput`; no reading a local agent's `.output`; a bounded tail of a shell log, or of an agent's `outputFile` when `canReadOutputFile` permits it.
- **Bounded reads only** — size check, then a tail. Never the whole file.
- **Never hide a row.** An unidentifiable but growing job gets reported with unknown provenance.
- **Never infer a stall from missing evidence.** Absence of a signal is absence of a signal — report it as such.
- **Do not block.** This is a snapshot, not a wait.
- **Do not restart or kill anything.** Only report; stopping is the user's call (`Ctrl+C`, or `Ctrl+X Ctrl+K` for background agents).
- **Single pass.** Don't loop, and don't rely on state from a previous invocation; there is none. Pair with `/loop` for periodic checks.
- **Respect CWD.** Scan the current project only.
- **Be terse.** A table, verdicts, at most 3 recommendations. No preamble.
