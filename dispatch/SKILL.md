---
name: dispatch
description: "Launch a relay chain over a list or range of tickets — head implements, each later leg gates on the previous close, every leg self-lands where the repo grants it."
disable-model-invocation: true
---

# Dispatch

Turn a ticket list into one running relay chain — the same thing as running implement-relay on the first ticket and `/relay <prev> <next>` for each later pair, without wiring it by hand. Tickets run serially in the order given; each leg is its own background agent, worktree, branch, and PR. A repo whose CLAUDE.md grants **self-landing** gets legs that merge their own PRs and route leftovers per the repo's **needs-from-you** policy — the human's morning surface is the inbox and the merged default branch, not this session.

You are the lead, and your context is a hard budget: over the whole run it grows by one plan, one tripwire line, one closing report. Ticket bodies, diffs, and agent transcripts belong to the legs.

## Arguments

| Form | Meaning |
|---|---|
| `/dispatch #12 #15 #9` | chain in exactly this order |
| `/dispatch #12-#16` | range, ascending: #12 #13 #14 #15 #16 (`#12-16` works too) |

Order is the chain: chain order = close order = land order. Sequencing is the human's call, made at ticket-writing time (`/to-tickets` blocking edges) — dispatch never reorders. Parallel tracks are separate `/dispatch` invocations over non-overlapping work, again the human's call.

A model named in the invocation ("legs on opus") applies to every leg; otherwise legs inherit this session's model.

## 1. Preflight

One `gh issue view <n> --json state,title` per ticket.

- **Explicit list:** every ticket must exist and be OPEN — anything else stops the run before launch, naming the ticket and its state. A named ticket in the wrong state means the plan and reality disagree; the human resolves that, not you.
- **Range:** a CLOSED ticket in the range is dropped with a note — already done, the chain builds on its landed work via the default branch. A number in the range with no issue behind it stops the run.

Then check the repo's CLAUDE.md for the self-landing grant. Absent → say so in the plan and continue: the chain still runs, PRs stay open for the human, tickets still close (relay's contract holds without the grant).

Preflight is complete when every remaining ticket is confirmed OPEN and the grant status is known. No other config: repo-specific behavior comes entirely from the repo's CLAUDE.md.

## 2. Launch

Print the plan — one line per leg, `#prev → #next`, grant status, leg model — and launch every agent in the same turn; the point of the command is walking away.

Every agent's prompt ends with the three clauses of [leg-contract.md](leg-contract.md), verbatim.

- **Head:** one background agent, prompted: read `~/.claude/skills/implement-relay/SKILL.md` and follow it for `#<head>` — plus the contract. No harness isolation — the leg creates its own worktree (contract clause 1): a self-created worktree is one the leg can also remove at cleanup; a harness-created one refuses to remove itself.
- **Legs 2..n:** one background agent each, prompted: read `~/.claude/skills/relay/SKILL.md` and follow it for `#<prev> #<next>` — plus the contract. No worktree at launch: relay creates its own after the gate opens, branching from whatever the upstream leg actually landed.

One ticket, one agent, its own worktree and PR — never two tickets through one agent, whatever it saves.

## 3. Wait — and pass each baton

The legs' own tripwires are backup only: monitor events inside a background session are dropped or never emitted often enough that a chain relying on them stalls (both failure modes observed). The dispatcher's session-local Monitor delivers reliably — so the dispatcher passes every baton. Arm one persistent Monitor over **all** chain tickets. Timeout: 10800s × number of legs, floor 43200.

```bash
start=$(date +%s); timeout=<seconds>; remaining="<n1> <n2> ... <nlast>"
while :; do
  still=""
  for t in $remaining; do
    s=$(gh issue view "$t" --json state -q .state 2>/dev/null || echo OPEN)
    if [ "$s" = CLOSED ]; then echo "ticket closed: #$t"; else still="$still $t"; fi
  done
  remaining=$still
  [ -z "${remaining// /}" ] && { echo "chain complete"; exit 0; }
  [ $(( $(date +%s) - start )) -ge "$timeout" ] && { echo "chain expired:$remaining still open"; exit 0; }
  sleep 120
done
```

On each `ticket closed: #N` event: SendMessage the leg gated on #N — its gate is open, this message is its tripwire line, verify state itself and run. Duplicate wakes are safe (relay re-checks everything), so send without wondering whether the leg's own monitor got there first. `chain complete` or `chain expired` means proceed to the report — an expired chain reports what it has. Per-agent completion notifications arrive first and they are traps: on one, do nothing — no summary, no `TaskStop`, no peeking at PRs — and go back to idle. Two exceptions, both one-message repairs that keep the chain alive without reading any transcript:

- **status `failed` / a watchdog stall** ("no progress") → SendMessage the same agent: name what killed it, tell it to check `git status` and its last commit first, then continue the leg. The resume costs the leg a few k tokens; a dead leg costs the whole chain.
- **a leg that stopped while claiming to wait** (its result says "waiting", the notification says no live background children) → SendMessage: nothing exists to wake it; re-check the awaited state now and continue, arming a real watch if one is genuinely needed. Two caveats from a live case: the harness's live-children count sees only *agent* children, so a background Bash task may still be running mute — background Bash never wakes a background session, which is usually exactly why the leg stalled; and an empty output file from such a task is pipeline buffering, not proof it never ran — the nudge should say "check your background task's process state and output before re-running anything".

Everything else: back to idle. Only the tripwire's line means proceed.

## 4. Report

Gather cheap state only: per ticket, `gh issue view <n> --json state` and the ticket's PRs by branch *prefix* — a leg may ship a follow-up PR on `issue-<n>-<suffix>` (contract clause 1), which an exact `--head issue-<n>` match misses:

```sh
gh pr list --state all --json number,state,isDraft,headRefName \
  -q '[.[] | select(.headRefName | startswith("issue-<n>"))]'
```

One screen, three lists, nothing else:

- **landed** — ticket closed, PR merged
- **held** — ticket closed, PR open (a self-landing gate held it, or no grant) — the PR comment names why
- **stalled** — ticket still open: the leg grounded (`needs input:` on the ticket) or never finished

End with the pointer that matters: leftovers, if any, are in the needs-from-you inbox — that plus this report is the whole morning read.
