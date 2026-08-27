# Playbooks

Complete, copy-adaptable runs. Each one names the decomposition, the verify
commands, and the briefs — the three things that decide whether a team produces
something or just spends tokens.

Assumes `teamctl` on PATH (`teamctl install` if not).

---

## 1. Parallel review

Three independent lenses on the same diff. No writes to source, so no lease
contention; each reviewer's deliverable is a file, which makes the verify
command trivial and real.

```bash
teamctl init --team review --goal "review PR 142 before merge"
mkdir -p reports

teamctl task add "security review of PR 142" --tags review \
  --verify "test -s reports/security.md" \
  --detail "Exploitable issues only. Each finding: severity, file:line, PoC, fix."
teamctl task add "performance review of PR 142" --tags review \
  --verify "test -s reports/perf.md" \
  --detail "Hot paths, allocations, N+1 queries, added latency. Numbers, not adjectives."
teamctl task add "test-coverage review of PR 142" --tags review \
  --verify "test -s reports/tests.md" \
  --detail "What changed without a test, and the specific test that should exist."

teamctl spawn --name sec --role security-reviewer --task T1 --brief \
  "Write reports/security.md. Read the diff with: git diff main...HEAD
   File every real issue with: teamctl finding add --claim '...' --evidence 'file:line + PoC'
   --confidence high|medium|low --broadcast
   Then review the other two reviewers' findings and vote:
   teamctl findings --full ; teamctl finding vote F-xxxx --agree|--disagree --note '...'"

teamctl spawn --name perf --role performance-reviewer --task T2 --brief \
  "Write reports/perf.md. Measure before you claim. Same finding + vote protocol as sec."

teamctl spawn --name qa --role test-reviewer --task T3 --brief \
  "Write reports/tests.md. Name the missing test, not the missing coverage percentage.
   Same finding + vote protocol."

teamctl wait --for all-done --timeout 3600
teamctl findings --full            # what survived cross-review
teamctl report --out REVIEW.md
teamctl end
```

Why it works: each reviewer owns a distinct output file, the verify command
proves the file exists and is non-empty, and cross-voting forces them to read
each other instead of filing three parallel monologues.

---

## 2. Competing hypotheses (debugging)

One agent finds one plausible cause and stops. Several agents that must attack
each other's theories find the real one. The findings ledger is the mechanism —
without it, this is just three transcripts.

```bash
teamctl init --team bug --goal "app exits after one message instead of staying connected"
mkdir -p findings

teamctl task add "hypothesis A: server closes the socket after first response" \
  --verify "test -s findings/A.md" --tags hypothesis
teamctl task add "hypothesis B: client event loop exits when the queue drains" \
  --verify "test -s findings/B.md" --tags hypothesis
teamctl task add "hypothesis C: proxy idle timeout kills the connection" \
  --verify "test -s findings/C.md" --tags hypothesis
teamctl task add "converge: write the agreed root cause and the minimal fix" \
  --deps T1 T2 T3 --verify "test -s findings/ROOT-CAUSE.md"

teamctl spawn --name a --role investigator --task T1 --brief \
  "Investigate hypothesis A only. Write findings/A.md.
   Publish every conclusion:  teamctl finding add --claim '...' --evidence '<log, trace, repro>'
                              --confidence high|medium|low --broadcast
   Then actively try to DISPROVE the others:
     teamctl findings --full
     teamctl finding add --claim '<A survives because ...>' --refutes F-xxxx --evidence '...'
     teamctl finding vote F-xxxx --disagree --note '<what the evidence actually shows>'
   A theory you cannot reproduce is not a finding. Say so."

teamctl spawn --name b --role investigator --task T2 --brief "<same, hypothesis B, findings/B.md>"
teamctl spawn --name c --role investigator --task T3 --brief "<same, hypothesis C, findings/C.md>"

teamctl wait --for all-done --timeout 3600
teamctl findings --full           # tallies decide, not eloquence
teamctl next --claim             # the lead takes T4 and writes ROOT-CAUSE.md
```

Rules that matter: no hypothesis owner may edit another's file; every claim
needs evidence a third party could re-run; the converge task depends on all
three so it cannot start early.

---

## 3. Cross-layer feature

Disjoint `--paths` are the whole game. The integration task is dependency-gated
so nobody wires up half a feature.

