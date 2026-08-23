# implement-relay

Implement a piece of work based on a spec or set of tickets: TDD at pre-agreed seams where possible, typechecking and single test files run regularly, the full suite once at the end, then `/code-review`, then commit.

This is a personal fork of `mattpocock-skills:implement`, renamed `implement-relay` to keep it distinct from the plugin skill, and kept user-invoked (`disable-model-invocation: true`) so its description costs no context in ordinary sessions. Spawned agents reach it by path pointer instead — [relay](../relay/README.md) prompts its downstream leg with "read `~/.claude/skills/implement-relay/SKILL.md` and follow it", and [dispatch](../dispatch/README.md) does the same for every singleton and chain head.

For work without a ticket the skill ends at "commit to the current branch", and the human decides what happens after. When the work comes from a ticket `#N`, the skill carries the four shipping clauses itself — branch named `issue-<N>`, push and draft PR, the `Review:` verdict line, land per a repo's self-landing grant when one exists, close the ticket last — restating the [leg contract](../dispatch/leg-contract.md) that dispatch appends to every agent's prompt; when both are in play the contract remains the authority.

Upstream sync check, worth one run whenever the plugin updates (only the core body above the shipping clauses is shared):

```sh
diff ~/.claude/skills/implement-relay/SKILL.md \
  "$(jq -r '.plugins["mattpocock-skills@claude-plugins-official"][0].installPath' \
     ~/.claude/plugins/installed_plugins.json)/skills/engineering/implement/SKILL.md"
```
