# Gauntlet — operations reference

Mechanics, invocation details, and the traps. Read before stage 2.

## Contents
- [Scratchpad layout](#scratchpad-layout)
- [Invoking Codex](#invoking-codex)
- [Managing Opus subagents](#managing-opus-subagents)
- [Stage 5: judging test results](#stage-5-judging-test-results)
- [Failure modes](#failure-modes)

---

## Scratchpad layout

Everything agents share lives in the session scratchpad. Numbered so the sequence is legible when
you come back to it:

```
00_DATA_ASSESSMENT.md     you write, stage 0
01_PLAN.md                planner writes, revises IN PLACE across rounds
02_CRITIC_PROMPT_R1.md    you write
03_REVIEW_R1.md           Sol writes
04_CRITIC_PROMPT_R2.md    …
05_REVIEW_R2.md
06_CODE_REVIEW_PROMPT.md
07_CODE_REVIEW.md         Terra writes
```

The plan is revised **in place**, with each round prepending a `## 0. Review response (round N)`
section. One current document beats a pile of versions nobody can diff — and the response sections
give the next reviewer a record of what was contested and settled.

---

## Invoking Codex

```bash
SP=<scratchpad>
codex exec --model gpt-5.6-sol --sandbox workspace-write \
  --cd <repo-root> "$(cat $SP/02_CRITIC_PROMPT_R1.md)" \
  > $SP/sol_r1_raw.log 2>&1
echo "EXIT=$?"; tail -4 $SP/sol_r1_raw.log
```

- `--cd <repo-root>` is **required** — Codex refuses to start outside a trusted git directory
  ("Not inside a trusted directory and --skip-git-repo-check was not specified").
- `--sandbox workspace-write` lets it write its review file. It cannot reach the network.
- Redirect to a log and `tail` it. The prompt is long and the transcript is longer; you want the
  verdict line, then you read the review file deliberately with `Read`.
- Pass the prompt via `"$(cat file)"` rather than inline. Long prompts with backticks and quotes get
  mangled by the shell otherwise.
- These are **blocking foreground** calls. Set a generous timeout (10 min); a thorough review of a
  large plan takes several minutes.
- Verify a model ID before relying on it: `codex exec --model <id> --sandbox read-only "Reply with
  exactly: OK" < /dev/null`. Current IDs live in `~/.codex/config.toml`.

---

## Managing Opus subagents

Spawn with the `Agent` tool, `model: "opus"`.

**Resume the same agent across rounds** via `SendMessage` with its agent ID — it keeps its context
and its own reasoning. Spawning a fresh planner for round 2 throws away everything it learned and
guarantees it re-derives (or contradicts) its earlier decisions.

**When routing a review to a subagent**, include:
1. Which round this is and what the cap is
2. The review file path — not a paraphrase
3. Which objections are **already adjudicated resolved**, so it leaves them alone
4. Your own verification of the contested ones, framed as *"my read, to sharpen your thinking —
   argue back if I'm wrong"*
5. Explicit permission to REJECT with a grounded rebuttal
6. What to produce and where

**Builder prompts** should state, in this order: read the plan in full; settled decisions are not
open to revisit; implement what the plan says rather than what you would have planned; report any
deviation explicitly; the out-of-scope list is binding; match the surrounding code's voice; run the
lint/type gate yourself.

Ask the builder to report: file-by-file changes, `git diff --stat`, deviations (or "no deviations"),
gate output, and — worth its own line — **anything it noticed that the reviewers and planner all
missed**. Builders reading code closely find things reviewers reasoning abstractly do not.

---

## Stage 5: judging test results

**Establish a baseline.** In a repo with pre-existing failures, a raw failure count is meaningless.
Stash only the changed tracked files (not `-u`, which sweeps unrelated untracked artifacts into the
stash), re-run, and compare.

**Compare failing node-ID sets, not counts.** Equal counts can hide one test newly failing while
another newly passes. The signal is *zero symmetric difference* in both directions.

**Run a collection check first** — `--collect-only` catches import-time breakage without needing
infrastructure, and it is fast. In repos with rotted test files, collection errors can abort the
entire run before anything executes; you may need `--continue-on-collection-errors`.

**Separate infra noise from regressions explicitly.** Group pre-existing failures by cause and
characterize them ("13 × missing tenant fixture data") rather than listing them. Never report an
environmental failure as a regression, and say plainly if the suite cannot meaningfully run here.

**Verify the formatter did not touch unrelated files.** Some repos have committed-unformatted files
that a format step reflows and the lint gate never inspects. Check `git status` and revert strays.

---

## Failure modes

**Codex refuses to start.** Missing `--cd`, or not a git repo. See above.

**A reviewer reviews the wrong code.** The test stage stashes to build a baseline; a reviewer running
concurrently reads the stashed tree and returns a confident, well-cited report about code you are not
shipping. Serialize them. Generalize the lesson: the git working tree is mutable state, so two
"read-only" agents can still conflict.

**The lint/type gate does not cover tests.** Many projects run their type checker on `src` only. Lint
new test files explicitly.

**A blocked git command.** Destructive operations like `git checkout HEAD -- <file>` may be denied.
Prefer staging-only approaches that never touch the working tree: build the intermediate file in the
scratchpad, `git diff --no-index` it against `git show HEAD:<path>`, rewrite the patch headers to the
repo-relative path, and `git apply --cached`. This also gives you a non-interactive `git add -p`.

**The loop will not converge.** Almost always a missing do-not-report list, or a missing
"verified-correct claims" section letting each round relitigate the last. Add both.

**The planner capitulates on everything.** It was not told it may reject. Add the right of rebuttal
and re-run the round.

**Reviewers manufacture findings.** They were told to be thorough without being told that finding
nothing is acceptable. Add that sentence, and require every claim to cite a line actually read.
