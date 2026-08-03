---
name: tidy
description: "Prune worktrees whose branch is already merged into the default branch; report everything else."
disable-model-invocation: true
---

# Tidy

Prune the worktrees whose branch is already in the default branch. Report everything else — the skill makes no judgement that deletes things.

## The rule

Remove a worktree iff **all three** hold:

- its branch is an ancestor of the default branch: `git merge-base --is-ancestor <branch> <default>` (default branch: `git symbolic-ref --short refs/remotes/origin/HEAD`, `main` as fallback)
- it is not locked
- it is clean — `git -C <path> status --porcelain` prints nothing

The commits of a merged branch are in the default branch, so removing its worktree loses nothing — that is what makes this safe unattended at 3am. Uncommitted changes are the one thing in a merged-branch worktree that is *not* merged, which is why clean is a condition and `git worktree remove` is never run with `--force`.

## Procedure

1. `git worktree list --porcelain` — the authoritative inventory (config `worktreeDir` only says where to expect them). Skip the main checkout.
2. Apply the rule to each; remove the qualifiers with `git worktree remove <path>`.
3. `git worktree prune` — clears stale administrative files.
4. Cross-reference the kept unmerged `issue-<N>` branches against their tickets (`gh issue view <N> --json state`). **Unmerged branch + closed ticket is the ambiguous class**: the work either landed under another branch name or was abandoned, the difference matters, and only a human knows which — surface it, never delete it. This class silently accumulates, which is why it gets named explicitly. Unmerged branches naming no ticket get their own report line.

Delete no branch refs — pruning worktrees is the whole job.

## Report

Short list, grouped:

- **pruned** — worktree and branch, one line each
- **kept: locked** — with branch name, so a stale lock is visible
- **kept: unmerged, ticket open** — in flight, expected
- **kept: unmerged, ticket closed** — the ambiguous class, flagged as such
- **kept: unmerged, no ticket** — branch names no ticket; judge by hand
- **kept: dirty** — uncommitted changes present
