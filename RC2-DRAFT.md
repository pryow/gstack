# `/autoplan` inlines sub-skills, so their `gstack-timeline-log` preamble never fires

Run `/autoplan` and the sub-skills (`plan-ceo-review`, `plan-eng-review`, `plan-design-review`) leave their artifacts on disk but never write to `~/.gstack/projects/<slug>/timeline.jsonl`. Any skill that reads timeline.jsonl to decide "has the user done X recently?" answers "no" incorrectly when X went through autoplan.

`autoplan/SKILL.md.tmpl` lines 272 / 397 / 481 contain inline directives telling the agent to follow each sub-skill SKILL.md to full depth. The agent reads the sub-skill file and follows the instructions, but doesn't re-execute its bash preamble. The preamble is where `gstack-timeline-log` lives (injected by `scripts/resolvers/preamble/generate-preamble-bash.ts:73` at gen-skill-docs time). So the started/completed timeline events for each sub-skill never land.

Autoplan does log its own pass via `gstack-review-log` at lines 863, 865, 870, 875, 880, 882, 887, 892, which writes to `~/.gstack/projects/<slug>/<branch>-reviews.jsonl` with `via:"autoplan"`. Good signal for the review pipeline. Wrong file for timeline-consuming code.

## Repro

```
$ bun run dev /autoplan some-branch
$ ls ~/.gstack/projects/<slug>/
  tasks-eng-review-<ts>.jsonl
  tasks-design-review-<ts>.jsonl
  <branch>-reviews.jsonl          # gets 4 entries via gstack-review-log
  timeline.jsonl                  # gets 1 entry: autoplan started/completed only
```

Compare with running plan-eng-review directly: timeline.jsonl gets a started + completed pair. Via autoplan: nothing.

## Fix

Symmetric to the existing `gstack-review-log` block. Add a `gstack-timeline-log` call right next to each `gstack-review-log` line at 863-892, with the same `via:"autoplan"` field and the same `SKILL_NAME` substitution the per-skill preamble uses. Five sub-skill emissions, ten extra log lines once you count started + completed pairs. No new binary, no schema change. Same shape as the autoplan-voices log entries that already share that block.

A lightweight static-grep test alongside what `test/static-no-legacy-writes.test.ts` does for #1677 would lock the invariant: every `gstack-review-log` call in `autoplan/SKILL.md.tmpl` must be paired with a `gstack-timeline-log` call. Stops the same drift on the next sub-skill addition.

## Related

#1677. Found during the same audit. Different code path (writer/reader split on developer-profile.json vs missing-writer on timeline.jsonl), different fix.

Filing as a regression report, not a PR. Happy to send one if useful.
