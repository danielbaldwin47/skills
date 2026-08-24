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

Order is the chain: chain order = close order = land order. Sequencing is the human's call, made at ticket-writing time (`/to-tickets` blocking edges, each justified by a named artifact) — dispatch never reorders, and one wrong edge outweighs every other inefficiency: a leg once idled 173 minutes on an upstream whose artifacts it never used. Serial is the design: one chain at a time. Nothing checks overlap between chains, and two self-merging chains touching the same files land semantic conflicts on the default branch overnight.

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
- **Legs 2..n:** one background agent each, prompted: read `~/.claude/skills/relay/SKILL.md` and follow it for `#<prev> #<next>` — plus the contract, plus its expiry: leg *k* gets `43200 + 10800×(k−1)` seconds, since tail legs of long chains outlive the 12h default before their gates open. No worktree at launch: relay creates its own after the gate opens, branching from whatever the upstream leg actually landed.

One ticket, one agent, its own worktree and PR — never two tickets through one agent, whatever it saves.

## 3. Wait — and pass each baton

Monitor events inside a background session are unreliable for everyone — the legs AND the dispatcher: one run's dispatcher-side Monitor over nine tickets delivered zero events. What arrives reliably is each leg's **completion notification** — that is the primary baton signal. On each one: check the chain tickets' states (`gh issue view`), and message every leg whose gate just opened. Arm the Monitor below anyway as a backup clock; when it does deliver, duplicate wakes are safe (relay re-checks everything). Timeout: 10800s × number of legs, floor 43200.

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

On a closed ticket — learned from either signal: SendMessage the leg gated on it — its gate is open, this message is its tripwire line, verify state itself and run; include the stack state (upstream PR merged, or held → branch from its head). `chain complete` or `chain expired` means proceed to the report — an expired chain reports what it has. Beyond the baton pass, a completion notification asks for nothing: no summary, no `TaskStop`, no PR reads beyond the single `gh pr view` that fills the baton's stack state — and never a message to a finished leg, which re-primes its whole context to read it (one no-op message cost a 435k-token re-read). Three exceptions, each a one-message repair that keeps the chain alive without reading any transcript:

- **status `failed` / a watchdog stall** ("no progress") → SendMessage the same agent: name what killed it, tell it to check `git status` and its last commit first, then continue the leg. The resume costs the leg a few k tokens; a dead leg costs the whole chain.
- **a leg that finished without closing its ticket** (completion notification, PR merged or ready, ticket still OPEN) → SendMessage: close #N now — the close is the baton, it is idempotent, and downstream stays gated until it lands.
- **a leg that stopped while claiming to wait** (its result says "waiting", the notification says no live background children) → SendMessage: re-check the awaited state now and continue, arming a waker that actually reaches it if one is genuinely needed. Two caveats from live cases: the harness's live-children count sees only *agent* children, so a background Bash task may still be running mute — and its wake delivery is harness-version-dependent, which is why the leg bet wrong; and an empty output file from such a task is pipeline buffering, not proof it never ran — the nudge should say "check your background task's process state and output before re-running anything".

Everything else: back to idle until the next completion notification or tripwire line.

## 3b. Land held stacks as their gates clear

A self-landing repo accumulates PRs held on the base gate — stacked on an unmerged branch. The dispatcher lands each one the moment its base reaches the default branch: base-gate holds only; a review-held PR always waits for the human. The chain otherwise manufactures held PRs by construction — after the first hold, every downstream leg is forced to stack — and follow-up tickets end up blocked on "closed" work that never landed. A landing is a batch of one or many: the moment one PR's base clears, that PR is the batch. Steps, proven across two runs of stacked landings with zero conflicts:

0. Verify the PR's review gate first: `Verdict:` comments present and a matching `Review:` line (the repo's gate). Missing or held review evidence → the PR waits for the human, whatever its base.
1. Before the batch's first merge, capture the current head of every branch stacked at or above it: `git ls-remote origin 'refs/heads/issue-*'`. After any rebase the branch names stop describing the old ancestry — the captured SHAs are the only correct rebase exclusion bases.
2. Per PR, bottom-up: `git rebase --onto origin/<default> <old-upstream-head-sha> issue-<n>` in the leg's kept worktree → `git push --force-with-lease` → `gh pr edit <n> --base <default>` → `gh pr ready <n>`.
3. CI on the rebased head: `gh pr checks <n> --watch`, then confirm the run watched is the rebased one — `gh run list --branch issue-<n> --limit 1 --json headSha` must equal the branch head; a watch launched early latches onto the previous run's green.
4. Re-run the acceptance tiers CI does not cover (the repo's evidence rule) against the rebased head.
5. `gh pr merge <n> --squash`, then verify `git merge-base --is-ancestor <mergeCommit.oid> origin/<default>`.

Worktrees and local branches are cleaned only after the run's report — a landed leg's worktree is the rebase workbench for the leg stacked on it.

## 4. Report

Gather cheap state only: per ticket, `gh issue view <n> --json state` and the ticket's PRs by branch *prefix* — a leg may ship a follow-up PR on `issue-<n>-<suffix>` (contract clause 1), which an exact `--head issue-<n>` match misses:

```sh
gh pr list --state all --json number,state,isDraft,headRefName \
  -q '[.[] | select(.headRefName | startswith("issue-<n>"))]'
```

One screen, three lists:

- **landed** — ticket closed, PR merged — link each PR's `Verdict:` comments: the quality record written at review time, not summarized after
- **held** — ticket closed, PR open (a self-landing gate held it, or no grant) — the PR comment names why
- **stalled** — ticket still open: the leg grounded (`needs input:` on the ticket) or never finished

Then two closing acts: the backlog delta — tickets the legs minted vs tickets the chain closed, so growth is visible — and one post-land verification: full suite plus the non-CI acceptance tier, once, on the merged default branch. Where the repo's needs-from-you policy assigns the inbox fold to the dispatcher, fold now. End with the pointer that matters: leftovers, if any, are in the needs-from-you inbox — that plus this report is the whole morning read.
