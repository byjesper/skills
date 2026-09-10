---
name: unattended-run
description: Running several pieces of work unattended while the human is asleep or away — the decision contract that replaces asking, the state file that survives compaction, pre-registering the stop condition before the evidence arrives, and harvesting the traps a long run finds. Load BEFORE starting any multi-stage run nobody will be watching. Sits on top of orchestrate-implementors, which still governs spawning, worktrees, briefing and review.
---

# Running unattended

Load this only when the human has said some version of *"I am going to sleep, run
these without me"*.

Everything about spawning agents, cutting worktrees, briefing, reviewing and
integrating is unchanged and lives in `orchestrate-implementors`. Read that too.

What follows is only what changes **because nobody is watching**. That test is why
this file is short: most of what makes a run good is true either way, and belongs in
the other skill. In particular the verification discipline — *a green suite is not
evidence* — is deliberately **not** here. Filed under "overnight" it would read as a
special-occasion practice, and it is the first thing dropped on an ordinary day.

## The decision contract

Attended, an open question stops the work and goes to the human. Unattended,
stopping wastes the night: the run stalls at the first fork and they wake to
nothing.

**So: decide, proceed, and leave the trail — a comment on the relevant issue *and* a
pointer in the project's durable planning document — then raise it in chat when they
are back.** Both halves matter. A decision recorded only in chat is lost to the next
compaction; one recorded only in an issue is one nobody reads.

**Say in the artifact that the decision was yours.** Measured: an implementor's
terminal showed a plausible instruction sitting in its prompt line, which turned out
to be the CLI's ghost text rather than anything a human typed. Reaching the same
conclusion independently is fine; attributing it to them is not.
`orchestrate-implementors` has the method for telling the two apart.

**The licence is narrow.** It covers questions the human would answer in a sentence.
It does not cover the thing they said not to start, and it does not cover widening
scope because the run is going well.

## The state file

Keep one file — conventionally `scratchpad/RUN-STATE.md` — and open every compacted
turn by reading it. `RUN-STATE-template.md` beside this skill is a starting point.

**It is not a handoff document.** A handoff is written for a *reader*; a state file
is written for **the next turn of the same agent**, and only the second survives
being summarised. It carries the scope, the per-stage gate as a numbered checklist,
the standing decisions that must not be relitigated, what is in flight, and the
process ids. Not narrative, and not findings already acted on.

Update it at every boundary, not at the end. The run that needs it most is the one
that got compacted before it could write it.

**Instantiate the gate into it rather than describing it in a skill.** A project's
gate changes — one gained a step mid-run, after a merged slice that added a
dependency left the primary checkout's installed packages behind the lockfile and
presented as an impossible error in unrelated code. A checklist in a skill goes
stale; a checklist in the run's own state file is re-read every turn.

## Pre-register the stop condition

**Put this first when the human sets a limit.** It is the highest-value thing here.

When work is running long and a deadline is named, write the bar for "done" into the
state file **before the evidence arrives** — the specific, countable criteria that
decide whether it ships or is abandoned. Then judge against what you wrote.

The reason is not tidiness. **The agent judging the evidence is the one holding the
sunk cost**, at 1am, in front of a nearly-finished artifact, and it will find a
reading of "good enough" that lets it continue. Written in advance, the call takes a
minute. Measured: four criteria written 35 minutes before a deadline produced a clean
abandonment 20 minutes inside it, with the branch preserved and every finding
recorded — against a piece of work whose author had just reported "ship it".

**Tell the agent that an honest negative is a complete answer.** "This is not viable,
here is the elapsed time" is a result to act on, not a failure. Without that sentence
the failure mode is an agent waiting longer to avoid delivering bad news, and the
deadline passing with nothing at all. Told this, one implementor killed a measurement
that would not finish and reported numbers at a scope it could complete — which was
the useful answer.

**Abandon cleanly.** Do not merge, and do not delete the branch; name it in the issue
comment so the work resumes rather than restarts. Record what exists, what remains,
and why it stopped. Then close that agent and start the next piece — abandoned work
must not consume the rest of the night.

## Harvest the traps

A long unattended run finds tooling traps at a rate an attended one does not, because
nothing is interrupted and the same mistakes recur. In the moment the pull is always
to fix and move on, and unattended there is no conversation for the lesson to live in
— so it is lost unless written down deliberately.

**Record each one where its trigger is** — a path-scoped rules file for a path-scoped
trap, a skill for procedure, a reasoning document for a measurement — as its own
pure-docs commit, at the moment you find it. One overnight run produced six.

**A freshly recorded trap has a scope, and the reflex is to over-generalise it.**
Measured: minutes after recording that an agent waiting on a background monitor reads
as idle, the rule was applied to *every* agent, including a reviewer that blocks in
the foreground where an idle timer was correct — costing a wake-up the human had to
point out. When you have just written a rule, state where it does **not** apply.

## Reporting back

Lead with what shipped and what did not. Then, in order: anything you decided on
their behalf, anything you stopped, and the numbers that fell the wrong way.

**Report a measurement whichever way it falls.** A tightening that did nothing is
worth knowing, and an unattended run is exactly where the temptation to present a
flattering number is strongest, because nobody watched it being taken.
