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

- **Head:** one background agent, `isolation: worktree`, prompted: read `~/.claude/skills/implement-relay/SKILL.md` and follow it for `#<head>` — plus the contract.
- **Legs 2..n:** one background agent each, prompted: read `~/.claude/skills/relay/SKILL.md` and follow it for `#<prev> #<next>` — plus the contract. No worktree at launch: relay enters its own after the gate opens, branching from whatever the upstream leg actually landed.

One ticket, one agent, its own worktree and PR — never two tickets through one agent, whatever it saves.

## 3. Wait — one wake

Arm one persistent Monitor watching the **last** ticket in the chain; its close is the whole chain done. Timeout: 10800s × number of legs, floor 43200.

```bash
start=$(date +%s); timeout=<seconds>
while :; do
  s=$(gh issue view <last> --json state -q .state 2>/dev/null || echo RETRY)
  [ "$s" = CLOSED ] && { echo "chain complete: #<last> closed"; exit 0; }
  [ $(( $(date +%s) - start )) -ge "$timeout" ] && { echo "chain expired: #<last> still open"; exit 0; }
  sleep 120
done
```

Either line means proceed to the report — an expired chain reports what it has. Per-agent completion notifications arrive first and they are traps: on one, do nothing — no summary, no `TaskStop`, no peeking at PRs — and go back to idle. Two exceptions, both one-message repairs that keep the chain alive without reading any transcript:

- **status `failed` / a watchdog stall** ("no progress") → SendMessage the same agent: name what killed it, tell it to check `git status` and its last commit first, then continue the leg. The resume costs the leg a few k tokens; a dead leg costs the whole chain.
- **a leg that stopped while claiming to wait** (its result says "waiting", the notification says no live background children) → SendMessage: nothing exists to wake it; re-check the awaited state now and continue, arming a real watch if one is genuinely needed.

Everything else: back to idle. Only the tripwire's line means proceed.

## 4. Report

Gather cheap state only: per ticket, `gh issue view <n> --json state` and `gh pr list --head issue-<n> --state all --json number,state,isDraft`. One screen, three lists, nothing else:

- **landed** — ticket closed, PR merged
- **held** — ticket closed, PR open (a self-landing gate held it, or no grant) — the PR comment names why
- **stalled** — ticket still open: the leg grounded (`needs input:` on the ticket) or never finished

End with the pointer that matters: leftovers, if any, are in the needs-from-you inbox — that plus this report is the whole morning read.
