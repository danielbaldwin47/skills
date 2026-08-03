---
name: awake
description: "Guided live-session verification for the tickets no background agent can close."
disable-model-invocation: true
---

# Awake

Walk a human through settling a ticket that needs eyes, hands, or hardware.

## Arguments

- `/awake #97` — walk this ticket's verification
- `/awake` — derive the backlog and pick one: open tickets (`gh issue list --state open --limit 100`, filtered to `ticketLabel` when `.claude/dispatch.json` declares one) judged against `~/.claude/skills/dispatch/unattended-fit.md` (read it — it is the same file dispatch's scout uses, so the two skills cannot disagree about what counts as awake work). The backlog is re-derived every time; no list is maintained anywhere.

First check either way: if the ticket is actually reachable from a seam declared in `.claude/dispatch.json`, say so and stop — a harness already covers it, and it belongs to dispatch, not a manual pass. No config, or no seams declared → skip this check.

## Walk

1. **Read the ticket.** Extract what would settle it — well-formed tickets state this outright.
2. **State the preconditions, and check the falsifiable ones first.** This is the step that earns the skill: confirm the effect renders on an ordinary window *before* judging the feature, the second output is attached, the fingerprint is enrolled. A negative result on an unverified precondition proves nothing — and reads exactly like a real failure.
3. **Set up** — wallpaper, config, harness, whatever the ticket needs, using the repo's existing tools where they apply.
4. **Tell the human exactly what to look at**, one observation at a time, ordered so each comparison is decisive. Toggle and compare; never ask for a judgement of an absolute. A ticket with several independent observations is walked the same way, observation by observation.
5. **Record the verdict on the ticket**: what was checked, on what hardware, with what result. Settled → close it. Partial — checked on one machine, unknown on another — → comment naming the hardware, ticket stays open. The conditions are the part that makes a negative result interpretable later; an unrecorded pass is a pass that gets re-run.
