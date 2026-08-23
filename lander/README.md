# lander

> **Superseded** in repos whose CLAUDE.md grants self-landing: every leg lands its own PR behind the grant's gates, so no separate landing pass exists to run. Kept for repos without the grant, and as the by-hand move for landing a fixed held PR.

The landing crew. Takes the batch's `issue-*` PRs and merges the ones that deserve it — strictly in sequence: retarget if stacked, rebase onto the current default branch, resolve hub-file conflicts keep-both, run the test command, merge if the triple gate holds. Anything with a real decision in it is held open with a comment naming exactly which gate failed. Reached two ways: spawned by [dispatch](../dispatch/README.md) at the end of a batch, or `/lander` by hand — the morning move when a batch expired overnight or a held PR has been fixed.

## The triple gate

A PR merges if and only if:

1. tests are green after the rebase,
2. the PR body's `Review:` line reads `clean` (a missing line fails the gate),
3. every conflict encountered was a hub-file keep-both.

A PR is merged only when it is *boring*. Green, reviewed clean, conflicting only where conflicts are meaningless. Everything else waits for a person — which is the point: the morning cost being removed is triage, and triage is only removed if the trivial cases are gone when you wake up.

After every merge the lander immediately verifies the commits actually reached the default branch (`git merge-base --is-ancestor`). A PR can read MERGED while its work never lands — the #179/#200 trap, a stacked PR merged into a stale or just-consumed base — and on that failure the head branch is relanded as a fresh PR against the default branch, named in the report. On a dispatch night the batch's intent — plan table, chains, branch list — comes from the newest run record in `~/.claude/dispatch-runs/`.

## Why it merges at all

Background agents are ordinarily told never to merge, and without an explicit grant a cautious agent at 3am refuses and guts the skill. So the grant lives in the skill text, versioned and auditable: the user durably pre-authorized these merges by invoking `/dispatch` against a repo carrying `.claude/dispatch.json`, and the triple gate *is* the safety review. A PR that fails the gate is held, never argued through it. And in every mode, at every autonomy level: nothing merges without green tests.

## Why it doesn't re-review

Every leg already ran `/code-review` on a full-context read of its own change — [implement-relay](../implement-relay/README.md) ends with it — and clause 3 of the [leg contract](../dispatch/leg-contract.md) records the verdict as one machine-readable line in the PR body. The lander reads that line and nothing else. A second review from a fresh agent on a rebased diff would be both weaker and redundant; a missing line simply holds the PR, which fails safe.

## Chains

Stacked PRs land in chain order, each rebase seeing the previous merge. A held chain head holds every downstream leg (reason: "base held") — their diffs contain the head's commits, and merging one would smuggle the held change into the default branch. Their rebases are skipped entirely; the human lands the head, and the next `/lander` takes the rest.

## Why a subagent, not the lead

Conflict resolution reads diffs and hunks; five PRs means five rebases and five test runs. Running that in the dispatch lead would pour all of it into the one context the whole design protects. One lander, spawned once, returns one screen: **merged** / **held** (with the failed gate) / **failed** (the leg never produced a PR) — plus the output of [tidy](../tidy/README.md), which it runs last, because the merges it just made turned more worktrees prunable.
