# `tidy` — spec

Prunes worktrees whose branch is already in main. Reports everything else.

User-invoked frontmatter (`disable-model-invocation: true`); reached by
[`lander`](lander.md) via path pointer, and `/tidy` by hand.
Read [README.md](README.md) first.

## The rule

Remove a worktree if and only if **all three**:

- its branch is an ancestor of main (`git merge-base --is-ancestor <branch> main`)
- the worktree is not locked
- the worktree is clean — no uncommitted changes

Everything else is reported and left alone. That includes: locked worktrees,
dirty worktrees even on a merged branch, and branches that are not in main
regardless of what their ticket says. `git worktree prune` runs afterwards to
clear stale administrative files.

**Rationale.** The commits in a merged branch are in main, so removing its
worktree loses nothing — that is what makes this safe to do unattended at 3am.
The clean condition exists because uncommitted changes are the one thing in a
merged-branch worktree that is *not* in main; an earlier draft omitted it and
would have deleted them. (`git worktree remove` refuses dirty worktrees
without `--force`, so the condition also keeps the skill off that flag.)
Every other case involves a judgement, and the skill does not make judgements
that delete things. Deleting the merged branch ref as well was offered and not
taken; pruning the worktree is the part that reclaims disk and reduces the
noise in `git worktree list`.

Locked is treated as an explicit human "leave this". On 2026-08-02 the locked
worktrees were the two with live PRs (#115, #116).

The worktree directory comes from config (`worktreeDir`), but
`git worktree list` is authoritative and wins where they disagree.

## The ambiguous class

Report, never delete: **unmerged branch whose ticket is already closed.**

On 2026-08-02 that was four worktrees — `issue-43`, `issue-44`, `issue-45`,
`issue-46`. Their tickets are closed and absent from the open list, but the
branches are not ancestors of main. That means either the work landed under
another branch name, or it was abandoned. Both are plausible and the difference
matters, so the skill surfaces the question rather than answering it.

This class is worth naming explicitly in the report, because it is the one that
silently accumulates.

## Output

A short list, grouped:

- **pruned** — worktree and branch, one line each
- **kept: locked** — with the branch name, so you can see if a lock is stale
- **kept: unmerged** — split into "ticket open" (in flight) and "ticket closed"
  (the ambiguous class)
- **kept: dirty** — uncommitted changes present

## Why it is a skill and not a shell script

A script can do the merged-ancestor check perfectly well and for free. It
cannot do the ambiguous class — that needs the ticket state cross-referenced
against branch state, and a judgement about what to say. The deterministic part
of tidy should still *be* shell inside the skill; the model's job is the
reporting and the cross-reference, not the `git` invocations.

## Why standalone as well as automatic

Cleanup is a step the lander needs — it has just made five more worktrees
prunable. But the 7 dead worktrees sitting in forest-shell today predate any
dispatch run, and there should be a way to clear them without launching a
night's work to do it.

## Expected first run

Against forest-shell as of 2026-08-02: 7 pruned (`worktree-issue-37/38/39/41/71`,
`worktree-relay-claude-md-line`, `research/claude-cli-contract`), 10 kept —
2 locked, 4 ambiguous, and the rest in flight.

## Open for the build session

- Whether to offer branch deletion behind an explicit user-invoked flag. Never
  automatic.
