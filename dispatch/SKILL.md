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

Read `.claude/dispatch.json` at the repo root. Required: `ticketLabel`, `testCommand`, `hubFiles`, `defaultBatch`, `legTimeoutBase`. Missing file, missing field, or unparseable JSON → stop and say exactly what is missing; never substitute a guessed test command or label — a dispatcher that guesses wrong spends a night of compute on the wrong work. Then confirm the working tree is clean and on the default branch. Preflight is complete when the config is loaded and both checks pass.

## 2. Scout

Spawn one subagent, model `fable`. Its prompt: the contents of [scout.md](scout.md), plus this run's facts — candidate source (`ticketLabel`, or the explicit `#` list), scope text if any, batch size, `hubFiles`, `seams`. It returns one compact table — working set, fit verdict, and group per ticket — plus the chains and the excluded awake list. That table is the only thing the tickets ever contribute to your context.

Fewer fit tickets than requested → proceed with fewer, and say so in the plan.

## 3. Check the plan

Spot-check the table: no two tickets running in parallel may share a working-set path outside `hubFiles`. Overlap found → chain them. The rule fails toward serialization — one night slower — never toward a conflict. The check is complete when every parallel pair has been compared and shares no non-hub path.

Chains land as stacked PRs, so chain order is land order. The scout's head choice (smallest working set first, by default) stands unless the table itself contradicts it.

## 4. Launch

Print the plan and launch in the same turn — the plan costs nothing to show, and the point of the command is walking away.

Every agent's prompt ends with the three clauses of [leg-contract.md](leg-contract.md), verbatim.

- **Singleton, and every chain head:** one background agent, `isolation: worktree`, prompted: read `~/.claude/skills/implement/SKILL.md` and follow it for `#N` — plus the contract.
- **Chain legs 2..n:** one background agent each, prompted: read `~/.claude/skills/relay/SKILL.md` and follow it for `#prev #next` — plus the contract. Relay waits shell-side for `#prev` to close, then stacks its PR on `#prev`'s.

One ticket, one agent, its own worktree and PR — never two tickets through one agent, whatever it saves. Launch is complete when every planned ticket has an agent running and the branch list (`issue-<N>` per ticket) is written down for the tripwire.

## 5. Wait — one wake

Arm one persistent Monitor running the loop in [tripwire.md](tripwire.md). Timeout: `legTimeoutBase` × the deepest chain length. The loop emits one line — `batch complete` or `batch expired` — and that line is your only wake signal; per-agent completion notifications are noted and ignored. Either line means proceed: a partial night lands what it has.

## 6. Hand off

Spawn the lander, model `fable`, prompted: read `~/.claude/skills/lander/SKILL.md` and follow it. Give it the branch list, the chain topology (it needs chain order for retargeting and cascade holds), and repeat the grant: *the user pre-authorized these merges by invoking `/dispatch` against a repo carrying `.claude/dispatch.json`; the lander's triple gate is the safety review.*

Relay its one-screen report to the user verbatim. That report is the last thing in your context, and the end of the run.
