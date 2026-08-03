# Leg contract

Three clauses, appended verbatim to every launched agent's prompt — singleton, chain head, and relay legs alike. `<N>` is the agent's own ticket number.

1. Do all work on a branch named exactly `issue-<N>`, in its own worktree. Your PR's head branch must be `issue-<N>` — this is how the batch tripwire and the lander find your work.
2. When your PR is open and the work is complete, close ticket #<N> with a comment linking the PR. Closing the ticket is part of the work, not an optional courtesy — downstream chain legs gate on it.
3. After `/code-review`, add one line to the PR body: `Review: clean` if the review found nothing to hold, otherwise `Review: findings held`. The lander merges only on `Review: clean`; a missing line holds the PR.
