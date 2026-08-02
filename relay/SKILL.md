---
name: relay
description: "Watch an upstream ticket; when it closes, implement a downstream ticket, then pass the baton by closing it."
disable-model-invocation: true
---

# Relay

One leg of a ticket chain. Arguments: `<upstream> <downstream> [interval]` — two ticket references (issue number or URL) and an optional poll interval, default 30m (intervals above 60m poll at 60m — that is the wakeup ceiling).

Ticket state is the bus between sessions: your gate is the upstream ticket closing, and the next runner's gate is yours.

## 1. Wait for the baton

Check the gate immediately, then re-check every interval (ScheduleWakeup, `delaySeconds` = interval, prompt: `relay: re-check the gate`).

**Gate open** = upstream ticket closed AND (its PR merged, OR an open PR exists to stack on).

- Closed as not-planned, or closed with no PR at all: stop and report the upstream state. Never implement on a dead foundation.
- 24 polls with the gate still shut: stop and write `needs input:` with the upstream state — ticket status, last comment, PR status.

## 2. Run your leg

1. Enter a worktree.
2. Pick the base: upstream PR merged → branch from the default branch. Unmerged → branch from the upstream PR's head, and set your PR's base to that branch (stacked PR).
3. `/implement <downstream>`
4. Push the branch; open a draft PR, noting the stacked base in the body if there is one.

## 3. Pass the baton

Complete = your PR is open, tests are green, review is done. Merge is not required — merging is the human's morning job.

- Complete → close the downstream ticket with a comment linking your PR (and its stacked base). This close IS the pass: the next runner gates on it. A leg run without the pass loses the whole race.
- Blockers that need the human's decision → leave the ticket open, comment the blockers on it, end with `needs input:`.
