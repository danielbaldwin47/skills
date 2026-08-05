# relay

One leg of a ticket chain. `/relay #A #B` watches upstream ticket `#A`; when it closes, relay implements downstream ticket `#B` — branching from the upstream PR's head if it hasn't merged yet (a stacked PR) or from the default branch if it has — then passes the baton by closing `#B` with a link to its PR. Ticket state is the bus between sessions: relay's gate is the upstream ticket closing, and the next runner's gate is relay's own close.

## How the waiting works

Shell-side, not model-side. A persistent monitor loop polls `gh issue view` and emits exactly one line — `gate open`, `gate dead` (upstream closed as not-planned), or `gate expired` (wall-clock: `legTimeoutBase` × 2 when the repo carries `.claude/dispatch.json`, 12h otherwise) — and the session wakes exactly once, on that line. Polling from the session itself would pay the whole accumulated context on every wake just to find the gate still shut.

## Preflight, and the one forbidden move

Before arming anything, relay verifies the downstream ticket exists and is open. Anything else grounds the leg: stop, report both tickets' states, leave the fix to the human. A grounded leg stays grounded — **substituting another ticket is the one forbidden move.** The arguments name the one leg this session runs; a relay that improvises a different ticket spends its budget on work nobody sequenced.

## Where it fits in the suite

[Dispatch](../dispatch/README.md) uses relay for chain legs 2..n: tickets whose working sets collide are serialized into a chain, the head runs [implement](../implement/README.md) directly, and each subsequent leg is a relay gated on its predecessor's ticket. The [leg contract](../dispatch/leg-contract.md) makes the chain repo-independent — clause 2 guarantees every leg closes its ticket, which is exactly the event relay gates on. Relay also works hand-wired, outside any dispatch night; it follows the same contract (branch `issue-<N>`, draft PR, `Review:` line) so a later `/lander` can find and land its PR either way. Merging is not relay's job — landing belongs downstream: the lander's on dispatch nights, the human's otherwise.
