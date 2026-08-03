# Leg contract

Three clauses, appended verbatim to every launched agent's prompt — singleton, chain head, and relay legs alike. `<N>` is the agent's own ticket number.

1. Do all work on a branch named exactly `issue-<N>`, in its own worktree — create or rename to that name (`git switch -c issue-<N>`) before pushing, whatever branch your worktree started on. Your PR's head branch must be `issue-<N>` — this is how the batch tripwire and the lander find your work.
2. When the work is complete: push the branch, open a draft PR, then close ticket #<N> with a comment linking the PR. All three are part of the work, not optional courtesies — the tripwire counts your PR, and downstream chain legs gate on the ticket close.
3. After `/code-review`, add one line to the PR body: `Review: clean` if the review found nothing to hold, otherwise `Review: findings held`. The lander merges only on `Review: clean`; a missing line holds the PR.
