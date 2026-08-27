# agent-skills

Portable skills for AI coding agents, using the `.agents/skills/` layout that
Devin CLI, Windsurf, Cursor, Claude Code, Codex, OpenCode, Zed and Copilot all
read. Each skill is a directory with a `SKILL.md` and whatever it ships
alongside.

## Skills

### `agent-teams`

Run a real team of agents on one job: a lead that plans and verifies, teammates
with their own context windows that claim work from a shared task list, message
each other, hold write leases on files, get plans approved, and have their work
machine-verified before it counts as done.

Host-agnostic by design — all shared state is files plus one dependency-free
CLI (`teamctl`), so a teammate can be a host subagent, a separate agent CLI
process, a tmux pane, a git worktree, or a human in another IDE window, mixed
freely in one team.

| | |
|---|---|
| Entry point | [`skills/agent-teams/SKILL.md`](skills/agent-teams/SKILL.md) |
| CLI | `skills/agent-teams/scripts/teamctl.py` — stdlib only, Python 3.8+, POSIX + Windows, no network |
| Docs | `reference/protocol.md`, `reference/spawning.md`, `reference/playbooks.md` |
| Tests | `python3 skills/agent-teams/tests/test_teamctl.py` — 91 end-to-end tests, ~80s |

What it enforces by mechanism rather than by instruction: verification gates on
completion, write leases on file globs, task claims as expiring leases with
automatic reclaim when a teammate dies, plan approval before risky work,
untrusted provenance on inter-agent messages, and pluggable hooks that can block
any action with feedback.

## Install

Clone anywhere and symlink (or clone straight into place):

```bash
git clone <this-repo> ~/.agents
python3 ~/.agents/skills/agent-teams/scripts/teamctl.py install   # ~/.local/bin/teamctl
```

`~/.agents/skills/` is picked up globally by Devin CLI and other tools that
follow the `.agents` standard. For hosts that use their own directory, link it:

```bash
ln -s ~/.agents/skills/agent-teams ~/.config/devin/skills/agent-teams   # Devin CLI / Windsurf
ln -s ~/.agents/skills/agent-teams ~/.claude/skills/agent-teams         # Claude Code
ln -s ~/.agents/skills/agent-teams .agents/skills/agent-teams           # vendored per project
```

To make a skill project-local instead, copy its directory into
`<repo>/.agents/skills/` and commit it.

## Conventions

- One directory per skill, `SKILL.md` at its root with YAML frontmatter
  (`name`, `description`, `argument-hint`, `triggers`).
- Anything executable goes in `scripts/`, long-form docs in `reference/`, tests
  in `tests/`. Keep `SKILL.md` scannable and push detail into `reference/`.
- No third-party dependencies unless a skill cannot work without them: skills
  run inside whatever environment the host agent happens to have.
