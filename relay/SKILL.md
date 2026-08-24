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

Preflight passed, check the gate once, immediately — already open means straight to step 2. Otherwise, take the base-independent reading first: the downstream ticket body, the spec and ADRs it cites, a test plan — none of it depends on the upstream diff, and done now it comes off the critical path for free. Code recon waits for the gate: the upstream leg is rewriting the very surfaces a map would describe, and a pre-gate map arrives stale. Then arm the tripwire and go idle: a Monitor (`persistent: true`) running the loop below. The shell does the waiting; the session wakes exactly once, on the line the loop emits. Never poll from the session itself — every model-side wake pays the whole accumulated context again just to find the gate shut. Idle means zero further tool calls until the tripwire's line arrives — a capped Bash sleep-loop "just to check" is a second poller that defeats the first.

```bash
start=$(date +%s)
while :; do
  s=$(gh issue view <upstream> --json state,stateReason \
        -q '.state + " " + (.stateReason // "")' 2>/dev/null || echo RETRY)
  case "$s" in
    CLOSED\ NOT_PLANNED) echo "gate dead: upstream closed as not-planned"; exit 0 ;;
    CLOSED*)             echo "gate open: upstream closed"; exit 0 ;;
  esac
  [ $(( $(date +%s) - start )) -ge <expiry> ] && { echo "gate expired: still shut after <expiry>s"; exit 0; }
  sleep <interval>
done
```

`<expiry>`: 43200 (12h), unless the invocation names another — a dispatcher launching the tail of a long chain names more: nine-leg chains have already produced 4h gate waits, and expiry is measured from launch, not from the upstream's start. The expiry is wall-clock, independent of the interval. The loop emits a line on every terminal state, so silence always means still-waiting. Step 1 is complete when the preflight has passed and the tripwire's one line has been interpreted:

- `gate open` — two `gh` calls before moving. First re-check the downstream ticket is still OPEN: hours have passed, and a closed downstream means the baton already passed — stop and report. Then check the upstream ticket's PR: merged, or open to stack on, means run your leg — and the ticket's closing comment should link that PR (the leg's baton comment); a close with no PR behind it, or no baton comment, is a mis-close, not a baton: stop and report the upstream state — never implement on a dead foundation.
- `gate dead` — stop and report the upstream state.
- `gate expired` — stop and write `needs input:` with the upstream state — ticket status, last comment, PR status.

A dispatcher's gate-open message counts as the tripwire line — monitor delivery inside a background session is unreliable, so a dispatched leg's real wake usually arrives by message. On any wake, before acting: verify the upstream state yourself with one `gh` call, read your monitor task's output file to learn what it actually did, and TaskStop it if still running — never assert a task's fate without its output.

## 2. Run your leg

1. Create your own worktree: `git worktree add <repo-root>/.claude/worktrees/issue-<N> -b issue-<N> <base>` — never EnterWorktree (it fails from a pinned-cwd subagent), and a self-created worktree is one you can also remove at cleanup. The branch name is the one `~/.claude/skills/dispatch/leg-contract.md` dictates, so the dispatch tripwire and downstream legs can find your work.
2. Pick the base: upstream PR merged → branch from the default branch. Unmerged → branch from the upstream PR's head, and set your PR's base to that branch (stacked PR).
3. Read `~/.claude/skills/implement-relay/SKILL.md` and follow it for `<downstream>`.
4. Push the branch; open a draft PR, noting the stacked base in the body if there is one. Run the review tail and add the `Review:` verdict line per leg-contract clause 3 — the reviewers' own `Verdict:` comments are what the merge gate cross-checks. Mark the PR ready (`gh pr ready`) before the close below: a draft is not a delivered PR.

## 3. Land, then pass the baton

Complete = your PR is open and marked ready, tests are green, review is done.

- Complete, and the repo's CLAUDE.md grants **self-landing** → land your own PR per the grant's gates before closing the ticket. The grant is durable pre-authorization: merge without asking, background agent or not. A PR the gates hold stays open with a comment naming the held gate — the ticket still closes below, and the next leg stacks on your head branch.
- No grant → leave the PR open; landing is the human's job downstream.
- Either way: close the downstream ticket with a comment linking your PR (and its stacked base, if unmerged). This close IS the pass: the next runner gates on it. A leg run without the pass loses the whole race.
- Blockers that need the human's decision: when the repo's CLAUDE.md carries a **needs-from-you** policy, follow it first — a blocker with a defensible default ships default-and-log, and the leg still completes and passes the baton. Only a blocker with no defensible default grounds the leg: leave the ticket open, comment the blockers on it, end with `needs input:`.
