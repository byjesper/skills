# <project> unattended run — state of play

**Read this first after any compaction.** <Human> is asleep. Keep going.

Copy this to `scratchpad/RUN-STATE.md` at the start of the run and keep it current at
every boundary — not at the end. Delete the guidance in angle brackets.

Orchestrator = process **<id>** (me). Adversarial reviewer = **<id>**, which outlives
the stages and is not closed. Implementors are spawned fresh per stage and closed on
merge.

## Scope

Finish **<a → b → c>**. **STOP before <x>** — <why it is the human's to start>.

<One line per stage: issue number, state, worktree, branch, brief path. Mark the one
in flight. Reduce merged ones to a single line with their merge commit.>

## The gate, every stage without exception

<Instantiate the project's own gate here, so it is re-read every turn rather than
remembered. This example is from a PHP/Composer project; replace the tooling.>

1. Cut the worktree from the trunk **only after the previous stage is merged with CI
   green**. Copy the environment file, clone-copy dependencies, then run the
   installer against the worktree path. Verify a class resolves *inside* the
   worktree, not the primary.
2. Spawn a fresh implementor; confirm auto mode **before** briefing; brief via a
   one-line message pointing at a file.
3. On its report: review the diff yourself, independently re-run at least one claimed
   mutation, run the gate yourself, and drive the app in a browser if a surface
   changed. Never accept a green suite as evidence.
4. Adversarial review: brief to a file, arm a wake-up, and **wait for the whole
   review before correcting anything**.
5. Fix in one pass. Re-run the gate. Push, open the PR, write the changelog entry
   yourself, poll checks until green, never auto-merge, then merge — never squash if
   later stages branch off.
6. Comment the acceptance criteria on the issue. Sync the trunk, **and reinstall
   dependencies in the primary checkout** — a merged stage that added one leaves the
   primary behind the lockfile, and nothing else reveals it.
7. Record any measurement the run is tracking.
8. Close the implementor and remove its worktree. The reviewer survives.
9. Compact, then cut the next worktree.

## Standing decisions — do not relitigate

<What is already settled and a compacted agent will otherwise reopen: merge policy,
what the gate is and how long it may take, conventions argued once. Lines, not
paragraphs.>

## Stop condition for <the stage at risk>

<Written BEFORE the evidence arrives, whenever a limit is set. The specific countable
criteria that mean "done". The deadline. That a hedge, an estimate, or an unfinished
measurement is NOT a clear result — and that an honest "not viable, here is the
elapsed time" IS. Then how to abandon cleanly: do not merge, do not delete the
branch, name it in the issue.>

## Because they are away

Anything that would normally stop and ask: **decide, proceed, and leave the trail** —
a comment on the relevant issue **and** a pointer in the durable planning document —
then raise it in chat when they are back. Say the decision was yours.

## Traps found this run

<Append as found, with the commit that recorded each. If this section is empty at the
end of a long run, the traps were not harvested — they were not absent.>
