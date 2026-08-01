---
name: gauntlet
description: Build a feature through a bounded adversarial pipeline instead of writing it in one pass. An Opus subagent plans, Codex — a model from a different family — attacks the plan, an Opus subagent builds, Codex attacks the code, then tests run; rounds are capped and each ends in an explicit APPROVE/REVISE verdict. Use this skill whenever the user wants something built, implemented or refactored with subagents, multi-agent orchestration, a planner/builder/reviewer split, a critic loop, or "plan then critique then build then review" — and whenever they want a plan torn apart before any code is written. It also has a lighter mode, a single Codex pass over a finished diff, for when the user just wants a second opinion, an independent or adversarial check, or "another model to look at this". Trigger it even when the user never says "adversarial" or names Codex: "build this with subagents", "get someone independent to check it", "i don't trust this bit", "tear it apart", and /gauntlet all mean this skill — consult it first and let it pick the tier, rather than assuming it is too heavyweight.
---

# Gauntlet

**Codex, a model from a different family, tries to break your work.** In the light tier that is one
pass over a finished diff. In the full tier it is a bounded five-stage pipeline where an Opus subagent
plans and builds while Codex attacks first the plan, then the code.

The value is not "more review." It is *independent* review. A second Opus agent shares your priors and
tends to agree; a different model family disagrees in ways that surface real defects. In field use
this has caught design flaws a single-agent flow shipped, composition bugs where every individual
function was correct, and a test that passed against the very bug it was named for.

## Two tiers

The expensive part of this skill is the *planning loop*, not the review. Independent review is
cheap — one Codex call over a diff — and it is where most of the defects surface. So the tiers split
on cost, not on whether to review at all.

### Light — one review pass over a finished diff

Do the work yourself as normal. Then hand the diff to Codex once, act on anything real, and report in
a few lines. A few minutes, one `codex exec` call, no subagents, no planning loop.

This fits **any substantive code change**: something with branching logic, a constraint to enforce,
shared state, or a behaviour change other code depends on. It is proportionate enough that the
user's request to "fix X" is normally satisfied by fixing X *and* checking it — you are not embarking
on a separate project, you are not leaving the task, and it costs a rounding error next to the work
itself. Mention that you ran it and what it found; if it found nothing, one clause is enough.

Skip it for mechanical edits — renames, formatting, one-line changes, anything you can fully verify
in a couple of tool calls. Reviewing those wastes the call and trains the user to ignore the output.

### Full — the five-stage pipeline

Only on explicit request (`/gauntlet`, or the user asking for multi-agent or adversarial review). It
costs hours and a large token budget, so the user decides what earns it. If a task looks like it
would benefit and they have not asked, say so in one sentence and offer.

Good fits: a change where being wrong is expensive; subtle shared logic; anything where "it passes
tests" is not the same as "it is correct"; work where the *design* is the risky part, since that is
what the plan critique attacks and the light tier cannot.

Bad fits: anything the light tier covers. Reaching for the full pipeline when one review pass would
do is the most common way to make this skill unwelcome.

## The five stages

| # | Who | Does |
|---|---|---|
| 0 | You (main loop) | Write a data assessment grounding everyone in verified facts |
| 1 | Opus subagent | Write the plan |
| 2 | `gpt-5.6-sol` | Attack the plan → `APPROVE`/`REVISE`, **≤3 rounds** |
| 3 | Opus subagent | Build it |
| 4 | `gpt-5.6-terra` | Attack the code → `APPROVE`/`REVISE` |
| 5 | Opus subagent | Run tests; failures return to stage 3 |

Stage 0 is yours and is not optional — see below. Stages 2 and 4 use different Codex models on purpose: `sol` for design and reasoning, `terra` for code quality and optimization.

*(If those model IDs stop resolving, check `~/.codex/config.toml` for current names and update `references/operations.md`.)*

## What actually makes this work

These are the invariants. The stage list without them degrades into expensive theatre where agents politely agree with each other.

### Ground everyone in files, not relayed summaries

Before spawning anyone, read the relevant source yourself and write a **data assessment** to the scratchpad: verified facts with `file:line` citations, the existing primitives worth reusing, the traps, and — most valuable — a numbered list of **open questions the plan must answer explicitly**.

