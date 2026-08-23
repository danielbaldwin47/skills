# Leg contract

Three clauses, appended verbatim to every launched agent's prompt — head and relay legs alike. `<N>` is the agent's own ticket number.

1. Do all work on a branch named exactly `issue-<N>`, in its own worktree — create or rename to that name (`git switch -c issue-<N>`) before pushing, whatever branch your worktree started on. Your PR's head branch must be `issue-<N>` — this is how the tripwire and downstream legs find your work. A follow-up PR fixing your own landed work keeps the prefix: head `issue-<N>-<suffix>`, never an unrelated name.
2. When the work is complete: push the branch and open a draft PR. The repo's CLAUDE.md grants **self-landing** → land the PR per the grant's gates. Then close ticket #<N> with a comment linking the PR — the close always comes last, because downstream legs gate on it. Leftovers for the human follow the repo's **needs-from-you** policy where one exists: default-and-log, file the inbox item, keep the chain moving.
3. After `/code-review`, add one line to the PR body: `Review: clean` if the review found nothing to hold, otherwise `Review: findings held`. The self-landing merge gate reads only this line; missing or not clean → the PR stays open for a human.
