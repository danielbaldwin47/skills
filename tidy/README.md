# tidy

The housekeeper. `/tidy` removes every worktree whose branch is already merged into the default branch — and reports everything else, grouped, without touching it. Reached by the [lander](../lander/README.md) after a batch lands (the merges just made more worktrees prunable), or by hand any time worktrees have piled up.

## The rule

A worktree is removed if and only if **all three** hold:

1. its branch is an ancestor of the default branch,
2. it is not locked,
3. it is clean — no uncommitted changes.

The commits of a merged branch are already in the default branch, so removing its worktree loses nothing — that's what makes this safe unattended at 3am. The clean condition exists because uncommitted changes are the one thing in a merged-branch worktree that is *not* merged; an earlier draft omitted it and would have deleted them. It also keeps the skill permanently off `git worktree remove --force`. Locked is treated as an explicit human "leave this". Every other case involves a judgement, and tidy makes no judgement that deletes things — it deletes no branch refs either; pruning worktrees is the whole job.

## The ambiguous class

One kept group is worth naming: **unmerged branch whose ticket is already closed.** Either the work landed under another branch name, or it was abandoned — both are plausible, the difference matters, and only a human knows which. Tidy surfaces the question rather than answering it. This class gets an explicit callout because it's the one that silently accumulates.

## Why a skill and not a shell script

A script does the merged-ancestor check perfectly well. It can't do the ambiguous class — that needs ticket state cross-referenced against branch state, and a judgement about what to say. The deterministic part is still plain `git` inside the skill; the model's job is the cross-reference and the report:

- **pruned** — worktree and branch, one line each
- **kept: locked** — with the branch name, so a stale lock is visible
- **kept: unmerged, ticket open** — in flight, expected
- **kept: unmerged, ticket closed** — the ambiguous class
- **kept: unmerged, no ticket** — judge by hand
- **kept: dirty** — uncommitted changes present
