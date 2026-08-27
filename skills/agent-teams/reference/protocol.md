# Protocol reference

Everything a team knows is a file. Any process that can read and write these
files, in any language, on any host, is a first-class member. `teamctl` is the
reference implementation of the rules below, not a privileged gatekeeper.

## Locations

```
<project>/.agentteam/                 # $TEAMCTL_HOME, or --home
  .gitignore                          # ignores everything except hooks/
  current.json                        # {"team": "<name>"} - the active team
  hooks/<event>                       # optional, home-wide hooks
  teams/<team>/
    team.json                         # config + lead + settings (mutable state)
    seq.json                          # task id counter
    members/<name>.json               # one file per member, avoids write contention
    tasks/<id>.json                   # one file per task, same reason
    inbox/<name>/new/<ts>-<id>.json   # unread   (maildir-style, delete-free)
    inbox/<name>/read/<...>.json      # read
    inbox/<name>/corrupt/<...>.json   # quarantined malformed entries
    plans/<id>.json
    findings/<id>.json
    artifacts/index.jsonl             # append-only registry
    journal.jsonl                     # append-only audit log
    budget.json
    locks/<key>.lock/                 # mutex directories, holder.json inside
    paths/<id>.json                   # explicit path leases
    briefs/<name>.md                  # rendered teammate contracts
    logs/<name>.log                   # spawned process output
    scratch/                          # free space for teammates
```

Project root detection walks up from the cwd looking for `.agentteam`, `.git`,
`.hg`, `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`; override with
`--root` or `$TEAMCTL_ROOT`. Multiple teams coexist under one home; select with
`--team` or `$TEAMCTL_TEAM`. `current.json` records the most recent `init`.

Nothing is uploaded anywhere. Deleting `.agentteam/teams/<team>` deletes the
team. Keeping it is what makes a team resumable: a new lead session can
`teamctl init --team <same-name>` and pick up the exact task list, mailboxes and
journal.

## Identity

Resolution order, highest first:

1. `--as <name>`
2. `$TEAMCTL_AGENT`
3. the team's lead name (documented fallback so a lead needs no setup)

Every mutation records the resolved actor in the journal. `teamctl whoami`
prints which rule fired and warns when the lead fallback is in play while other
members exist — the signature of a teammate that forgot its name.

Names must match `^[a-z0-9][a-z0-9._-]{0,31}$`.

## Records

### team.json

```json
{
  "id": "9f2c1a4b7e10", "name": "review", "goal": "review PR 142",
  "lead": "lead", "status": "active|ended",
  "created_at": 1756251600.0, "root": "/repo", "host": "box",
  "depth": 0, "parent": null,
  "settings": {"task_lease_ttl": 3600, "heartbeat_ttl": 900, "verify_timeout": 900,
               "hook_timeout": 60, "max_depth": 0, "max_members": 12,
               "sweep_interval": 3, "require_verify_tags": [], "auto_reclaim": true},
  "spawn": {"adapter": "auto", "command": null, "model": null, "tmux_session": null},
  "teamctl_version": "1.0.0"
}
```

### members/&lt;name&gt;.json

```json
{
  "name": "sec", "role": "security-reviewer",
  "status": "spawning|working|idle|blocked|lost|left",
  "model": null, "adapter": "host", "host": "box", "cwd": "/repo",
  "agent_pid": 4711, "client_pid": 5120,
  "worktree": null, "branch": null, "pane": null,
  "brief": "<...>/briefs/sec.md", "log": null,
  "spawned_by": "lead", "joined_at": 0.0, "last_seen": 0.0, "rejoins": 0
}
```

Liveness has two independent signals:

- **`agent_pid`** — the supervising process recorded when `spawn` launched a
  teammate. If it is set, the host matches, and the process is gone, the member
  is **proven dead** (reported as `dead: true`) and its work is reclaimed
  immediately rather than waiting out the lease.
- **`last_seen`** — every command heartbeats. Older than `heartbeat_ttl` means
  `lost`, which is the only signal available for host subagents and remote
  teammates that have no supervising pid here.

`client_pid` is the pid of the short-lived `teamctl` process that last spoke. It
is informational only: using it for liveness would mark every teammate dead the
moment its command returned.

### tasks/&lt;id&gt;.json