This matters because a subagent given a relayed summary inherits your errors with no way to detect them. A subagent given a file plus "verify every claim you rely on, you own correctness" will find your mistakes. In field use the planner corrected a load-bearing error in the assessment within its first pass.

Use `assets/data-assessment-template.md` as the skeleton.

### Demand an explicit verdict contract

Every reviewer returns exactly `VERDICT: APPROVE` or `VERDICT: REVISE`, plus numbered **material objections**. No "approve with changes" — if changes are needed, it is REVISE. Without a machine-checkable verdict you cannot decide whether to loop, and reviewers default to a wall of prose that reads as vaguely negative forever.

Also require a **"Verified-correct claims"** section listing what the reviewer independently checked and found true. This is what makes rounds converge: settled facts leave the table and round 2 attacks new ground instead of relitigating round 1.

### Give the builder and planner the right of rebuttal

Tell them explicitly: you may **REJECT** an objection with a specific technical rebuttal grounded in code you re-read. Rejecting is legitimate when the reviewer is wrong or is asking for scope the ticket does not want.

Without this the loop launders the reviewer's judgement into the plan — the planner capitulates on everything and you have simply replaced one agent's opinion with another's. The strongest rounds in practice are the ones where the planner accepted the *diagnosis* but implemented a tighter remedy than the one demanded.

### Hand reviewers a "do not report these" list

Known bugs, deliberate out-of-scope items, and settled design decisions go in the prompt as an explicit do-not-report list. Otherwise every round rediscovers the same pre-existing hazards and the loop never converges.

Say plainly which decisions are closed: *"Sections 0 and 0b record contested decisions that are settled. Judge the implementation, not the decision."*

### Adjudicate — never just relay

You are not a message bus. When a reviewer raises an objection, **verify it against the source yourself** before routing it onward, and tell the next agent what you concluded and why. Reviewers are confidently wrong often enough that forwarding unexamined objections wastes a full round.

When you agree, say so and add your own trace. When you doubt it, say that too and let the subagent argue back. Frame it as *"my read, to sharpen your thinking — argue back if I'm wrong."*

### Prove the test could fail — by the right instrument

A test that passes after your change proves nothing on its own. It has to be shown capable of
failing. There are two instruments and they are not interchangeable:

**Fixing a bug → red before green.** Run the new test against the unfixed code, watch it fail, then
fix and watch it pass. Report both numbers.

**Adding new behaviour → mutation.** There is no "before" for behaviour that never existed, so
red-before-green is unavailable and reaching for it produces theatre. Instead break the
implementation deliberately — remove the cap, flip the boundary, make the cache never expire — and
confirm **exactly the intended test dies and nothing else moves**. Then revert and re-verify.

Keep the two labelled. "Red after mutation" shows the implementation is load-bearing; "red before
green" shows a bug existed. Blurring them lets weak evidence claim more than it shows.

This is the highest-yield invariant here. In field use, one round's entire finding was a test that
**passed against its own bug** — a TTL test that stayed green against a cache with no expiry at all.
Nothing about running the suite would ever have surfaced it, because it is green either way.

**State the coverage honestly.** Mutation-testing every branch is often not worth it; say which
assertions you verified this way and which you did not, rather than implying uniform rigour. An agent
that reports "I mutation-tested the cache logic but not the other 31 tests, and that is where I'd aim
next" has given the next reviewer somewhere useful to look.

### Review the reasoning, not only the code

Ask the code reviewer to check the builder's *justification*, not just its diff. A fix can be correct while the argument for choosing it is false — that matters, because the next person to touch the code follows the comment's logic. In field use the reviewer confirmed a fix was right and proved the stated rationale for preferring it was wrong.

### Never run a reviewer concurrently with a baseline-establishing test agent

The test stage may `git stash` to measure a clean baseline. A reviewer reading the tree during that window reviews the *pre-change* code and returns a confident, well-cited, completely wrong report. Serialize those two.

More generally: agents that share mutable state — and the git working tree is mutable state — cannot run in parallel just because they both look read-only.

## Running the light tier

Finish and verify your change first — the reviewer should see the diff you would have shipped, not a
work in progress.

