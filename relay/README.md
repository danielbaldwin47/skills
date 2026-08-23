# relay

One leg of a ticket chain. `/relay #A #B` watches upstream ticket `#A`; when it closes, relay implements downstream ticket `#B` — branching from the upstream PR's head if it hasn't merged yet (a stacked PR) or from the default branch if it has — then passes the baton by closing `#B` with a link to its PR. Ticket state is the bus between sessions: relay's gate is the upstream ticket closing, and the next runner's gate is relay's own close.

## How the waiting works

Shell-side, not model-side. A persistent monitor loop polls `gh issue view` and emits exactly one line — `gate open`, `gate dead` (upstream closed as not-planned), or `gate expired` (wall-clock, 12h by default) — and the session wakes exactly once, on that line. Polling from the session itself would pay the whole accumulated context on every wake just to find the gate still shut.

## Preflight, and the one forbidden move

Before arming anything, relay verifies the downstream ticket exists and is open. Anything else grounds the leg: stop, report both tickets' states, leave the fix to the human. A grounded leg stays grounded — **substituting another ticket is the one forbidden move.** The arguments name the one leg this session runs; a relay that improvises a different ticket spends its budget on work nobody sequenced.

## Where it fits in the suite

[Dispatch](../dispatch/README.md) is the batch form: `/dispatch #a #b #c` launches the head on [implement-relay](../implement-relay/README.md) and each subsequent ticket as a relay gated on its predecessor — one turn instead of one `/relay` per pair. The [leg contract](../dispatch/leg-contract.md) makes the chain repo-independent — clause 2 guarantees every leg closes its ticket, which is exactly the event relay gates on. Relay also works hand-wired; it follows the same contract (branch `issue-<N>`, draft PR, `Review:` line) either way. Merging depends on the repo: a CLAUDE.md **self-landing** grant means the leg lands its own PR behind the grant's gates before closing the ticket; without a grant, the PR stays open for the human.
