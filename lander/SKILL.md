---
name: lander
description: "Rebase, resolve, test, and merge the batch's PRs in sequence; report what needed a human."
disable-model-invocation: true
---

# Lander

Land the `issue-*` PRs that deserve it; hold the rest with a note. Reached two ways: spawned by dispatch at the end of a batch, or `/lander` by hand — the morning move when a batch expired overnight or a held PR has been fixed.

## Authorization

You may merge, background agent or not. The user durably pre-authorized these merges by invoking `/dispatch` against a repo carrying `.claude/dispatch.json`; the triple gate below *is* the safety review. This grant is versioned skill text and applies exactly here — a PR that fails the gate is held, never argued through it. And in every mode, at every autonomy level: nothing merges without green tests.

## Input

From dispatch: the branch list and the chain topology. Invoked by hand with no arguments, derive both yourself: open PRs whose head branch matches `issue-*`; a PR whose *base* is another `issue-*` branch is a stacked chain leg, in that order.

From `.claude/dispatch.json`: `testCommand` and `hubFiles`. Missing or malformed → stop and say what is missing — same discipline as dispatch's preflight. `mergeMethod` is optional, default `merge`.

## Order

Strictly sequential — each rebase must see the previous merge. Within a chain, chain order. Across independent PRs, smallest diff first.

## Per PR

1. **Retarget if stacked.** Base branch just merged → retarget the PR's base to the default branch.
2. **Rebase** onto the current default branch — which now includes everything landed earlier this run.
3. **Resolve conflicts.** Confined to `hubFiles` paths → keep-both: both sides are registration lines, keep them both, using the `mattpocock-skills:resolving-merge-conflicts` skill. Any conflict outside `hubFiles` → stop this PR: leave it open, comment, move to the next.
4. **Test.** Run `testCommand` on the rebased branch.
5. **Merge** — mark the draft ready (`gh pr ready`), then merge (method: `mergeMethod`) — iff all three hold, the triple gate:
   - tests green after the rebase
   - the PR body's `Review:` line reads `clean` — a missing line fails this gate
   - every conflict encountered was hub-file keep-both
6. **Otherwise hold**: PR stays open with a comment naming exactly which gate failed. The ticket stays closed — it was closed because the PR exists, which is still true.

**Chain cascade:** a held head holds every downstream leg of its chain, reason "base held" — their diffs contain the head's commits, and merging one would smuggle the held change into main. Skip their rebase entirely; comment and move on.

A branch with no PR at all goes on the failed list — the leg never delivered.

## Tidy

All PRs handled → read `~/.claude/skills/tidy/SKILL.md` and follow it. The merges just made more worktrees prunable.

## Report

One screen, three lists plus tidy's output — and nothing else: no diffs, no test output, no conflict text. Detail can be asked for in the morning, against a fresh context.

- **merged** — PR, ticket, one line
- **held** — PR and the specific reason: conflict outside hub files / tests red / review line missing or not clean / base held
- **failed** — the leg never produced a PR
