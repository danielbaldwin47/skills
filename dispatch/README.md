# dispatch

`/dispatch #12 #15 #9` (or `/dispatch #12-#16` for a range) launches a relay chain over the named tickets: the first runs [implement-relay](../implement-relay/README.md) directly, each later ticket runs [relay](../relay/README.md) gated on its predecessor's close — exactly what running `/relay <prev> <next>` once per pair builds by hand, launched in one turn. Each leg is its own background agent, worktree, `issue-<N>` branch, and PR, and every prompt carries the three clauses of [leg-contract.md](leg-contract.md).

In a repo whose CLAUDE.md grants **self-landing**, legs merge their own PRs behind the grant's gates and route human leftovers per the repo's **needs-from-you** policy. The morning read is two things: the inbox, and dispatch's closing report (landed / held / stalled). Without the grant the chain still runs — PRs stay open for the human, tickets still close, downstream legs still fire.

## Setup

None. Tickets as GitHub issues, `gh` authenticated — that's the whole requirement. Repo-specific behavior (landing authority, leftover routing) comes from the repo's CLAUDE.md, so dispatch itself stays repo-agnostic. `.claude/dispatch.json` is no longer read.

## Usage

```
/dispatch #12 #15 #9   # chain in exactly this order
/dispatch #12-#16      # range, ascending (also #12-16)
```

Order is the chain: chain order = close order = land order. Sequencing and parallelism are the human's calls, made at ticket-writing time — `/to-tickets` emits tickets with blocking edges, and those edges are the chain. Parallel tracks over non-overlapping work are separate `/dispatch` invocations. An explicit ticket in the wrong state stops the run; a closed ticket inside a range is dropped as already done.

## How a run goes

1. **Preflight** — one `gh issue view` per ticket confirms it exists and is open; the repo's CLAUDE.md is checked for the self-landing grant.
2. **Launch** — plan printed and all agents launched in the same turn. The head gets a worktree at launch; relay legs make their own after their gate opens, branching from what upstream actually landed.
3. **Wait** — one persistent shell loop watches the last ticket in the chain; its close is the whole chain done. One wake, however long the night.
4. **Report** — landed / held / stalled per ticket, plus the pointer to the needs-from-you inbox.

## Design notes

**The lead's context is a hard budget.** Over a whole run the orchestrating session accumulates one plan, one tripwire line, one report — no ticket bodies, no diffs, no agent transcripts. Waiting is shell-side because reacting to per-agent notifications pays the whole accumulated context N times.

**No scout, no collision grouping.** The old dispatch predicted working sets and serialized colliding tickets itself. The new baseline moves that judgement to where the information actually is: the human (and `/to-tickets`) decide order and parallelism when the tickets are written. Dispatch executes the order it is given — one chain per invocation, serial by construction, so collisions inside a run are impossible.

**No lander, no awake.** Self-landing makes every leg its own lander — merge gates travel with the leg, and a gate-held PR waits for a human with a comment naming the gate. The needs-from-you inbox absorbs what awake existed to surface: work needing eyes, hands, or hardware becomes an inbox item (or a `ready-for-human` ticket, which files one), never a separate skill's backlog. Both skills remain in the folder for repos without the grant, marked superseded.

**One ticket, one agent.** Colliding work is chained, never collapsed into one agent running two tickets back to back — doubling an agent's context is the more expensive price.

**Ticket size bounds leg context.** A leg pays for everything its ticket bundles: one package or subsystem per ticket is the size that lands near the ~120k context target. The 404k outlier leg traced straight to a ticket carrying three packages and a 33k spec — the fix was at ticket-writing time, not in the leg. A later 7-leg run confirmed it at scale: every ticket's bare implement core (reading + edits + tests, before any relay or review machinery) ran 1.4–3.5x the 120k target, making ticket sizing the primary overrun cause across the whole run — workflow overhead and waste were secondary. Audit legs against that decomposition before blaming the workflow.

| File | Read by |
|---|---|
| `SKILL.md` | the lead |
| [leg-contract.md](leg-contract.md) | every launched agent (appended to its prompt) |
