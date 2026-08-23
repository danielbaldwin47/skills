# Unattended fit

The judgement `awake` applies when deriving its backlog.

A ticket is **unattended-fit** when a background agent can close it end to end: implement, verify at a declared seam, open the PR, close the ticket — no human eyes, hands, or hardware in the loop.

## The tell

The disqualifier is a **precondition in the ticket body** that needs a live session or physical hardware: a real compositor effect that must be seen, a second monitor that must be attached, a fingerprint that must be enrolled. Look for the precondition before judging anything else — a missed precondition doesn't fail loudly, it produces a run that proves nothing.

Labels do not discriminate: on forest-shell every awake ticket carries the same `ready-for-agent` label as the fit ones. Only the body tells.

## Seam reachability

Check the ticket's acceptance criteria against the `seams` declared in `.claude/dispatch.json`:

- Every criterion observable by some declared seam's command → fit.
- Any criterion no declared seam can observe → awake work; name the missing observation.
- Repo declares no seams → skip this check and judge by the tell alone.

## Worked examples (from forest-shell)

- **#97 bar blur** — needs a machine where Hyprland blur is known to render; no seam sees composited pixels. Awake.
- **#98 multi-monitor** — fit, *once the nested seam is declared*: `tools/nested-session.sh` drives headless outputs (`NESTED_MONITORS`, `nested_output_add`/`remove`), so per-screen geometry and hotplug are log assertions, no physical monitor involved. This ticket was wrongly bounced to awake when `dispatch.json` listed only `tests` and `capture` — the cautionary example for keeping the seam list complete, which dispatch's preflight now checks.
- **#96 lock failure path** — needs an enrolled fingerprint and a human watching the shake animation. Awake.
- A layout, parsing, or policy ticket checkable at `tests/` or a capture harness — fit.

## Verdict

Binary: `yes` or awake. "Fit if X were true" is not fit — that conditional *is* the tell.
