# Tripwire

One persistent Monitor, shell-side; the lead wakes exactly once, on the single line the loop emits. Never poll from the session — every model-side wake pays the whole accumulated context again.

Before arming:

1. Write the branch list to a temp file, one exact branch name per line (`issue-72`, `issue-55`, …).
2. `want` = number of branches. `timeout` = `legTimeoutBase` converted to seconds × the deepest chain length (base `2.5h`, deepest chain 3 → 27000).

```bash
start=$(date +%s); want=<N>; timeout=<seconds>
while :; do
  have=$(gh pr list --state open --limit 100 --json headRefName -q '.[].headRefName' 2>/dev/null \
           | grep -cxFf <branch-list-file> || true)
  [ "$have" -ge "$want" ] && { echo "batch complete: $have/$want PRs up"; exit 0; }
  [ $(( $(date +%s) - start )) -ge "$timeout" ] && { echo "batch expired: $have/$want PRs up"; exit 0; }
  sleep 60
done
```

Notes that keep the count honest:

- `grep -x` matches whole lines only, so `issue-7` never counts `issue-72`'s PR.
- `--state open` includes drafts — the legs open draft PRs.
- The `|| true` keeps a zero count (grep exit 1) and a transient `gh` failure (empty input) both reading as `have=0`; the loop just tries again next minute.

Either line means proceed to hand-off. A loose timeout only delays the report — it never wastes compute, because the lander lands whatever PRs exist on expiry.
