# Skills

A home for reusable AI coding skills and agents, organized by tool.

| Folder | Tool | Contents |
|---|---|---|
| `claude/` | Claude Code | `skills/orchestrate` multi-agent pattern selector, `skills/agents-status` background-job status report + 10 agent roles under `agents/`, and `CLAUDE.example.md` — the always-loaded rules that make the toolkit work. |

Future tools (Codex, Cursor, etc.) get their own top-level folder.

## Installing the Claude toolkit

Copy `claude/` into a project as `.claude/` (or symlink it):

```bash
cp -r claude /path/to/project/.claude
```

Claude Code then auto-discovers the skills and the agent roles.
Roles are defined in `claude/agents/README.md`.

Then merge `claude/CLAUDE.example.md` into your own `CLAUDE.md` — do not copy it over
one. A skill only loads when it is triggered, so the few rules you cannot afford to miss
(collect every agent, state cost and time first, read-only roles cannot write) belong in
the always-loaded file. The `orchestrate` skill holds the detail.

## Contributing

Fork the repo, make your changes in your fork, and open a Pull Request.
Ideas for new patterns, roles, or improvements to existing ones are welcome.

## License

[MIT](LICENSE) © Mikhail Koryakov
