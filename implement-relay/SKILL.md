---
name: implement-relay
description: "Implement a piece of work based on a spec or set of tickets."
disable-model-invocation: true
---

Implement the work described by the user in the spec or tickets.

Use /tdd where possible, at pre-agreed seams.

Run typechecking regularly, single test files regularly, and the full test suite once at the end.

Once done: commit, push the branch, and open a draft PR — reviewers post their verdicts as PR comments, so the PR must exist before `/code-review` runs.

When the work comes from a ticket `#N`, finishing means shipping — these four
are part of the work, not courtesies (they restate the dispatch leg contract;
that contract remains the authority when both are in play):

1. Branch named exactly `issue-<N>` — rename before pushing if the worktree
   started on another name.
2. Push the branch and open a draft PR; mark it ready (`gh pr ready`) once
   review is done — a held PR left in draft is invisible.
3. Run the review tail on the open PR: `/code-review` over the whole diff,
   each reviewer posting its verdict as its own PR comment (first line
   `Verdict: clean` / `Verdict: findings-<n>` / `Verdict: held`, one line
   per finding). No findings → body line `Review: clean`. Findings → fix in
   one batch, one scoped delta reviewer over the fix commits' diff (its own
   `Verdict:` comment), body line `Review: fixed-<n>`; delta unclean twice →
   `Review: findings held`. The merge gate cross-checks the line against
   the comments.
4. When the repo's CLAUDE.md grants **self-landing**, run the repo's
   needs-from-you leftover scan *before* merging — the scan reads the work
   with fresh eyes and once surfaced a shipped defect only after the merge,
   forcing a second PR to fix the first. Then land the PR per the grant's
   gates, and close ticket #N with a comment linking the PR — post the
   comment *before* `gh issue close`: a squash-merge's auto-close races the
   close call and can drop the comment. The close always comes last —
   downstream work gates on it. A follow-up PR fixing your own landed work
   keeps the ticket's branch prefix: head `issue-<N>-<suffix>`, never an
   unrelated name.

Cost rules, each learned the expensive way:

- **Delegate orientation reading.** Pre-edit recon — whole-file surveys,
  `cat` loops, prototype/corpus ingestion — goes to a read-only subagent
  that returns a summary: `cavecrew-investigator` for where-is-what maps
  (compressed output, cheapest), `Explore` for broader questions; your own
  context takes targeted-range reads of the files you will actually edit.
  Ask the investigator to return `file:line` ranges and read only those —
  paying for a map and then re-reading the mapped files whole pays twice,
  and eight of nine audited legs did exactly that (one read the same file
  seven separate times; inline recon ran 28–90k tokens per leg).
- **End a turn only with a live waker.** The turn-end check, every time:
  name what will re-invoke you. A background *agent* or a Monitor qualifies;
  a background *Bash* task is a bet — its wake delivery has flipped between
  harness versions, and sessions have both hung on it and been woken by it.
  Bounded suites run foreground with an explicit `timeout` above the suite's
  real runtime: the 120s default backgrounds a long suite mid-command and
  manufactures exactly this trap — it caught two legs in one audited run.
  An empty output file is pipeline buffering, not a dead process — check
  process state before declaring a run never happened and re-running it.
- **Fix with Edit, not heredoc patch scripts.** Inline `python3 - <<'PY'`
  patchers pay the patch text twice (output, then echoed result) and hide
  the diff; one fix cycle ran ~15 of them for work Edit does natively.
- **Filter bulky output at the source.** `pytest -q | tail`, listings
  through `head`/`grep`, quiet lint flags — one unfiltered file listing
  cost 12k tokens, and an early dump re-bills every turn until the harness
  clears it.
- **Revert-verify each new regression test**: revert the fix, confirm the
  test fails, restore. Cost ~5k for a whole set; it caught a test that
  passed with its fix reverted — a test proving nothing. State the result
  in the PR body (`revert-verified: <tests>`) — the delta re-reviewer
  checks the claim, which is what makes skipping it visible.
- **End the turn while your agents run.** Background agents (reviews, advisors)
  re-invoke you when they finish; a yield/sleep loop while they work is the
  most expensive known mistake — 123 idle turns once re-billed a 300k context
  into 40M+ cache-read tokens. Resumed without a completion notification →
  read the agent's task output file rather than waiting again.
- **Targeted tests while iterating, the full gate once per push.** Single test
  files during the loop; suite + lint + typecheck once before each push. Batch
  a round of review fixes before re-running the gate — one session re-ran the
  full suite after each of seven small fixes, re-ingesting the suite output
  seven times. After fixing a review finding, re-run the reviewer's own repro
  before pushing — fix commits have twice introduced regressions the next
  check caught.
- **Probe the reference implementation.** When the real binary or library a
  spec describes is installed, settle behavior disputes by probing it — a
  five-call compiled probe once refuted a review finding and convicted three
  shipped bugs; re-reading docs settles nothing the docs got wrong.

- **One full-diff review, one delta re-review — never a second full pass.**
  Why the delta scope exists: fix batches have twice introduced regressions,
  and nine audited legs shipped 10–30-edit fix batches (data-loss repairs
  among them) with no second look. A second *full*-diff pass costs more than
  the first and reverses structure the first pass approved. The fix batch is
  a push point like any other — full gate after it.
- **One watch per push, then idle.** CI runs and review agents re-invoke you
  when they finish; a foreground `sleep`, a manual re-poll, or a second
  watcher on the same run only re-bills your context to learn nothing. The
  merge-verify line passing is the whole confirmation — close the ticket
  then, never after a vigil on post-merge CI.
- **Fetch each ticket or spec once**, to a file, and read the file — the
  second `gh issue view` of the same body pays its full length again.
- **Follow-up tickets carry `needs-triage`.** Promotion to agent-ready is
  the human's act — it is the approval self-landing rests on, and one run
  shipped seven self-approved tickets before this rule existed. Search open
  issues for a duplicate before filing; and a question a ≤10-call probe
  against the installed reference would settle is work, not a ticket — run
  the probe.
- **Cite acceptance evidence CI cannot see.** When a ticket's proof lives
  in a tier CI does not run, the PR body names the local run: command, pass
  count, HEAD SHA. Any rebase makes the citation stale — re-run before
  landing.
