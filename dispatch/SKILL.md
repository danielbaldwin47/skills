---
name: dispatch
description: "Bedtime dispatcher — pick tonight's tickets, group them by collision risk, launch overnight agents, wait once, hand the PRs to the lander."
disable-model-invocation: true
---

# Dispatch

Launch a night of unattended ticket work, then walk away. You are the lead, and your context is a hard budget: over the whole run it grows by one plan table, one tripwire line, one lander report. Every bulky read — ticket bodies, diffs, conflict hunks, agent transcripts — belongs to a subagent that returns a summary. A step that would put one in your context is being done wrong.

## Arguments

| Form | Meaning |
|---|---|
| `/dispatch` | default batch (`defaultBatch` from config), anything unattended-fit |
| `/dispatch 4` | bare integer = batch size |
| `/dispatch settings` | free text = scope filter, matched semantically by the scout |
| `/dispatch settings 4` | both |
| `/dispatch #72 #55 #56` | exactly these tickets — bypasses selection, never collision grouping |

Batch size counts tickets, not parallel tracks: a chain of three is three of the batch.

## 1. Preflight

Read `.claude/dispatch.json` at the repo root. Required: `ticketLabel`, `testCommand`, `hubFiles`, `defaultBatch`, `legTimeoutBase`. Optional: `legModel` (model for implementation legs, default `opus`), `maxStackDepth` (longest allowed chain, default `3`). Missing file, missing field, or unparseable JSON → stop and say exactly what is missing; never substitute a guessed test command or label — a dispatcher that guesses wrong spends a night of compute on the wrong work.

Then check the `seams` list is complete: every top-level seam runner present in the repo (`tests/run.sh`, `tools/nested-session.sh`, `tools/capture-harness.sh`) must appear in some seam's `command`. One exists that no seam names → stop and name it — a missing seam makes the scout silently reject every ticket only that seam can verify (this happened: #98 and #70 were bounced to `awake:` while `tools/nested-session.sh` sat unlisted).

Then confirm the working tree is clean and on the default branch. Preflight is complete when the config is loaded and all three checks pass.

## 2. Scout

Spawn one subagent, model `fable`. Its prompt: the contents of [scout.md](scout.md), plus this run's facts — candidate source (`ticketLabel`, or the explicit `#` list), scope text if any, batch size, `hubFiles`, `seams`, `maxStackDepth`. It returns one compact table — working set, fit verdict, and group per ticket — plus the chains and the excluded awake list. That table is the only thing the tickets ever contribute to your context.

Fewer fit tickets than requested → proceed with fewer, and say so in the plan.

## 3. Check the plan

Spot-check the table: no two tickets running in parallel may share a working-set path outside `hubFiles`. Overlap found → chain them. The rule fails toward serialization — one night slower — never toward a conflict. The check is complete when every parallel pair has been compared and shares no non-hub path.

Chains land as stacked PRs, so chain order is land order. The scout's head choice (smallest working set first, by default) stands unless the table itself contradicts it. A chain longer than `maxStackDepth` is cut at the cap — the tail tickets drop from the batch and are named in the plan; deep stacks are unreviewable against the default branch and the stacked-merge API is still classifier-gated.

## 4. Launch

Print the plan and launch in the same turn — the plan costs nothing to show, and the point of the command is walking away.

Every agent's prompt ends with the three clauses of [leg-contract.md](leg-contract.md), verbatim. Every leg launches with `model: <legModel>` set explicitly — legs do the implementation, the one place model choice matters most; never leave them inheriting whatever the lead happens to run.

- **Singleton, and every chain head:** one background agent, `isolation: worktree`, prompted: read `~/.claude/skills/implement-relay/SKILL.md` and follow it for `#N` — plus the contract.
- **Chain legs 2..n:** one background agent each, prompted: read `~/.claude/skills/relay/SKILL.md` and follow it for `#prev #next` — plus the contract. Relay waits shell-side for `#prev` to close, then stacks its PR on `#prev`'s.

One ticket, one agent, its own worktree and PR — never two tickets through one agent, whatever it saves.

Then write the run record: `~/.claude/dispatch-runs/<UTC-timestamp>.json` (create the directory if absent; never under `~/.claude/daemon/` — that tree belongs to the daemon supervisor, which quarantined the 2026-08-04 record into `daemon/dispatch/rejected/` within a minute) with the plan table, chains, awake list, branch list, and per-leg model. Update it at each later step (tripwire verdict, lander report) — it is the only artifact that survives an interrupted run, and the morning `/lander` reads it to know what the batch intended.

Launch is complete when every planned ticket has an agent running, the branch list (`issue-<N>` per ticket) is written down for the tripwire, and the run record exists.

## 5. Wait — one wake

Arm one persistent Monitor running the loop in [tripwire.md](tripwire.md). Timeout: `legTimeoutBase` × the deepest chain length. The loop emits one line — `batch complete` or `batch expired` — and that line is your only wake signal. Per-agent completion notifications will arrive first and they are traps: on one, do nothing — no summary, no `TaskStop`, no peeking at the PR list — and go back to idle. Acting on them is how a lead pays its whole context per leg and lands the batch off half a picture (both prior runs did exactly this). Only the tripwire's own line means proceed: a partial night lands what it has. Record the verdict line in the run record.

## 6. Hand off

Spawn the lander, model `fable`, prompted: read `~/.claude/skills/lander/SKILL.md` and follow it. Give it the branch list, the chain topology (it needs chain order for retargeting and cascade holds), and repeat the grant: *the user pre-authorized these merges by invoking `/dispatch` against a repo carrying `.claude/dispatch.json`; the lander's triple gate is the safety review.*

Append the lander's report to the run record, then relay it to the user verbatim. That report is the last thing in your context, and the end of the run.
