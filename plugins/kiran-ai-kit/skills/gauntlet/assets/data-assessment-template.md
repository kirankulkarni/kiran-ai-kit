# <TICKET> — Data Assessment

> Stage 0 skeleton. You (the main loop) write this **before** spawning anyone, by reading the source
> yourself. Every claim carries a `file:line`. Delete these instruction blocks as you fill it in.
>
> Why this exists: a subagent handed a relayed summary inherits your errors with no way to detect
> them. A subagent handed this file plus "verify every claim you rely on — you own correctness"
> will find your mistakes. Expect it to correct you, and treat that as the system working.

**Task, verbatim from the user:**
> <paste the request exactly — not your paraphrase of it>

Repo: `<path>`
Branch: `<branch>` (branched from `<base>`)

---

## 1. The files named in the request

For each: what it does, the call path in and out, and the specific functions that matter.
Cite line numbers. Note anything the request assumes that is **not** actually true — a request often
points at the wrong file or misdiagnoses the cause, and catching that here saves an entire cycle.

## 2. What is genuinely unverifiable from the repo

Tenant data, production config, external API payloads, anything living in a database.

State the evidence for and against your best guess, then say plainly: **the plan must not assume
this.** Design defensively around it instead. A plan built on a confident guess about data you
cannot see is the most expensive kind of wrong.

## 3. The gap, stated precisely

What is actually broken, in a few sentences, in terms of observable behaviour rather than code
aesthetics. If there are two independent gaps, number them — plans blur them together otherwise.

## 4. Existing primitives to reuse (do not reinvent)

Decorators, helpers, base classes, established patterns, and **sibling precedents** — places the
codebase already solved this shape of problem. Include the failure behaviour of each (does it raise?
swallow? return a sentinel?), because plans routinely get this wrong.

## 5. Tempting reuse that is actually a trap

The near-miss that looks perfect and is not — often something whose docstring or comment explicitly
invites the use you are about to reject. Say exactly why it fails here. Without this section a
planner will find it independently and burn a round proposing it.

## 6. Constraints the plan must honour

Repo conventions from `CLAUDE.md` files, layering rules, lint/type gates, the exact test invocation,
and any known trap in the local toolchain. Be specific: "run `X` not `Y`, because `Y` silently does `Z`."

## 7. Open questions the plan MUST answer explicitly

The highest-value section. Number them. Each should be a real fork where different answers produce
materially different code — not questions with an obvious default.

Force a decision, not a survey. Phrase them so a hedge is visibly a non-answer:

1. **Where does X live?** Consider that <constraint A> and <constraint B> pull in opposite directions.
2. **What does "<vague phrase from the request>" mean concretely?** Enumerate the readings and pick one.
3. **How do <new behaviour> and <existing behaviour> interact?** Define the ordering precisely.
4. **What is the blast radius**, and who else is affected?
5. **Test strategy**, given <local constraint about the test suite>.

Require the plan to answer each as `Decision:` / `Why:` / `Rejected alternative: … because …`.
The rejected-alternative line is what stops a plan from looking considered while actually being
the first idea that occurred to it.
