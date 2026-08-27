# Spawning reference

A teammate is any agent that runs the brief and talks to the team through
`teamctl`. There are five ways to get one, and they mix freely inside a single
team — a Claude Code subagent, a headless Codex process in a tmux pane, and a
human in a second Cursor window can share one task list.

## Adapters

### `host` (default)

Registers the member, renders the brief to `<team>/briefs/<name>.md`, prints it,
and stops. **You** launch it with the mechanism your host already has. Nothing
is bypassed: the teammate inherits your host's permission model and credentials.

```bash
teamctl spawn --name sec --role security-reviewer --task T1
teamctl brief --name sec            # print it again, any time
```

| Host | How to launch the brief |
|---|---|
| Claude Code | Agent/Task tool with `name=sec` (a named subagent under agent teams becomes a real teammate) |
| Devin CLI / Windsurf | `run_subagent`, profile `subagent_general`, `is_background=true`, brief as the task |
| Cursor | Background agent, or a second chat in the same workspace |
| Codex / Copilot / Gemini / Amp / OpenCode / Aider | A new session in this repo, brief as the first message |
| A human | Paste `teamctl brief --name sec` into any editor's agent |
| CI | A job step running the brief through any of the CLIs below |

The member sits in `spawning` until it runs `teamctl join`. `teamctl members`
shows who has actually arrived; `doctor` flags a member that never joined.

### `process`

`teamctl` launches a detached agent CLI with the brief as its prompt, in the
project root, with `TEAMCTL_*` exported, output tailing to
`<team>/logs/<name>.log`.

```bash
teamctl spawn --name api --role backend --task T1 --adapter process --dry-run
teamctl spawn --name api --role backend --task T1 --adapter process --model sonnet
```

**Always `--dry-run` first** and read the command. These run unattended, which
means they run with that CLI's auto-approval flags and your credentials. That is
the only way a background teammate does not hang forever on a permission prompt,
and it is why `host` is the default.

### `tmux`

Same as `process`, in its own window of tmux session `team-<team>`:

```bash
teamctl spawn --name api --adapter tmux --task T1
tmux attach -t team-<team>          # watch them work
```

Requires `tmux` on PATH. `teamctl end` kills the session it created (keep it
with `--keep`). Orphans: `tmux ls` then `tmux kill-session -t team-<team>`.

### `worktree`

A `git worktree` per teammate on branch `team/<team>/<name>`, then `process` (or
`tmux` when you are already inside tmux) inside it. The team directory stays
shared through `$TEAMCTL_HOME`, so tasks and mail are common while working trees
are completely separate — parallel implementation with zero file contention.

```bash
teamctl spawn --name api --adapter worktree --task T1
teamctl spawn --name ui  --adapter worktree --task T2
# integrate when they finish
git merge team/<team>/api team/<team>/ui
teamctl end --prune-worktrees
```

Give the integration step its own task with `--deps` on both, so nobody merges
half-finished work.

### `print`

Brief only, for pasting somewhere else (another machine, a ticket, a human).

## Verified CLI invocations

Auto-detected from PATH in this order. Flags were checked against each vendor's
current CLI reference; override any of them with
`teamctl config --spawn-command '...'` or `spawn --cmd '...'`. Placeholders:
`{brief_file}`, `{brief_quoted}` (= `"$(cat <file>)"`), `{model}`, `{cwd}`,
`{name}`, `{bin}`.

