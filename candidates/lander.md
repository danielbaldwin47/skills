# `lander` — spec

Takes the night's PRs and lands the ones that deserve it. Runs when the batch
finishes, typically at 3am with nobody watching.

User-invoked frontmatter (`disable-model-invocation: true`); reached two ways:
spawned by [`dispatch`](dispatch.md) via path pointer (model `fable`), and
`/lander` by hand — the real morning move when a batch expired overnight or a
held PR has been fixed and the rebase→test→gate→merge process should re-run
on what's sitting there. Read [README.md](README.md) first.

## Why it is a subagent and not the lead

The lead could resolve conflicts itself. It must not: conflict resolution
reads diffs and hunks, and merging five PRs means five rebases, five test
runs, and every hunk landing in the lead's context — the one thing the whole
design protects.

One lander per PR was also considered. Rejected: each PR rebases on the
previous merge, so they must run strictly serially anyway, and N reports come
back instead of one.

So: one lander, spawned once, returns one screen.

## Input

The branch list (`issue-<N>` names, dictated by the leg-contract) and the
chain topology from dispatch, plus the repo's `.claude/dispatch.json`
(`testCommand`, `hubFiles`, `mergeMethod`). Invoked by hand with no arguments,
it derives the list itself: open PRs whose head branch matches `issue-*`.

## Flow

Sequential over the PRs. Order matters — each rebase sees the previous merge.
Within a stacked chain, land in chain order; otherwise any order works, and
smallest-diff-first is a reasonable tiebreak.

For each PR:

1. **Retarget if stacked.** A chain leg whose base branch just merged is
   retargeted to main before rebasing.
2. **Rebase** on the current main — which by now includes anything landed
   earlier in this same run.
3. **Resolve conflicts.** A conflict confined to a `hubFiles` path is
   keep-both, resolved via `mattpocock-skills:resolving-merge-conflicts`.
   A conflict anywhere else stops this PR — leave it open, comment, move on to
   the next.
4. **Test.** Run `testCommand`. Green or the PR is held.
5. **Merge** (method from `mergeMethod`), if and only if all three hold:
   - tests green after the rebase
   - the PR body's `Review:` line reads `clean`
   - every conflict encountered was hub-file keep-both
6. **Otherwise hold** — PR stays open with a comment saying exactly which of
   the three failed. The ticket stays closed: it was closed because the PR
   exists, which is still true.

**Chain cascade:** a held head holds every downstream leg in its chain, reason
"base held" — their diffs contain the head's commits, and merging one would
smuggle the held change into main. The morning human lands the head; the rest
is boring again and the next `/lander` takes it.

Then read `~/.claude/skills/tidy/SKILL.md` and follow it — there is more to
prune than an hour ago.

## Why it merges at all

Merging was documented as the human's morning job — `relay/SKILL.md` said so,
and forest-shell's `CLAUDE.md` says a background session closes its ticket but
cannot merge. That rule is being deliberately relaxed here, because the morning
cost being removed is triage, and triage is only removed if the trivial cases
are already gone when you wake up.

The gate is what makes it safe. A PR merges only when it is boring: green,
reviewed clean, and conflicting only where conflicts are meaningless. Anything
with a real decision in it is still yours, sitting open with a note. You wake to
a merged main plus a short list of what actually needed a person.

Two alternatives were considered. Staging everything and merging nothing keeps
today's rule but keeps today's morning. Merging everything that resolves
maximizes throughput and lets main be red when you wake — rejected: nothing
merges without green tests, in any mode.

**Authorization.** Harness guidance tells background agents never to merge;
without an explicit grant, a cautious agent at 3am refuses and guts the skill.
So the grant lives here, in the skill text, versioned and auditable: the user
has durably pre-authorized these merges by invoking `/dispatch` against a repo
carrying `.claude/dispatch.json` — the triple gate above *is* the safety
review. Dispatch's spawn prompt repeats the grant.

## Why it does not review

`~/.claude/skills/implement/SKILL.md` ends with "Once done, use /code-review to
review the work." Every leg has therefore already been reviewed by the agent
that wrote it, on a full-context read of its own change, and clause 3 of the
leg-contract records the verdict in the PR body. The lander reads that one
line; it does not run a second review. A re-review from a `fable` agent on a
rebased diff would be both weaker and redundant. A PR with no `Review:` line
is held — fail-safe.

## Report

One screen back to the lead. Three lists:

- **merged** — PR number, ticket, one line
- **held** — PR number and the specific reason (conflict outside hub files /
  tests red / review line missing or not clean / base held)
- **failed** — the leg never produced a PR at all

Plus whatever tidy reported. Nothing else — no diffs, no test output, no
conflict text. If the lead needs detail it can be asked for in the morning,
against a fresh context.

## Open for the build session

None of substance — the 2026-08-03 grilling settled the review signal (PR-body
`Review:` line), the merge method (`mergeMethod` config, default merge-commit),
stacked retargeting (step 1), the chain cascade, and held-PR ticket handling
(stays closed).
