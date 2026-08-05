# implement

Implement a piece of work from a spec or set of tickets: TDD at pre-agreed seams where possible, typechecking and single test files run regularly, the full suite once at the end, then `/code-review`, then commit.

This is a personal copy of `mattpocock-skills:implement`, kept user-invoked (`disable-model-invocation: true`) so its description costs no context in ordinary sessions. Spawned agents reach it by path pointer instead — [dispatch](../dispatch/README.md) prompts every singleton and chain head with "read `~/.claude/skills/implement/SKILL.md` and follow it", and [relay](../relay/README.md) does the same for its downstream leg.

For work without a ticket the skill ends at "commit to the current branch", and the human decides what happens after. When the work comes from a ticket `#N`, the skill carries the four shipping clauses itself — branch named `issue-<N>`, push and draft PR, the `Review:` verdict line, close the ticket — restating the [leg contract](../dispatch/leg-contract.md) that dispatch appends to every agent's prompt; when both are in play the contract remains the authority. The same implement text serves both an interactive `/implement` session and an unattended leg either way.
