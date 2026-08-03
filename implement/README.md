# implement

Implement a piece of work from a spec or set of tickets: TDD at pre-agreed seams where possible, typechecking and single test files run regularly, the full suite once at the end, then `/code-review`, then commit.

This is a personal copy of `mattpocock-skills:implement`, kept user-invoked (`disable-model-invocation: true`) so its description costs no context in ordinary sessions. Spawned agents reach it by path pointer instead — [dispatch](../dispatch/README.md) prompts every singleton and chain head with "read `~/.claude/skills/implement/SKILL.md` and follow it", and [relay](../relay/README.md) does the same for its downstream leg.

The skill itself ends at "commit to the current branch" — deliberately. Pushing, opening the draft PR, closing the ticket, and writing the `Review:` verdict line are supplied by the [leg contract](../dispatch/leg-contract.md) that dispatch appends to every agent's prompt, so the same implement text serves both an interactive `/implement` session (where the human decides what happens after the commit) and an unattended leg (where the contract decides).
