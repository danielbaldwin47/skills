# brainstorm

The idea generator. `/brainstorm <topic>` runs a light, phased session on anything — code, product, names, writing, life — and leaves behind one small markdown doc per topic. Standalone; it doesn't call other skills and nothing calls it.

## Stance

Sparring partner, not decider. The skill generates wide and the user steers: they pick standouts in free text, say what grabs them, and the skill riffs on *that*. It never ranks, never recommends, never compares to prior art unless asked. The deepen step is where this most easily slips — the runbook says so out loud.

## Shape

1. **Vibe check** — ≤3 one-line questions (goal, constraints, already-tried), only the ones the topic left open. Skipped if covered.
2. **Diverge** — 10–20 numbered ideas, ≤20 words each, tagged `[lens]`. Every default lens gets at least one idea so the list covers bases *and* stretches.
3. **Stop** — "Which grab you? Any thoughts?" Zero picks is fine; "more like #4" or "wilder" runs another round.
4. **Deepen** — per standout: what grabs you → 3–5 riffs → one "would need to be true" line. Effort and first step only on request.
5. **Doc** — `./brainstorm-<slug>.md` inside a repo, else `~/brainstorms/<slug>.md`.

`quick` skips the questions, the stop, and the file. Continuation needs no keyword — a reference or a slug match, one-line confirm, then a new round appended.

## Files

- `SKILL.md` — the runbook.
- `lenses.md` — the lens table. Eight default (obvious, inversion, borrow, remove-constraint, add-constraint, cheapest, most-ambitious, question-premise), two held back for later rounds (combine, archetype). **This is the tuning surface.** Edit it directly.
- `README.md` — this map.

## Why the doc is light

Five sections, one of them (`Threads`) deliberately loose. The doc exists so a later session can build on the earlier one and so the `Lens notes` line accumulates a hint about which lenses produce ideas the user actually likes. Nothing reads that line programmatically — it's for a human pruning `lenses.md`. Anything heavier turns a brainstorm into a knowledge base, which is a different job.

## Why nothing is read by default

Ideas tinted by whatever's in cwd are a silent bias. The user opts in with "look at the repo" or by naming a file; even then it's README, CLAUDE.md, the top-level tree, and named files only.

## Not yet

Parallel sub-agents per lens, a wildness dial, automatic lens tuning, invoking other skills. All deliberately left out of v1; add when the default proves too shallow.