```bash
teamctl init --team refresh --goal "add token refresh across api, ui and e2e"

teamctl task add "api: POST /auth/refresh with rotation" \
  --paths "src/api/**" "tests/api/**" --verify "pytest tests/api -q" --priority 1 \
  --detail "Rotate on use, 401 on reuse, 15m access / 7d refresh."
teamctl task add "ui: retry once on 401 using the refresh endpoint" \
  --paths "src/ui/**" "tests/ui/**" --verify "npm run test:ui" --priority 1
teamctl task add "e2e: expiry, refresh, reuse-detection flows" \
  --deps T1 T2 --paths "tests/e2e/**" --verify "npm run e2e"
teamctl task add "docs: document the refresh flow" \
  --paths "docs/**" --verify "test -s docs/auth-refresh.md" --priority 4

teamctl spawn --name api --role backend  --adapter worktree --task T1
teamctl spawn --name ui  --role frontend --adapter worktree --task T2
teamctl spawn --name doc --role writer    --task T4

teamctl wait --for task:T1 --timeout 3600     # gate on the dependency
teamctl wait --for task:T2 --timeout 3600
git merge team/refresh/api team/refresh/ui    # integrate the worktrees
teamctl next --claim                          # lead takes the e2e task
```

If a teammate needs a file outside its paths, it takes a lease instead of
guessing: `teamctl lock acquire "src/shared/http.ts" --task T2`, then releases
it. Two claims over the same glob are refused, which is exactly the collision
you want to hit at claim time rather than in a merge.

---

## 4. Large refactor with a required plan

Risky mechanical change: force a plan through review before anything is touched.

```bash
teamctl init --team refactor --goal "replace the ad-hoc cache with a single LRU layer"

teamctl task add "plan and execute the cache unification" \
  --paths "src/cache/**" "src/services/**" --plan-required \
  --verify "make test && make bench-cache" \
  --detail "No behaviour change. Bench must not regress more than 2%."

teamctl spawn --name arch --role architect --task T1 --brief \
  "This task requires an approved plan. Read the current call sites first, then:
   teamctl plan submit --task T1 --file plan.md
   teamctl wait --for plan:<id> --timeout 1800
   Only implement after approval. If rejected, revise and resubmit."

teamctl wait --for inbox --timeout 1800
teamctl plan list
teamctl plan review P-xxxxxx --reject --feedback \
  "No migration order and no rollback. Add both, plus a bench baseline number."
# ... teammate resubmits ...
teamctl plan review P-xxxxxx --approve
teamctl wait --for task:T1 --timeout 7200
```

`task done` is refused while `plan.status != approved`, so approval is a real
gate rather than a suggestion. Rejection without `--feedback` is refused too —
a teammate cannot act on "no".

---

## 5. Enforced quality gates

Team-wide policy that does not depend on anyone remembering it. Hooks live in
`.agentteam/teams/<team>/hooks/` and exit 2 to block with feedback.

Every implementation task must ship a verify command:

```sh
#!/bin/sh
# hooks/task_created
python3 - <<'PY' || exit 2
import json, sys
t = json.load(sys.stdin)["task"]
if "impl" in (t.get("tags") or []) and not t.get("verify"):
    sys.stderr.write("impl tasks need --verify (a command that proves it works)\n")
    sys.exit(2)
PY
```

Nothing is completed without an artifact:

```sh
#!/bin/sh
# hooks/task_completed
python3 - <<'PY' || exit 2
import json, sys
p = json.load(sys.stdin)
if not (p["task"].get("artifacts") or p.get("summary")):
    sys.stderr.write("attach --artifact <path> or --summary before closing\n")
    sys.exit(2)
PY
```

No teammate goes idle while claimable work remains:

```sh
#!/bin/sh
# hooks/teammate_idle
if teamctl next --json --quiet | grep -q '"task": {'; then
  echo "there is still claimable work: run teamctl next --claim" >&2
  exit 2
fi
```

Same effect as Claude Code's `TaskCreated` / `TaskCompleted` / `TeammateIdle`
hooks, but host-independent: they fire for every teammate whatever CLI it runs.

---

## 6. Resuming after a crash

Team state is on disk and outlives every session, including the lead's.

```bash
teamctl init --team refresh          # same name: attaches, does not reset
teamctl status                       # who was working on what
teamctl doctor --fix                 # reclaim dead leases, unblock satisfied deps
teamctl journal --tail 60            # what actually happened before the crash
teamctl spawn --name api --replace --task T4     # respawn a lost teammate
```

Rules of thumb:

- A task stuck `in_progress` with an expired lease and a silent owner is
  reclaimed automatically; `sweep --force` does it now.
- A member showing `lost` is gone; `spawn --replace` gives the seat to a fresh
  agent, which then claims the reclaimed task.
- Anything a dead teammate did not publish (`artifact add`, `finding add`, a
  task summary) is gone. That is the argument for small tasks and frequent
  publishing, not for trusting long-running agents.