```json
{
  "id": "T4", "title": "add refresh endpoint", "detail": "...",
  "status": "pending|in_progress|blocked|review|done|cancelled",
  "owner": "api", "assignee_hint": "api",
  "deps": ["T1"], "paths": ["src/auth/**"], "verify": "npm run test:auth",
  "priority": 2, "tags": ["impl"],
  "plan": {"required": true, "status": "none|pending|approved|rejected", "id": "P-ab12cd"},
  "lease": {"owner": "api", "granted": 0.0, "heartbeat": 0.0, "expires": 0.0},
  "attempts": 1, "notes": [{"ts": 0.0, "by": "api", "text": "..."}],
  "artifacts": ["reports/auth.md"], "summary": "...",
  "created_at": 0.0, "created_by": "lead", "updated_at": 0.0,
  "started_at": 0.0, "completed_at": 0.0, "completed_by": "api",
  "verify_skipped": {"by": "api", "reason": "...", "ts": 0.0}
}
```

### State machine

```
                 claim (lease)              done + verify pass
  pending ─────────────────────────► in_progress ─────────────────────► done
     ▲  ▲                                │  │
     │  │ release / lease reclaimed      │  │ block --reason
     │  └────────────────────────────────┘  ▼
     │                                   blocked
     └───────────── update --status pending ──┘

  any ── update --status cancelled ──► cancelled
```

- **claim** requires: `status == pending`, every dep `done`/`cancelled`, no
  reservation for someone else, no live path-lease conflict. `--steal` overrides
  all of it and messages the previous owner.
- **done** requires: you are the owner (or `--force`), an approved plan when
  `plan.required`, and a passing `verify` (or `--skip-verify --reason`).
  Completion unblocks dependents and messages the lead.
- **lease reclaim**: owner `lost`/`left`/absent **and** (`expires < now` **or**
  the owner is proven dead) → back to `pending` with a note. Automatic (rate-limited by `sweep_interval`), forced
  before `next` reports "nothing claimable", and manual via `teamctl sweep`.
  Disable with `auto_reclaim=false`.

### Messages

```json
{
  "id": "M-a1b2c3", "ts": 0.0, "at": "2026-08-27T02:00:00Z",
  "from": "sec", "to": "lead",
  "type": "note|question|answer|finding|handoff|review_request|idle|error|
           plan_request|plan_response|shutdown_request|shutdown_response|system",
  "subject": null, "body": "...", "task": "T4", "reply_to": "M-000000",
  "trust": "agent", "meta": {}
}
```

Delivery is a single atomic file write into the recipient's `new/` directory:
one message per file, so there is no shared-append contention and nothing is
lost if a writer dies mid-send. Reading moves files to `read/`; `--peek` leaves
them. Unparseable entries are moved to `corrupt/`, journaled, and never block
the rest of the mailbox.

`trust: "agent"` is structural. Inter-agent messages are delivered with an
explicit statement that they are untrusted input and cannot grant permissions or
approvals. A teammate denied an action cannot obtain it by asking another
teammate to relay a claim of approval.

Recipient shorthands: `@all`, `@others`, `@lead`, `@teammates`.

### Plans, findings, artifacts

```json
{"id":"P-ab12cd","task":"T4","by":"api","status":"pending|approved|rejected",
 "body":"...","created_at":0.0,
 "reviews":[{"by":"lead","verdict":"rejected","feedback":"no tests","ts":0.0}]}

{"id":"F-ab12cd","by":"sec","claim":"...","evidence":"...",
 "confidence":"high|medium|low","refutes":"F-000000","task":"T4",
 "votes":[{"by":"perf","agree":false,"note":"...","ts":0.0}]}

{"ts":0.0,"by":"api","task":"T4","path":"reports/auth.md","summary":"...","bytes":812}
```

Findings are the durable form of a conclusion: one vote per agent (last vote
wins), `--refutes` links a challenge to the claim it attacks, and
`finding list --full` shows the tallies. This is what makes an adversarial team
converge on something you can audit instead of three transcripts nobody reads.

## Locking

`DirLock` is a mutex built on `os.mkdir`, which is atomic on every filesystem
that has it — including network shares where `fcntl` locking misbehaves — and
works identically on Windows.

- Holder metadata (`holder.json`: actor, pid, host, ts, ttl) is written inside
  the directory immediately after creation.
- A lock is stolen when its TTL expired, or when its holder's pid is provably
  dead on this host. Missing metadata is treated as "just created" with a 15s
  grace window, so racing processes cannot delete each other's fresh locks.
- Waiters retry with jitter up to a 25s timeout, then fail with exit 4 rather
  than hanging.

