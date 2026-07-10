# Skills

A home for reusable AI coding skills and agents, organized by tool.

| Folder | Tool | Contents |
|---|---|---|
| `claude/` | Claude Code | `skills/orchestrate` multi-agent pattern selector + 10 agent roles under `agents/`. |

Future tools (Codex, Cursor, etc.) get their own top-level folder.

## Installing the Claude toolkit

Copy `claude/` into a project as `.claude/` (or symlink it):

```bash
cp -r claude /path/to/project/.claude
```

Claude Code then auto-discovers the `orchestrate` skill and the agent roles.
Roles are defined in `claude/agents/README.md`.
