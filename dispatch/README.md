# dispatch

The bedtime command. `/dispatch` picks a batch of open tickets that a background agent can genuinely close, groups them so their branches won't merge-conflict, launches one overnight agent per ticket — each in its own worktree, on its own branch, opening its own PR — waits for the batch on a single shell-side tripwire, then hands the finished PRs to the [lander](../lander/README.md) to rebase, test, and merge. You invoke it, read the plan it prints, and walk away. In the morning: a merged default branch plus a short list of what actually needed a person.

## Setup

Dispatch is repo-agnostic; every repo-specific fact comes from `.claude/dispatch.json` at the root of the repo you run it in. No config, no dispatch — the skill stops and says what's missing rather than guessing. A dispatcher that guesses wrong spends a night of compute on the wrong work; one that stops costs a minute of setup.

Requirements:

- Tickets tracked as GitHub issues, with the `gh` CLI authenticated.
- A `.claude/dispatch.json` at the repo root:

```json
{
  "ticketLabel": "ready-for-agent",
  "testCommand": "tests/run.sh",
  "hubFiles": [
    "Core/ServiceInit.qml",
    "Core/SettingsSchema.qml",
    "CLAUDE.md",
    "tools/*.sh"
  ],
  "worktreeDir": ".claude/worktrees",
  "defaultBatch": 5,
  "mergeMethod": "merge",
  "legTimeoutBase": "2.5h",
  "seams": [
    { "name": "tests", "command": "tests/run.sh", "covers": "decisions: policy, formatting, parsing, merging, migration, thresholds" },
    { "name": "capture", "command": "tools/capture-harness.sh", "covers": "client-side pictures: layout, colour, opacity compositing" }
  ]
}
```

| Field | Required | Meaning |
|---|---|---|
| `ticketLabel` | yes | Issue label that marks candidate tickets. |
| `testCommand` | yes | The command that must be green before anything merges. |
| `hubFiles` | yes | Registration-point paths (globs allowed) where parallel work is *expected* to conflict trivially — both sides append a line, resolution is keep-both. Overlap here doesn't count as a collision. Can be `[]`. |
| `defaultBatch` | yes | Batch size when `/dispatch` is run bare. Counts tickets, not parallel tracks. |
| `legTimeoutBase` | yes | Time budget for one leg (e.g. `"2.5h"`). The batch timeout is this × the deepest chain length. |
| `legModel` | no | Model for implementation legs (default `opus`). Every leg launches with it set explicitly rather than inheriting the lead's model. |
| `maxStackDepth` | no | Longest allowed chain (default `3`). A longer chain is cut at the cap; the tail tickets drop from the batch and are named in the plan. |
| `worktreeDir` | no | Where worktrees are expected to live. `git worktree list` remains authoritative. |
| `mergeMethod` | no | `merge` (default), `squash`, or `rebase` — passed to the lander's `gh pr merge`. |
| `seams` | no | Declared verification harnesses (`name`, `command`, `covers`). Used by the scout and by [awake](../awake/README.md) to judge whether a ticket's acceptance criteria are observable without a human. Omitted → fit is judged by ticket text alone. |

## Usage

```
/dispatch              # default batch, anything unattended-fit
/dispatch 4            # bare integer = batch size
/dispatch settings     # free text = scope filter, matched semantically by the scout
/dispatch settings 4   # both
/dispatch #72 #55 #56  # exactly these tickets
```

Tickets always carry `#` — that's what makes a bare integer unambiguously a count. An explicit `#` list bypasses selection but **not** fit judgement or collision grouping: naming a ticket says which work, not that a background agent can close it, and never that it can run in parallel.

## How a night runs

1. **Preflight** — load the config, check every seam runner present in the repo appears in some seam's `command` (a missing seam makes the scout silently bounce every ticket only that seam can verify), confirm a clean tree on the default branch.
2. **Scout** — one subagent reads every candidate's body and returns one table: predicted working set (subtree granularity), unattended-fit verdict, group. Nothing else in the run ever reads a ticket body.
3. **Check the plan** — no two parallel tickets may share a working-set path outside `hubFiles`. Overlap → chain them.
4. **Launch** — print the plan and launch in the same turn. Singletons and chain heads run [implement](../implement/README.md); chain legs 2..n run [relay](../relay/README.md), which waits for the upstream ticket to close and stacks its PR on the upstream branch. Every agent's prompt ends with the three clauses of [leg-contract.md](leg-contract.md), and every leg launches with `model: <legModel>` set explicitly. Then the run record is written — `~/.claude/dispatch-runs/<UTC-timestamp>.json`, holding the plan table, chains, branch list, and per-leg model — and updated at each later step; it is the only artifact that survives an interrupted run, and the morning `/lander` reads it to know what the batch intended.
5. **Wait** — one persistent shell loop ([tripwire.md](tripwire.md)) greps `gh pr list` for the expected `issue-<N>` branches (forgiving the known `worktree-issue-<N>-slug` drift). It emits one line — `batch complete` or `batch expired` — and either way the run proceeds: a partial night lands what it has.
6. **Hand off** — spawn the lander with the branch list and chain topology; relay its one-screen report verbatim.

## Design notes

**The lead's context is a hard budget.** Over a whole night the orchestrating session accumulates exactly: one plan table, one tripwire line, one lander report. It reads no ticket bodies, no diffs, no conflict hunks, no agent transcripts — every bulky read belongs to a subagent that returns a summary. This is why the scout is one subagent rather than inline, why waiting is a shell loop rather than reacting to per-agent notifications (N wakes, each re-paying the whole accumulated context), and why landing is delegated entirely.

**The collision rule fails toward serialization.** Two tickets collide when their predicted working sets overlap anywhere except a hub file — no allowlist of "collision-relevant" paths. An earlier design had one, and paths in neither list overlapped silently, launching parallel work with unaccepted conflict risk. The current rule's failure mode is one night slower, never a conflict that holds a PR. Hub files are the sole exception because counting them collapses nearly every pair into one serial chain and destroys the parallelism the skill exists to find; the trade is explicit — hub conflicts are accepted at dispatch time and resolved keep-both at land time.

**One ticket, one agent, ~120k context.** Colliding tickets are never collapsed into a single agent running both back to back, even though that would trivially avoid the conflict — it would double that agent's context. Chains cost the lander an ordering constraint instead; that's the cheaper price.

**The folder is disclosure-shaped.** Each file loads into exactly the context that needs it:

| File | Read by |
|---|---|
| `SKILL.md` | the lead |
| [scout.md](scout.md) | the scout subagent |
| [leg-contract.md](leg-contract.md) | every launched agent (appended to its prompt) |
| [unattended-fit.md](unattended-fit.md) | the scout and `awake` — one file, so they can't drift apart |
| [tripwire.md](tripwire.md) | the lead, at step 5 only |
