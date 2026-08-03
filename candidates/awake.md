# `awake` — spec

Guided live-session verification for the tickets no background agent can close.
The counterpart to [`dispatch`](dispatch.md): dispatch takes the sleeping work,
awake takes what's left.

User-invoked. Read [README.md](README.md) first.

## Arguments

```
/awake #97    # walk this ticket's verification
/awake        # list the awake backlog and pick one
```

The backlog is re-derived each time — open tickets read against the fit
judgement in `dispatch/unattended-fit.md`, the same file dispatch's scout
uses, so the two skills cannot disagree about what counts as awake work. No
list is maintained anywhere; with a backlog of three, re-deriving is cheaper
than keeping state fresh.

## Why it exists

Dispatch's scout excludes unattended-unfit tickets every night and hands
them back as a list. Without this skill that list is just a growing pile with
nothing attached to it — and those tickets are exactly the ones that have been
sitting open longest, because they're the ones that need a person.

They are also the ones where doing it wrong is easy and invisible. forest-shell
#78 checked the bar's blur on a machine where blur renders for no window at
all, saw no change, and proved nothing — the ticket says so in as many words.
#97 exists only because that happened. A guided pass is worth having precisely
because the trap is a missing precondition, not a missing observation.

## Flow

1. **Read the ticket.** Extract what would settle it. Ticket bodies in this
   repo state this explicitly — #97 has a "What would settle it" section
   naming the machine requirement, the wallpaper, and the comparison.
2. **State the preconditions, and check the falsifiable ones first.** This is
   the step that earns the skill. For #97: confirm Hyprland blur renders on an
   ordinary window *before* looking at the bar, so that "no blur" cannot be
   read as "no blur support" a second time. For #98: two outputs actually
   connected. For #96: a finger actually enrolled.
3. **Set up.** Wallpaper, config, harness, whatever the ticket needs. Use the
   repo's existing tools where they apply — forest-shell has
   `tools/blur-harness.sh`, `tools/nested-session.sh`, `tools/capture-harness.sh`,
   `assets/noise.png`.
4. **Tell the human exactly what to look at**, in the order that makes the
   comparison decisive. One thing at a time; toggle and compare rather than
   judge an absolute. A ticket with several independent observations (#96 has
   three) is walked the same way — observation by observation; no separate
   checklist mode needed.
5. **Record the verdict on the ticket** — what was checked, on what hardware,
   with what result. Close it if settled. A partial verdict — checked on one
   machine, unknown on another — is a comment naming the hardware, and the
   ticket stays open.

Step 5 matters as much as the rest: the reason #97 had to be re-filed is that
the first attempt's *conditions* weren't written down, so its negative result
couldn't be interpreted later.

## What it is not

Not a test runner. If a seam covers the thing, it isn't awake work and dispatch
should have taken it. Awake is for what forest-shell's `CLAUDE.md` calls out as
uncovered: compositor composition — blur behind the bar, layer stacking, frame
pacing — which needs presents the nested compositor cannot make and pixels a
client-side grab never sees.

## The seam question

Part of the shared fit judgement (`dispatch/unattended-fit.md`): before
setting anything up, check whether the ticket is actually reachable from a
seam declared in `.claude/dispatch.json` — and if it is, say so and stop,
rather than running a manual pass for something a harness already covers.

This is the lightweight version of the deferred `seam` skill (see README).
If it turns out to want real depth, that's the signal to build `seam` properly
and have both dispatch's scout and awake call it.

## Current backlog

On 2026-08-02, three tickets: #96 (lock failure path — shake, PAM message,
fingerprint-when-enrolled), #97 (bar blur, needs a machine where blur works),
#98 (multi-monitor, needs two outputs). All three are labeled
`ready-for-agent`, which is exactly why the label can't be used to route them.

## Open for the build session

None of substance — the 2026-08-03 grilling settled backlog derivation
(re-derive), partial verdicts (comment naming hardware, ticket stays open),
and multi-observation tickets (walked in step 4; no separate mode).