| CLI | Command teamctl builds | Notes |
|---|---|---|
| `claude` | `claude -p {brief_quoted} --dangerously-skip-permissions [--model M]` | No `--cwd` flag; child process cwd is set instead. Run as non-root |
| `codex` | `codex exec - --cd {cwd} --sandbox workspace-write --skip-git-repo-check < {brief_file}` | `-p` means `--profile`, not prompt. Default sandbox is read-only, so raise it. Needs a git repo or `--skip-git-repo-check` |
| `cursor-agent` / `agent` | `{bin} -p {brief_quoted} --force --trust --workspace {cwd}` | `-p` alone can leave it interactive; `--force` is required. Can hang after finishing in CI — the process adapter's log plus `doctor` will show it |
| `devin` | `devin --prompt-file {brief_file} --permission-mode dangerous --respect-workspace-trust false -p` | `--respect-workspace-trust false` is mandatory for headless in an untrusted dir |
| `gemini` | `gemini -p {brief_quoted} --approval-mode yolo` | `--yolo` and `--approval-mode` are mutually exclusive |
| `copilot` | `copilot -p {brief_quoted} --allow-all-tools --no-ask-user -s --no-color` | `--allow-all-tools` is required for programmatic use; stdout is noisy by design |
| `opencode` | `opencode run --dir {cwd} --auto {brief_quoted}` | On `run`, `-p` means `--password`. `--auto` still honours explicit deny rules |
| `qwen` | `qwen -p {brief_quoted} --approval-mode yolo` | Gemini-CLI fork; also has `--max-session-turns`, `--max-wall-time` |
| `amp` | `amp -x {brief_quoted} --dangerously-allow-all` | **No model flag exists.** `--model` is ignored for Amp |
| `crush` | `crush run --cwd {cwd} --quiet {brief_quoted}` | `--yolo` placement varies by build; `run` is already non-interactive |
| `aider` | `aider --message-file {brief_file} --yes-always --no-pretty --no-stream [--model M]` | Usually also needs the files to edit; the brief's `paths` tell it which |

Sanity-check a vendor's flags on your installed version before relying on it:

```bash
teamctl spawn --name probe --adapter process --dry-run
<cli> --help | head -40
```

## Custom commands

Anything that takes a prompt can be a teammate — including a shell script, a
Python agent, or a remote session:

```bash
teamctl config --spawn-command 'ssh build-box "cd /repo && claude -p \"$(cat {brief_file})\" --dangerously-skip-permissions"'
teamctl spawn --name remote --adapter process --task T3
```

For a remote teammate, the team directory must be reachable from both sides
(shared mount, or `--home` on a network path). Otherwise use `print` and let the
remote agent report back through a task it can reach.

## Model selection

`spawn --model <m>` (or `teamctl config --spawn-adapter`/`--spawn-command` with
`{model}` baked in) applies to `process`/`tmux`. For `host`, choose the model
when you launch the subagent — most hosts let you pick per spawn. Amp has no CLI
model flag at all.

## Nesting and caps

- `max_depth` (default 0) — only the lead may spawn. Raise it
  (`teamctl config --set max_depth=1`) to let teammates delegate too; nested
  teams inherit `TEAMCTL_DEPTH` and are bounded by the same number.
- `max_members` (default 12) — refuses further spawns; `--force` overrides.
- `budget --usd-cap/--token-cap` — an exhausted budget refuses spawns (exit 4).
  Agents cannot measure their own spend reliably, so report it explicitly:
  `teamctl budget --add-usd 0.42 --add-tokens 120000`.

## Shutting teammates down

```bash
teamctl shutdown request sec --reason "review is merged"
# the teammate answers:
teamctl shutdown respond --approve --summary "3 findings, all filed"
teamctl shutdown respond --reject  --reason "mid-refactor, 5 minutes"
```

Approval releases every task the teammate holds and marks it `left`, so nothing
is stranded. `teamctl end` requests shutdown from everyone, writes the report,
and keeps the record. A teammate that just vanishes is handled by lease reclaim
instead — the work returns to the pool automatically.

## CI

```bash
export TEAMCTL_HOME="$PWD/.agentteam"
teamctl init --team ci --goal "$PR_TITLE" --setting max_depth=0
teamctl task import --file .ci/review-plan.json
for r in sec perf qa; do teamctl spawn --name $r --adapter process --task T${r}; done
teamctl wait --for all-done --timeout 3600 || true
teamctl report --out review.md
teamctl doctor --strict          # non-zero if anything stalled or went unverified
```

`doctor --strict` is the gate: it fails the job on stalls, cycles, path
collisions, unanswered plans, or work closed without verification.
