---
name: relay
description: "Watch an upstream ticket; when it closes, implement a downstream ticket, then pass the baton by closing it."
disable-model-invocation: true
---

# Relay

One leg of a ticket chain. Arguments: `<upstream> <downstream> [interval]` — two ticket references (issue number or URL) and an optional poll interval, default 1m. The interval is a shell `sleep` argument (`5m`, `2h`), so any duration works; polling is shell-side and costs nothing but a `gh` call, so frequent is fine.

Ticket state is the bus between sessions: your gate is the upstream ticket closing, and the next runner's gate is yours.

## 1. Preflight the baton, then arm the tripwire

Preflight first: one `gh issue view <downstream> --json state,title` call.
The arguments name the one leg this session runs, and the downstream ticket
must exist and be OPEN — anything else grounds the leg: stop and report both
tickets' states with `needs input:`, and leave the fix to the human. A
grounded leg stays grounded; substituting another ticket is the one forbidden
move.

Preflight passed, check the gate once, immediately — already open means straight to step 2. Otherwise arm the tripwire and go idle: a Monitor (`persistent: true`) running the loop below. The shell does the waiting; the session wakes exactly once, on the line the loop emits. Never poll from the session itself — every model-side wake pays the whole accumulated context again just to find the gate shut.

```bash
start=$(date +%s)
while :; do
  s=$(gh issue view <upstream> --json state,stateReason \
        -q '.state + " " + (.stateReason // "")' 2>/dev/null || echo RETRY)
  case "$s" in
    CLOSED\ NOT_PLANNED) echo "gate dead: upstream closed as not-planned"; exit 0 ;;
    CLOSED*)             echo "gate open: upstream closed"; exit 0 ;;
  esac
  [ $(( $(date +%s) - start )) -ge 43200 ] && { echo "gate expired: still shut after 12h"; exit 0; }
  sleep <interval>
done
```

The loop emits a line on every terminal state, so silence always means still-waiting. The 12h expiry is wall-clock, independent of the interval. Step 1 is complete when the preflight has passed and the tripwire's one line has been interpreted:

- `gate open` — two `gh` calls before moving. First re-check the downstream ticket is still OPEN: hours have passed, and a closed downstream means the baton already passed — stop and report. Then check the upstream ticket's PR: merged, or open to stack on, means run your leg. Closed with no PR at all: stop and report the upstream state — never implement on a dead foundation.
- `gate dead` — stop and report the upstream state.
- `gate expired` — stop and write `needs input:` with the upstream state — ticket status, last comment, PR status.

## 2. Run your leg

1. Enter a worktree.
2. Pick the base: upstream PR merged → branch from the default branch. Unmerged → branch from the upstream PR's head, and set your PR's base to that branch (stacked PR).
3. Run the `implement` skill on `<downstream>` — the personal one at `~/.claude/skills/implement`, invoked by bare name. Not `mattpocock-skills:implement`, which is user-invoked and unreachable from here.
4. Push the branch; open a draft PR, noting the stacked base in the body if there is one.

## 3. Pass the baton

Complete = your PR is open, tests are green, review is done. Merge is not required — merging is the human's morning job.

- Complete → close the downstream ticket with a comment linking your PR (and its stacked base). This close IS the pass: the next runner gates on it. A leg run without the pass loses the whole race.
- Blockers that need the human's decision → leave the ticket open, comment the blockers on it, end with `needs input:`.
