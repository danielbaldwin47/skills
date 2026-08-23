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
   post the comment *before* `gh issue close`: a squash-merge's auto-close
   races the close call and can drop the comment. The close always comes
   last — downstream work gates on it.

Cost rules, each learned the expensive way:

- **End the turn while your agents run.** Background agents (reviews, advisors)
  re-invoke you when they finish; a yield/sleep loop while they work is the
  most expensive known mistake — 123 idle turns once re-billed a 300k context
  into 40M+ cache-read tokens. Resumed without a completion notification →
  read the agent's task output file rather than waiting again.
- **Targeted tests while iterating, the full gate once per push.** Single test
  files during the loop; suite + lint + typecheck once before each push. After
  fixing a review finding, re-run the reviewer's own repro before pushing —
  fix commits have twice introduced regressions the next check caught.
- **Probe the reference implementation.** When the real binary or library a
  spec describes is installed, settle behavior disputes by probing it — a
  five-call compiled probe once refuted a review finding and convicted three
  shipped bugs; re-reading docs settles nothing the docs got wrong.

- **One full-diff review per leg.** Sequence the tail: tests and CI green
  first, then `/code-review` over the whole diff, fix the findings, then
  verify only the fix commits — a delta-scoped check, stating in the PR body
  what it covered. A second full-diff pass costs more than the first and
  reverses structure the first pass approved.
- **One watch per push, then idle.** CI runs and review agents re-invoke you
  when they finish; a foreground `sleep`, a manual re-poll, or a second
  watcher on the same run only re-bills your context to learn nothing. The
  merge-verify line passing is the whole confirmation — close the ticket
  then, never after a vigil on post-merge CI.
- **Fetch each ticket or spec once**, to a file, and read the file — the
  second `gh issue view` of the same body pays its full length again.
