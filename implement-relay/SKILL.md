---
name: implement-relay
description: "Implement a piece of work based on a spec or set of tickets."
disable-model-invocation: true
---

Implement the work described by the user in the spec or tickets.

Use /tdd where possible, at pre-agreed seams.

Run typechecking regularly, single test files regularly, and the full test suite once at the end.

Once done, use /code-review to review the work.

Commit your work to the current branch.

When the work comes from a ticket `#N`, finishing means shipping — these four
are part of the work, not courtesies (they restate the dispatch leg contract;
that contract remains the authority when both are in play):

1. Branch named exactly `issue-<N>` — rename before pushing if the worktree
   started on another name.
2. Push the branch and open a draft PR.
3. Add one line to the PR body after /code-review: `Review: clean` if nothing
   was held, else `Review: findings held`.
4. When the repo's CLAUDE.md grants **self-landing**, land the PR per the
   grant's gates first. Then close ticket #N with a comment linking the PR —
   downstream work gates on the close, so the close always comes last.
