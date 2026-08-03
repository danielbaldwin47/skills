# `dispatch` — spec

The bedtime command. Picks tonight's tickets, groups them so their branches
don't collide, launches them, waits once, hands off to the lander.

User-invoked (`disable-model-invocation: true`). Read [README.md](README.md)
first for the cross-cutting decisions this depends on.

## Folder structure

Quality lives in disclosure, not in skill count. Each context loads only its
slice:

```
dispatch/
  SKILL.md            # thin orchestrator: the steps below, nothing bulky
  scout.md            # scout subagent prompt + table format
  leg-contract.md     # the three leg-prompt clauses (README: leg-prompt contract)
  unattended-fit.md   # the fit judgement — shared with awake by path pointer
  tripwire.md         # the monitor loop
```

The lead reads `SKILL.md`; the scout agent gets `scout.md` +
`unattended-fit.md`; leg agents get `leg-contract.md`; `awake` points at
`unattended-fit.md`. Same discipline as the runtime design, applied to the
skill text itself.

## Arguments

```
/dispatch              # default batch (config defaultBatch, 5), anything unattended-fit
/dispatch 4            # bare integer = batch size
/dispatch settings     # free text = scope filter
/dispatch settings 4   # both
/dispatch #72 #55 #56  # exactly these tickets, still collision-grouped
```

Tickets always carry `#`. That is what makes a bare integer unambiguously a
count. Batch size counts **tickets**, not parallel tracks — a chain of three
is three of the batch.

**Rationale.** Three modes were wanted, and they are three different nights:
"I don't care what gets done, just that stuff gets done"; "finish this one
feature, wherever it spans"; and "do exactly N". Named flags (`--n 4
--scope settings`) were rejected as too much typing at bedtime; free-form
intent parsing was rejected as unpredictable for a command that spends a night
of compute. The `#` prefix is the cheapest disambiguator that keeps the bare
forms short.

Scope text is matched by the scout against ticket titles and bodies —
semantically, not by substring. "settings" should find the Settings-tab ticket
and the schema ticket even where the word doesn't appear.

Explicit `#` tickets bypass selection but **not** collision grouping: naming
three tickets says which work, never that it can run in parallel.

## Flow

### 1. Preflight

Read `.claude/dispatch.json`. Missing or malformed → stop, say what's missing,
change nothing. Confirm the repo is clean and on the default branch.

### 2. Scout

Spawn **one** subagent, model `fable`. It reads the candidate tickets and the
repo tree, and returns **one compact table**. Nothing else in the run reads a
ticket body.

Per candidate ticket the table carries:

- predicted working set — the paths the ticket will touch, at
  subtree granularity (`Surfaces/Settings/**`), not exact filenames
- **unattended-fit verdict** — can a background agent actually close this?
- proposed group assignment

Then a proposed plan: parallel singletons, serial chains, and the excluded
list. If fewer fit tickets exist than the requested count, the plan proceeds
with fewer and says so. The plan is printed, and the launch follows in the
same turn — the point is walking away, but the printed plan costs nothing.

**Rationale for one scout, not one per ticket, and not inline.** Inline was
rejected outright: every ticket body would land in the lead's context, which is
the thing being protected. One-per-ticket returns N results instead of 1 for a
wall-clock saving that doesn't matter at bedtime. A single scout costs the lead
exactly one table.

**Rationale for coarse paths.** Tickets name features and reference prior PRs;
they do not name files. Any working-set prediction is inference. Subtree
granularity is the level at which inference is reliable, and it is the level
the collision rule consumes anyway.

#### Unattended-fitness

The judgement lives in `unattended-fit.md` — the tell (a precondition needing
live hardware or a real session), the seam-reachability check against the
config's `seams`, and worked examples. The scout prompt and `awake` both point
at that one file, so the two skills cannot drift apart on what counts as
awake work.

The scout drops tickets a background agent cannot close, and lists them back
as awake work — the input to [`awake`](awake.md).

The tell is in the ticket body. Live examples from forest-shell: #97 needs a
real session on a machine where Hyprland blur is known to work; #98 needs two
physical outputs; #96 needs an enrolled fingerprint. All three are labeled
`ready-for-agent`.

**Rationale.** The label does not discriminate, so filtering on labels can't
work. A `needs-human-session` label was considered and rejected: it's
per-ticket maintenance, and it wastes the read the scout is doing regardless.
Having the scout also *write* the label back was rejected as letting the
dispatcher mutate the tracker. Not filtering at all wastes a leg of the night
on a ticket that will stall at `needs input:`.

### 3. Collision rule

Two tickets collide when their predicted working sets overlap **anywhere
except a `hubFiles` entry**. No allowlist of collision-relevant paths — any
non-hub overlap is a collision.

**Rationale, and it is the load-bearing decision in this skill.** An earlier
draft had a `collisionPaths` allowlist; paths in neither list overlapped
silently, launching parallel work with unaccepted conflict risk (in
forest-shell: `Widgets/`, `assets/`). The rule now fails toward serialization
— one night slower — instead of toward a conflict that holds a PR.

