# Candidates

Specs for four skills, designed in a `/grilling` session on 2026-08-02 and
sharpened in a second grilling on 2026-08-03. Each file is a spec plus the
rationale behind every decision, so a build session can pick up without
re-running either interview.

Nothing here is a `SKILL.md`, so nothing here is loaded as a skill. Building one
means writing `~/.claude/skills/<name>/SKILL.md` from its spec, per
`mattpocock-skills:writing-great-skills`.

## The four

| Spec | Invocation | One line |
|---|---|---|
| [dispatch.md](dispatch.md) | user | Pick tonight's tickets, group them by collision risk, launch them, wait, hand off to the lander. |
| [lander.md](lander.md) | dispatch (path pointer) + user | Rebase, resolve, test, and merge the night's PRs in sequence; report what needed a human. |
| [tidy.md](tidy.md) | lander (path pointer) + user | Prune worktrees whose branch is already in main. |
| [awake.md](awake.md) | user | Guided live-session verification for the tickets no background agent can close. |

Build order: `tidy` → `lander` → `dispatch` → `awake`. Tidy is standalone and
smallest; lander depends on tidy; dispatch depends on lander; awake is
independent of all three and can be built any time.

Four skills, not fewer, was re-examined quality-first and stands: each has a
genuine independent invocation moment (bedtime, morning re-land, anytime
cleanup, daytime verification), and every merge candidate puts material into a
context that must not read it — lander-into-dispatch inlines landing
instructions into the lead, awake-into-dispatch bloats the bedtime path.
Quality is won inside dispatch's folder structure instead (see dispatch.md).

## The problem being solved

The problem is not writing code — background agents already do that well on
this workflow. Two things cost hand-time:

1. **Deciding what runs next.** Not dependency order — the wayfinder map closed
   every blocking edge, so on 2026-08-02 all 17 open forest-shell tickets were
   `ready-for-agent` with nothing gating them. The hard part is **collision
   risk**: which tickets can run concurrently without their branches
   merge-conflicting, and which must be serialized.
2. **Housekeeping.** 17 git worktrees, 7 of them fully merged and dead, plus
   wiring each relay chain by hand before bed.

Everything in these specs follows from those two.

## Cross-cutting decisions

### Every skill is user-invoked; scripted reaches are path pointers

The whole suite carries `disable-model-invocation: true`. A spawned agent does
not need the Skill tool: its prompt says "Read
`~/.claude/skills/relay/SKILL.md` and follow it for #A #B" — a context pointer
by path. That is how dispatch reaches relay and the lander, and how the lander
reaches tidy.

**Rationale.** A model-invoked description sits in every session's context, on
every repo, forever. Every cross-skill reach in this suite is scripted —
dispatch always spawns the lander, the lander always calls tidy — never the
autonomous discovery that `writing-great-skills` reserves model-invocation
for. `/lander` and `/tidy` still work by hand, which their standalone uses
need. Net context load added to a normal session by building all four skills:
zero. The same decision flips the personal `implement` copy (see prerequisite
edits below).

### Repo-agnostic, config-driven

These skills live in `~/.claude/skills` and must work on projects other than
forest-shell. Every repo-specific fact is read from `.claude/dispatch.json` in
whatever repo the skill runs in.

**Rationale.** The design leans hard on forest-shell specifics — the hub-file
allowlist, the `ready-for-agent` label, `tests/run.sh`, the three test seams.
Hardcoding any of them buys shorter skills and costs the next project a
rewrite. A config file is the only pairing where the skills are both globally
available and correct.

**Missing or malformed config → the skill stops and says so.** It does not
infer a test command or guess a label. A dispatcher that guesses wrong launches
five agents at the wrong work overnight; a dispatcher that stops costs one
minute of setup. This mirrors the preflight already in `relay/SKILL.md`, whose
whole argument is that a bad argument should ground the leg rather than get
substituted.

#### `.claude/dispatch.json`

Settled shape — the build session should treat field names as settled but is
free to add:

```json
{
  "ticketLabel": "ready-for-agent",
  "testCommand": "tests/run.sh",
  "hubFiles": [
    "Core/ServiceInit.qml",
    "Core/SettingsSchema.qml",
    "CLAUDE.md",
    "tools/*.sh"
  ],
  "worktreeDir": ".claude/worktrees",
  "defaultBatch": 5,
  "mergeMethod": "merge",
  "legTimeoutBase": "2.5h",
  "seams": [
    { "name": "tests", "command": "tests/run.sh", "covers": "decisions: policy, formatting, parsing, merging, migration, thresholds" },
    { "name": "nested", "command": "tools/nested-session.sh", "covers": "anything needing a real compositor: lifecycles, IPC, layer-shell, focus" },
    { "name": "capture", "command": "tools/capture-harness.sh", "covers": "client-side pictures: layout, colour, opacity compositing" }
  ]
}
```

`collisionPaths` (an earlier field) is gone — the collision rule needs no
allowlist (see dispatch.md). `mergeMethod` matches forest-shell's merge-commit
history; a squashing repo sets `"squash"`. `legTimeoutBase` feeds the batch
tripwire (see dispatch.md). `seams` is consumed by `awake` and by dispatch's
scout; a repo with no seams declared simply skips that judgement.

### The leg-prompt contract

Every launched agent — singleton, chain head, and (via relay) legs 2..n —
gets three clauses in its prompt, kept as one reference block in dispatch's
folder (`leg-contract.md`):

1. Work in a worktree/branch named `issue-<N>`.
2. When the PR is open and the work complete, close #N with a comment linking
   the PR.
