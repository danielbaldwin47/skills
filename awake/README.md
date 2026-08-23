# awake

> **Superseded** in repos carrying a needs-from-you policy: work needing eyes, hands, or hardware reaches the human as an inbox item (a `ready-for-human` ticket files one), so no separate backlog walk exists to run. Kept for repos without that policy.

The daytime counterpart to [dispatch](../dispatch/README.md). Dispatch takes the sleeping work; `/awake` walks you through settling the tickets no background agent can close — the ones that need eyes on a screen, hands on hardware, or a live session: a compositor effect that must be seen, a second monitor that must be attached, a fingerprint that must be enrolled. Run `/awake #97` to walk one ticket, or bare `/awake` to derive the backlog and pick.

## Why it exists

Some tickets need a person — and those are exactly the ones that sit open longest. Without this skill they are a growing pile with nothing attached to them.

They are also the ones where doing it wrong is easy and invisible. The founding example: a bar-blur ticket once got checked on a machine where blur rendered for *no* window at all — "no change" was read as "feature doesn't work", and the run proved nothing. The trap in live verification is a missing precondition, not a missing observation, which is why the walk's load-bearing step is: **state the preconditions and check the falsifiable ones first.** Confirm blur renders on an ordinary window before judging the bar; confirm the second output is attached; confirm the finger is enrolled. A negative result on an unverified precondition reads exactly like a real failure.

## How the backlog is derived

Re-derived on every run, never maintained as a list: open tickets judged against [unattended-fit.md](unattended-fit.md). Labels don't discriminate (awake tickets often carry the same ready-for-agent label as fit ones); only the ticket body tells. And the first check runs the other way too: a ticket that *is* reachable from a seam declared in `.claude/dispatch.json` isn't awake work at all — a harness covers it, and it belongs to dispatch.

## The walk

Read the ticket, verify preconditions falsifiable-first, set up, then direct one observation at a time — always toggle-and-compare, never a judgement of an absolute. Finally, **record the verdict on the ticket**: what was checked, on what hardware, with what result. Settled → close; partial → a comment naming the hardware still unknown, ticket stays open. The recorded conditions are what make a negative result interpretable later — an unrecorded pass is a pass that gets re-run.