Hub files are the one exception because they are registration points. Measured
over the last 12 merges: `Core/ServiceInit.qml` appears in 5,
`Core/SettingsSchema.qml` in 3, the harness scripts and `CLAUDE.md` in 4 each.
Both legs append a line; git reports a conflict; the resolution is keep-both
and takes seconds. Counting those as collisions collapses nearly every
candidate pair into one serial chain and destroys the parallelism the skill
exists to find.

The trade is explicit: hub conflicts are *accepted* at dispatch time and
*resolved* at land time. See [lander.md](lander.md).

### 4. Launch

Every leg prompt carries the three clauses of `leg-contract.md` (README:
leg-prompt contract): branch `issue-<N>`, close the ticket when the PR is up,
write the `Review:` verdict line in the PR body.

**Non-colliding ticket** → one background agent, `isolation: worktree`, told
to read `~/.claude/skills/implement/SKILL.md` and follow it for that ticket.
One ticket, one session, ~120k context.

**Colliding group** → a chain:

- head ticket runs plain implement in its own agent, exactly like a singleton
- legs 2..n each get their own agent told to read
  `~/.claude/skills/relay/SKILL.md` and follow it for `#prev #next`

Relay arms a shell-side tripwire on `#prev` closing, then branches from
`#prev`'s PR head and stacks its own PR on it. Every leg keeps its own session,
its own worktree, its own PR.

**Rationale.** The obvious cheap alternative — one agent running implement on
#A then #B in the same worktree — makes the conflict vanish entirely, because
the second leg sees the first's files. It was rejected because it doubles that
agent's context, and one-ticket-one-session at ~120k is the whole point of the
arrangement. Stacked PRs cost the lander an ordering constraint; that is the
cheaper price.

Relay is reused rather than reimplemented because its gate is already exactly
right: it waits on the upstream *ticket* closing, and clause 2 of the
leg-contract guarantees every leg closes its ticket — on any repo, not just
one whose CLAUDE.md requires it. Relay stays user-invoked; the leg's prompt
reaches it by path pointer (see README: invocation).

Chain-gate alternatives were stress-tested and rejected: gating relay on the
upstream PR *merging* instead of the ticket closing deadlocks against the
end-of-night lander (nothing merges until the tripwire fires, which waits on
the chain); launching colliding tickets in *waves* after the first merges
serializes globally and loses wall-clock to local chain serialization.

### 5. Wait — one tripwire, one wake

A single `Monitor` running a shell-side loop until every launched branch has a
PR, or the timeout expires. It emits one line. The lead wakes once.

The branch list is known before launch — `issue-<N>` per leg, dictated by the
leg-contract — so the grep is exact. Timeout is global, scaled by plan shape:
`legTimeoutBase` (config, default 2.5h) × the deepest chain length. A
five-singleton night times out at 2.5h; a night whose deepest chain is three
legs gets 7.5h. A loose timeout only delays the report — it never wastes
compute, because the lander lands whatever exists on expiry. True per-leg
tracking was rejected: it puts chain topology into shell state for no gain.

Sketch, to be sharpened against the real `gh` output during the build:

```bash
start=$(date +%s)
want=<N>
while :; do
  have=$(gh pr list --state open --json headRefName -q '.[].headRefName' \
           | grep -cFf <branch-list>)
  [ "$have" -ge "$want" ] && { echo "batch complete: $have/$want"; exit 0; }
  [ $(( $(date +%s) - start )) -ge <timeout> ] && { echo "batch expired: $have/$want"; exit 0; }
  sleep 60
done
```

**Rationale.** The alternative is reacting to harness task-completion
notifications, which re-invokes the lead once per finishing agent — N wakes,
each re-reading the whole lead context. That is precisely what `relay/SKILL.md`
was rewritten to avoid ("every model-side wake pays the whole accumulated
context again"). A third option, spawning the lander up front to do its own
waiting and land PRs as they appear, was rejected: rolling merges mean each
landing rebases a moving main, and the batch report fragments.

On `batch expired`, the lander still runs — on whatever PRs exist. A partial
night lands what it has.

### 6. Hand off

Spawn the lander (model `fable`), told to read
`~/.claude/skills/lander/SKILL.md` and follow it. Pass it the branch list
**and the chain topology** — it needs chain order for landing sequence and
cascade holds. Its one-screen report is the last thing in the lead's context.

## Lead context budget

Over a full night the lead accumulates: the plan table, one tripwire line, one
lander report. It reads no ticket body, no diff, no conflict hunk, no agent
transcript. Any change to this skill that adds a bulky read to the lead is a
regression, regardless of what it buys.

## Open for the build session

- Exact scout prompt and table format (`scout.md`). Keep it narrow — the scout
  should return a table, not prose.
- Chain ordering within a collision group — which ticket heads the chain. The
  scout proposes; smallest-first is a reasonable default heuristic.
- Model for leg agents (scout and lander are `fable`; legs default to the
  session default unless the build finds a reason otherwise).