3. After `/code-review`, write `Review: clean` or `Review: findings held` in
   the PR body.

**Rationale.** (1) The tripwire greps `gh pr list` against a branch list the
lead knows before launch, and the lander receives the same list — no guessing
at auto-generated worktree names. (2) Chain gating becomes repo-independent:
relay gates on the upstream *ticket* closing, and without this clause a repo
whose CLAUDE.md lacks forest-shell's close-on-PR rule strands every chain at
the 12h expiry. (3) The lander gets a machine-readable review verdict — one
`gh` call, no diff read, and a missing line means held, which fails safe.

### Lead-agent context is a hard budget

The orchestrating agent must not accumulate context across a long night. It
reads no ticket bodies, no diffs, no conflict hunks, no agent transcripts. Over
a full run its context grows by: the plan table, one tripwire line, one lander
report.

**Rationale.** This was the user's stated constraint and it is the same lesson
`relay/SKILL.md` already encodes — "never poll from the session itself — every
model-side wake pays the whole accumulated context again just to find the gate
shut." Every bulky read is delegated to a subagent that returns a summary.

The corollary: **one ticket, one session, ~120k context budget per implement
agent.** That is why colliding tickets are not collapsed into a single agent
running two tickets back to back, even though that would trivially avoid the
conflict — it would double one agent's context.

### Nothing merges without green tests

In every mode, at every autonomy level. Not negotiable by argument or flag.

## Environment as of 2026-08-02

Facts gathered during the grilling, recorded because the specs reference them.
**Verify before relying on any of them** — they are a snapshot.

**Existing personal skills** (`~/.claude/skills`, a git repo):
`relay`, `implement`. `implement` is a copy of `mattpocock-skills:implement`
made model-invocable; it ends by running `/code-review`, which is why the
lander does not review again. (The model-invocable part is being reverted —
see prerequisite edits.)

**Plugins installed:** `mattpocock-skills`, `caveman`. Already covered by
mattpocock and therefore *not* proposed here: `to-tickets`, `triage`,
`handoff`, `research`, `prototype`, `tdd`, `code-review`,
`resolving-merge-conflicts`, `domain-modeling`, `writing-great-skills`,
`wayfinder`.

**forest-shell:** Quickshell/QML desktop shell for Hyprland. Ticket-driven off
a wayfinder map (issue #1), which reached its destination 2026-08-01 — 28
tracer-bullet build tickets published, all blocking edges closed. 17 open
issues, all `ready-for-agent`. 2 open draft PRs (#115, #116).

**Worktree state:** 17 worktrees under `.claude/worktrees/`. 7 branches fully
merged into main (`worktree-issue-37/38/39/41/71`, `worktree-relay-claude-md-line`,
`research/claude-cli-contract`). 4 unmerged branches whose tickets are already
closed (`issue-43/44/45/46`) — the ambiguous class tidy reports rather than
deletes. 2 locked with live PRs.

**File churn, last 12 merges** — this is the measurement the collision rule is
built on:

```
5  tools/capture-harness.sh      27  tests/
5  Core/ServiceInit.qml          22  tools/
4  tools/services-harness.sh     19  Core/
4  tools/launcher-harness.sh     15  Services/Launcher/
4  CLAUDE.md                     14  Surfaces/Bar/Modules/
3  Core/SettingsSchema.qml       13  Surfaces/Drawers/
```

`Core/ServiceInit.qml` and `Core/SettingsSchema.qml` are registration points —
nearly every feature ticket appends a line to each. That is the entire reason
the hub-file allowlist exists.

**forest-shell `CLAUDE.md`** documents the three test seams and the rule that a
ticket whose acceptance criteria cannot be checked at any seam is not ready to
build. The rule exists because seven tickets ran green against `tests/` alone
and the first pass under a real compositor produced eight bugs at once
(#74–#81), all living at a seam that did not yet exist.

## Deferred, not dropped

Two skills came up and were consciously postponed:

- **`seam`** — pre-flight a ticket against the repo's seam rule; name the seam
  and the harness invocation, or declare the ticket not ready. Natural gate
  inside dispatch's scout. Deferred because the scout can carry a lightweight
  version of the judgement first, and a separate skill is worth building only
  if that proves too thin.
- **`budget`** — run the acceptance budgets from forest-shell #22 (first frame
  ≤1.5 s, interactive ≤2 s, idle ≤0.5 % CPU, <5 wakeups/s, ≤8 ms GPU frame at
  60 Hz) and diff against target, so #95-class re-measures stop being tickets.
  Deferred as the least urgent of the three.

## Prerequisite edits outside these specs

No frontmatter flip on relay — it stays user-invoked; dispatch's chain legs
reach it by path pointer. Small text edits instead:

- `relay/SKILL.md` step 3: legs adopt clause 3 of the leg-prompt contract —
  write the `Review:` verdict line into the PR body ("complete" already
  requires the review to have run; this records its outcome where the lander
  reads it).
- `relay/SKILL.md` step 3: "merging is the human's morning job" is stale —
  landing is downstream's job (the lander's, on dispatch nights).
- `relay/SKILL.md` step 2 invokes `implement` by bare skill name; change to a
  path pointer (`Read ~/.claude/skills/implement/SKILL.md and follow it`),
  because of the next edit.
- `implement/SKILL.md` gains `disable-model-invocation: true`. The personal
  copy existed only to be reachable from agents; path pointers retire that
  reason, and its description — the fattest in the suite — leaves every
  session on every repo. Interactive use is `/implement` by name, unchanged.
