# Iteration 1 — findings for the next revision

## Applied after Kiran's review (iteration 1 → 2)

His feedback, and what changed:

| Run | Feedback | Change |
|---|---|---|
| eval-0 with-skill | "very verbose" | New brevity section in *Reporting back*: length named as a failure mode, table not paragraphs, "assume yours is too long and cut it", progress narration delegated to the task list |
| eval-0 baseline | "some more planning would be helpful" | Confirms the planning stage earns its place — kept |
| eval-1 with-skill | "didn't run the whole process" ✓ | Kept; the bad-fit criteria work |
| eval-2 with-skill | "I would like some adversarial review for the code" | **Structural: added a LIGHT tier.** One Codex pass over a finished diff, minutes, no planning loop. Full pipeline still explicit-only. |

The eval-2 note overturned a pre-test decision (explicit-invocation-only). The skill was binary —
100-minute pipeline or nothing — and the wanted behaviour was in between. Tiers now split on **cost**,
not on whether to review.

Also applied from S1–S3 below: the "red before green" invariant now names **two** instruments and
says when each applies, plus a requirement to state mutation coverage honestly.

**One thing left undone, needs Kiran's decision:** the first draft of the description said the light
tier should run *"unprompted, no permission needed"*. That edit was blocked by the permission
classifier — correctly, since it amounts to a skill granting itself standing authority to invoke an
external CLI. The current wording describes when the tier *fits* without asserting it needs no
permission. If Kiran wants it to genuinely run without a prompt each time, that is a settings-level
permission rule for `codex exec`, which is his to add, not the skill's to claim.

---

## Harness defects (mine, fix before trusting a quality comparison)

**H1. Eval arms were not isolated — via two separate channels.** Every run got a private repo copy,
but nothing stopped one arm *reading* another. The instruction "do all work inside this directory
only" governed writes, not reads.

- *Sibling workspace:* the eval-0 with-skill planner cited the baseline arm's exact identifiers
  (`_CACHE`, `_fetch_promo_catalog`, `clear_promo_catalog_cache`) and observed its `promos.py:75`
  mid-mutation-test.
- *Shared session scratchpad:* worse, and the one I initially missed. The eval-0 orchestrator found a
  tree there holding **a previous run's complete solution to this same ticket**. Every agent in a
  session shares that root, so a prior run's finished work is readable by a later run that is
  supposed to be solving it fresh. It silently handed one agent a "before" state that was already
  fixed.

→ **Eval-0's head-to-head quality comparison is void.** Structural assertions still hold.
→ Fix: give each arm a temp directory with no sibling reachable, **and** a private scratchpad root.
  Clear or namespace the session scratchpad between runs of the same task.

**H2. The scratch repo's CLAUDE.md contradicts eval-1's task.** It lists "f-string logging" as a
convention while eval-1 asks to convert an f-string logger to `%`-style. Both arms caught the clash,
so eval-1 partly measures "does the agent read CLAUDE.md" rather than "does it decline to
over-engineer." Remove the contradicting line, or make the conflict deliberate and assert on it.

**H3. eval-1 and eval-2 have very strong baselines**, so they can only show the skill does no harm,
not that it helps. That is the correct shape for a negative test, but it should be stated rather
than read as a weak result.

## Skill defects (found by running it)

**S1. "Red before green" is the wrong instrument for new behaviour, and the skill overstates it.**
SKILL.md says regression tests must be demonstrated failing before the fix. That holds for a *bug*,
where a pre-fix red run is available. For behaviour that never existed there is no "before" — the
correct instrument is **mutation testing**: break the implementation, confirm exactly the intended
test dies, revert.

The builder in eval-0 drew this distinction unprompted and precisely: *"these are red-after-mutation,
not red-before-green — they demonstrate the implementation is load-bearing, not that a bug existed."*
It also insisted the two not be blurred, since round 1's lesson was evidence claiming more than it
shows.

→ Revise the invariant to name both instruments and say when each applies. Require the report to
state which one was used, so "the test passed after my fix" can never masquerade as evidence.

**S2. Mutation testing deserves promotion from implicit to explicit.** Both the eval-0 baseline and
the with-skill builder reached for it independently, and it is what caught the strongest finding of
the whole run (a TTL test that passed against a never-expiring cache). The skill currently never
names it.

→ Add it to the invariants: after a reviewer approves, mutate each load-bearing branch and confirm a
specific named test dies. Cheap, mechanical, and it catches the failure mode least visible by running
the suite — a test that is green whether or not the bug is present.

**S3. Coverage of the mutation pass should be stated, not assumed.** The eval-0 planner mutation-tested
the TTL logic and then said plainly it had *not* done so for the other 31 tests, logging that as where
it would aim next. That self-reported blind spot is exactly the behaviour to encourage.

→ Ask agents to report which assertions were mutation-verified and which were not, rather than
implying uniform rigour.

## What worked (keep)

- The prescribed scratchpad chain appeared exactly as specified:
  `00_DATA_ASSESSMENT` → `01_PLAN` → `02_CRITIC_PROMPT_R1` → `03_REVIEW_R1`.
- The right of rebuttal produced a real, well-argued **REJECT** (a sleep-spy suggestion, refused
  because the fake clock's no-op `sleep` would make the spy count zero in exactly the tests that hit
  the store — an argument the reviewer had not anticipated).
- The planner reproduced a reviewer's objection rather than accepting it on trust, by building the
  broken implementation and observing its own tests pass against it.