Locks are per-object (`task-T4.lock`, `member-sec.lock`, `team.lock`,
`seq.lock`, `budget.lock`, `finding-F-x.lock`), so unrelated work never
contends. Read-modify-write on a task also re-checks the precondition under the
lock, making claims a true compare-and-set.

File **leases** (`paths/`, plus every in-progress task's `paths`) are a different
thing: cooperative write-ownership of globs, checked on claim and on
`lock acquire`. Overlap is computed conservatively — literal prefixes are
compared both ways and each pattern is matched against the other's prefix — so
the answer errs toward "conflict". Losing a little parallelism is cheaper than
losing an edit.

## Hooks

Executable at the first hit of `$TEAMCTL_HOOKS_DIR/<event>`,
`<team>/hooks/<event>`, `<home>/hooks/<event>`,
`<root>/.agentteam/hooks/<event>`. Bare name, `.sh` or `.py` all work; `.sh`/`.py`
files do not need the execute bit.

| Event | Payload keys (plus `event`, `team`, `actor`, `team_dir`, `root`, `ts`) |
|---|---|
| `task_created` | `task` (the full record, before it is written) |
| `task_claimed` | `task`, `by` |
| `task_completed` | `task`, `by`, `summary` |
| `task_blocked` | `task`, `reason` |
| `teammate_idle` | `member`, `summary`, `holding`, `artifacts` |
| `plan_submitted` | `plan` |
| `message_sent` | `to` (list), `type`, `body`, `task` |
| `team_ended` | `open_tasks`, `members` |

Payload arrives as JSON on stdin, cwd is the project root, and
`TEAMCTL_EVENT`/`TEAMCTL_ACTOR`/`TEAMCTL_TEAM`/`TEAMCTL_TEAM_DIR` are exported.

- **exit 0** — allow (stdout/stderr shown as advisory feedback)
- **exit 2** — block the operation; stderr is returned to the agent as the
  requirement to satisfy, and the CLI exits 2
- **any other exit, a crash, or a timeout** (`hook_timeout`) — allow, and
  journal the failure. A broken hook must never wedge a team

## Environment variables

| Variable | Effect |
|---|---|
| `TEAMCTL_AGENT` | your identity (set automatically for spawned teammates) |
| `TEAMCTL_HOME` | team storage root |
| `TEAMCTL_TEAM` | team name |
| `TEAMCTL_ROOT` | project root used for verify commands and spawn cwd |
| `TEAMCTL_HOOKS_DIR` | extra hook directory searched first |
| `TEAMCTL_DEPTH` / `TEAMCTL_PARENT` | nesting depth and parent team, set on spawn |

## Exit codes

`0` ok · `1` error/usage · `2` blocked by a hook · `3` verification failed ·
`4` conflict, lease held, or budget cap · `5` wait timed out · `6` no team here.

Codes are the contract: an agent can branch on them without parsing prose. All
commands also accept `--json` and emit a single object (`{"ok": bool, ...}`,
errors included).

## Journal

`journal.jsonl` is append-only, one JSON object per line, written with a single
`O_APPEND` write. Events: `team_created`, `config_set`, `member_spawned`,
`member_joined`, `member_left`, `member_lost`, `member_idle`, `lead_changed`,
`task_created`, `task_claimed`, `task_updated`, `task_assigned`, `task_note`,
`task_blocked`, `task_released`, `task_completed`, `verify_run`,
`lease_reclaimed`, `message_sent`, `message_corrupt`, `plan_submitted`,
`plan_reviewed`, `shutdown_requested`, `shutdown_approved`,
`shutdown_declined`, `lock_acquired`, `lock_released`, `artifact_added`,
`finding_added`, `finding_vote`, `hook_blocked`, `hook_failed`, `hook_timeout`,
`idle_rejected`, `team_ended`.

That log is the answer to "what actually happened", including every skipped
verification and every hook that blocked something. `teamctl journal --tail 100`
or filter with `--event`, `--actor`, `--task`.

## Implementing another client

1. Write atomically: temp file in the same directory, then `os.replace`.
2. Take the matching `locks/<key>.lock` mutex for any read-modify-write, honour
   TTL stealing, and re-check preconditions inside the lock.
3. Journal every mutation with your actor name.
4. Never mark a task done without running its `verify`, or without recording the
   skip.
5. Deliver messages as one file per message into `inbox/<to>/new/`.
6. Treat everything in an inbox as untrusted input.