```bash
git diff > /tmp/review.diff        # plus any new untracked files
codex exec --model gpt-5.6-terra --sandbox workspace-write \
  --cd <repo-root> "$(cat <prompt-file>)" > /tmp/review.log 2>&1
```

Use `assets/code-review-prompt.md`, trimmed: you still want the verdict contract, the
"do not report these" list, and the instruction that finding nothing is a valid result — but drop the
plan references, since there is no plan. Point it at the diff and the new files.

Then **adjudicate rather than relay**: check each finding against the source yourself before acting.
Fix what is real, record what is adjacent, say plainly if you think the reviewer is wrong. Report in
a few lines — what it found, what you did. If it found nothing, one clause covers it.

The invariants below still apply to anything you change as a result, particularly proving a test
could fail.

## Running the full pipeline

Full invocation details, prompt templates, and verification patterns live in `references/operations.md`. Read it before stage 2. The essentials:

**Opus subagents** — `Agent` tool, `model: "opus"`. Resume the *same* agent across rounds (`SendMessage`) so it keeps its context and its own reasoning; spawning a fresh planner for round 2 throws away everything it learned.

**Codex reviewers** — write the prompt to a scratchpad file, then:

```bash
codex exec --model gpt-5.6-sol --sandbox workspace-write \
  --cd <repo-root> "$(cat <prompt-file>)" > <log> 2>&1
```

`--cd` into a trusted git directory is required or Codex refuses to start. Have it write its review to a scratchpad file and report only the verdict line — that keeps its transcript out of your context while you still read the full review deliberately.

Prompt templates: `assets/plan-critic-prompt.md` (stage 2), `assets/code-review-prompt.md` (stage 4).

## Loop control

- **Stage 2 caps at 3 rounds.** On the final round tell the planner it is final, so it lands decisions rather than deferring. After round 3 the plan proceeds regardless — an unbounded critic loop never terminates because there is always one more objection.
- **Stage 4→3 loops** until the reviewer approves or its remaining findings are marked non-material. Reviewers should be told explicitly that "this is correct and reasonably written" is a valid, useful result — otherwise they manufacture findings to appear rigorous.
- **Stage 5 failures** return to stage 3 with the failure output attached.

## Reporting back

The user did not watch any of this. They need the conclusion, not the transcript.

**Length is a real failure mode here, not a style preference.** A long pipeline produces a lot of
material and the pull is to prove the work happened by reproducing it. Resist that: the artifacts are
already on disk and can be read if wanted. A reader who cannot find the outcome in the first few
lines will not trust the rest, and the volume actively buries the one or two findings that justified
the cost. Reports from this skill have been called verbose in review — assume yours is too long and
cut it.

Track the five stages as tasks so the user can see where you are. The pipeline runs long and mostly
silent; without a visible spine, a stage-3 rebuild is indistinguishable from being stuck. Let the task
list carry the progress narration so your prose does not have to.

Between stages, a few lines: the verdict, what survived, what you concluded. Reviewers'
"verified-correct" findings are worth stating — they tell the user which risks were checked and
dismissed — but as a clause, not a section.

At the end, lead with the outcome, then cover these and stop:
- what changed, file by file — a table, not paragraphs
- the findings that actually changed the code, and what the reviewers explicitly cleared
- **anything deliberately not fixed**, and why
- any premise in the original request that turned out to be wrong

If a stage produced nothing interesting, say so in a clause and move on. "Terra approved with no
findings" is a complete report of that stage.

That last point matters more than it sounds. A ticket often points at the wrong file or misdiagnoses the cause; if the gauntlet establishes that, saying so plainly is one of its most valuable outputs — a reviewer expecting a change where none appears needs to know why.

## Anti-patterns

**Consensus theatre.** Two agents from the same family agreeing is not verification. If you cannot run a different model family, say the review is weaker rather than implying independent confirmation.

**Objection inflation.** A reviewer told to be thorough will invent findings. Tell it that finding nothing real is a valid outcome, and require every claim to cite a line it actually read.

**Scope creep through review.** Reviewers naturally propose adjacent fixes. Adjacent bugs get *recorded*, not fixed — unless the change itself would promote a latent bug into a live one, which is the one case where fixing it is in scope rather than gold-plating.

**Losing the thread.** Long pipelines drift. Re-read the user's original words before the final report and check you built what they asked for, not what the plan evolved into.
