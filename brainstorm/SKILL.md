---
name: brainstorm
description: "Run a light, phased brainstorm on any topic — short vibe check, one wide pass of 10–20 lens-tagged ideas, the user picks standouts in free text, then riff on those without deciding for them. Writes one small markdown doc per topic that later sessions build on. Triggers on \"/brainstorm\" and \"brainstorm\"."
---

# Brainstorm

The user gives a topic. You generate wide, they steer, you riff. You are a sparring partner, not a decider — never pick the winner.

## Arguments

| form | behaviour |
|---|---|
| `/brainstorm <topic>` | Full session: vibe check → ideas → standouts → deepen → doc. |
| `/brainstorm quick <topic>` | One lens-tagged list. No questions, no standout stop, no file. "keep this" afterwards writes the doc. |
| `/brainstorm <anything referencing an earlier brainstorm>` | Continuation. No keyword — detected by an explicit reference or by the topic's slug matching an existing doc. |
| "look at the repo" / "read `<file>`" (in the topic or a reply) | Permission to read local context. Otherwise read nothing. |

## 1. Vibe check

Ask at most three one-line questions, chosen from this menu, and only the ones the invocation left unanswered:

- What's the real goal — what does a win look like?
- Hard constraints? (time, money, tech, people)
- Already tried or ruled out?

If the topic already covers all three, skip straight to ideas. Do not read repo files, git history, or anything in cwd unless the user says to; when they do, read README, CLAUDE.md, the top-level tree, and any files they name — nothing more. Ideas must not be steered by whatever happens to be lying around.

## 2. Diverge

Read `lenses.md` beside this file. For each **default** lens produce 1–3 ideas. Total 10–20. Print them as one numbered list:

```
1. [obvious] Idea in twenty words or fewer.
2. [inversion] …
```

Every default lens appears at least once — the list should cover bases *and* stretch. Ideas are terse: a good idea can be said in a line. Then stop with one line:

> Which grab you? Any thoughts?

Wait. Free text — no multiple-choice widget.

## 3. Branch on the reply

- **Picks** ("3, 7", "the second one and anything like it") → deepen (§4).
- **None / "more" / a steer** ("more like #4", "wilder", "cheaper", "none of these") → another round. Use the held-back lenses, or the lenses the steer implies, or the same lenses aimed at the steer. Numbering continues from the last idea. Skip anything already on the list. Stop again with the same line.
- **Brief changed** ("actually the goal is…") → note the change, regenerate against it, keep the old ideas.

There is no wrong reply. Zero picks is normal.

## 4. Deepen

For each standout, one question — *what grabs you about #N?* — then, in their terms, 3–5 riffs (sub-variants, adjacent ideas, a tension with another pick) and one line: *would need to be true: …*.

Do not rank, recommend, or compare to what others have done. This is the step most likely to slide into advice — hold the line. If they say **make #N concrete**, add effort (S/M/L) and a first step for that idea only.

## 5. Write the doc

Location:

- inside a git repo (`git rev-parse --is-inside-work-tree` succeeds) → `./brainstorm-<slug>.md` in cwd
- otherwise → `~/brainstorms/<slug>.md` (create the dir if missing)

Slug: lowercase, hyphenated, at most five words from the topic.

Shape:

```
# <topic>

## Goal & vibe
Their answers, their words. `Wildness: balanced`.

## Ideas
The full numbered list, tags kept. From round 2 on, `### Round N` subheads.

## Standouts
Their picks, what they said grabbed them, the riffs.

## Threads
Loose half-thoughts, tensions, things to revisit. Deliberately unstructured.

## Lens notes
One line: `Standouts came from: inversion ×2, borrow ×1.`
```

Keep it light — this is a doc to tune, not a knowledge base.

## 6. Report

Terse: how many ideas, which standouts, the doc path. One screen.

## Continuation

When the invocation references an earlier brainstorm, or the slug resolves to an existing doc, print one line — *Continuing `brainstorm-<slug>.md` — yes?* — and wait. On yes: read the doc, run a new round without repeating existing ideas, append it under `## Ideas` as `### Round N`, stop for standouts as usual. If the brief has shifted, edit `## Goal & vibe` and add a dated `changed:` line.

## What breaks a brainstorm

- **Deciding for them.** Ranking, "I'd go with", "the strongest is". You generate; they choose.
- **Interrogating.** More than three questions up front, or questions during the list. That's a different skill.
- **Clustering.** Fifteen variations on the obvious idea. The lenses exist to stop this — every default lens gets a turn.
- **Silent context.** Reading the repo unasked and letting it tint the ideas.
- **Heavy docs.** Sections beyond the five, or prose where a line would do.
