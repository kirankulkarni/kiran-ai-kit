# Trigger-description optimization — 2026-07-27

## The bundled optimizer does not work for an already-installed skill

`scripts/run_loop.py` creates a throwaway pseudo-skill named `<name>-skill-<uuid>` in
`.claude/commands/` and counts a trigger only when that exact name appears in the tool input.
`~/.claude/skills/gauntlet/` is user-level and visible to every `claude -p`, so it shadows the
pseudo-skill: the model invokes `gauntlet`, the substring check fails, and every query scores 0/3 --
positives and negatives alike. A uniform zero across a whole eval set is an instrumentation failure,
not a description failure. Verified by probing directly: the first tool call was
`Skill {'skill': 'gauntlet'}`.

Replaced with `scratchpad/measure_trigger.py`, which measures the real installed skill: 3 votes per
query, majority threshold, trigger = the skill is consulted within the first 3 tool calls, process
killed early so a trigger does not start an actual Codex run.

## Result (20 queries, 3 votes each)

accuracy 70% | precision 100% | recall 40% (tp=4 fp=0 tn=10 fn=6)

Precision held at 100% across four separate measurements, on deliberately tricky near-misses:
GitHub PR review, security review, "critique my design doc", a rename, a pricing question about the
Codex models themselves. The skill does not over-trigger, so there is headroom to push recall.

## Two measurement artifacts inflating the false negatives

1. **`/gauntlet ...` scores 0/3** -- `claude -p` does not process slash commands the way an
   interactive session does. This is the primary invocation path and is not measurable this way.
2. **"implement X, then review it" scores 0/3** (4 of the 6 misses). The model correctly begins
   implementing and would consult the skill *after* the code exists -- which is exactly right for the
   light tier. The within-3-tool-calls window scores deferred consultation as a miss.

So real-world recall is higher than 40%. Do not chase the remaining misses with an ever-pushier
description: the measurable headroom is small and precision is the thing worth protecting.

## What changed the number

Moving cost-gating OUT of the description. The first version led with "costs roughly 500k-1M tokens
and 1-2 hours, so it is reserved for explicit requests" -- a model reading that decides not to consult
at all, and so never discovers the cheap light tier. Cost belongs in the body, where it governs which
tier to run. Recall 30% -> 40% with tighter voting; the qualitative change is that explicit asks
("have codex do an adversarial review", "a second opinion from something that isn't claude") went
from intermittent to 3/3.

## Methodology note

At 1 run per query, two different descriptions produced identical aggregate scores from *disjoint*
sets of passes. Never compare descriptions at n=1.
