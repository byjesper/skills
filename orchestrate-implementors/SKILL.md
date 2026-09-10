---
name: orchestrate-implementors
description: Running work as an orchestrator plus implementing agents — spawning subagents through the Solo MCP server, provisioning parallel git worktrees, briefing, adversarially reviewing and integrating. Load BEFORE spawning any subagent of any kind or creating any worktree; the setup mistakes here silently corrupt a run and the affected agent reports success.
---

# Orchestrated implementation

For work run as **orchestrator/integrator + implementors**: one session briefing
implementing agents, reviewing their diffs, and merging. The orchestrator writes no
feature code; implementors merge nothing.

Almost every rule here was paid for. Where a claim carries a date it was measured,
and the measurement is kept so the next agent does not retake it. Where a rule says
a failure is *silent*, that is the point: these mistakes do not produce errors, they
produce confident wrong results.

Examples are PHP/Composer because that is where they were measured. The rules are
not language-specific; translate the tooling and keep the shape.

## Choosing the agents

- **Implementor: a strong model at medium effort.** Decided after a three-way
  comparison on real slices under identical briefs. The strongest of the three was
  also the fastest, and — decisively — it surfaced its judgment calls instead of
  deciding silently. Prefer the model that tells you what it chose.
- **The reviewer is a different model from the implementor, on purpose.** It
  re-derives from the vendored source rather than from the implementor's reasoning.
  That difference is what found a build-time failure appearing only under a
  production install, which the implementor's own model had reasoned past twice.
- **Set effort with the launch flag, not the in-session command.** Measured twice
  independently on Claude Code v2.1.226: `--effort` applies to that session and does
  not persist, while the in-session `/effort` saves itself as the default for new
  sessions. The flag is the only form that leaves no state for the next spawn to
  inherit.
- **Grunt work goes to cheap models.** Research sweeps, bulk reading and inventories
  run as background agents on a small model that report facts back. The
  orchestrator's context is for review, verification, integration and synthesis.

## Spawning

- **Spawn every subagent through the Solo MCP server when it is available** —
  `spawn_agent`, prompts via `send_input`, wake-ups via timers rather than polling.
  **Every** means every: a reviewer, a one-off researcher and a throwaway
  grep-runner are spawned the way an implementor is. A built-in subagent tool is
  what you reach for when Solo is *unavailable*, never when the work feels too small
  to warrant it — a rule that said "implementors" was read narrowly and a reviewer
  went out through the built-in tool.

  The reason is addressability. A Solo-spawned agent is a process with an id that
  `list_processes`, `get_process_output` and `close_process` reach from any session
  holding the Solo tools; a built-in subagent is reachable only from the session
  that spawned it. Which an agent turns out to need is not knowable when it is
  spawned, which is why the choice is not made per agent.

- **`spawn_agent` has no working-directory parameter, and does not need one to put
  an implementor in a worktree.** "It can't open in worktrees" is half-right and
  wholly misleading: the session opens in the project path and then moves itself.
  Measured with five agents at once — register the worktree first, spawn, and make
  the brief's first action the worktree-entering tool with an explicit path. The
  isolation is real: one agent's compound shell command was refused by its own
  harness for being unverifiable as staying inside its worktree.

- **A spawned agent may start in manual mode — verify before briefing it.**
  Measured: two spawned sessions both came up paused, and one sat at a permission
  prompt until a human noticed. **A stalled agent is indistinguishable from a
  working one**, so an idle wake-up fires on the stall and reads as a result. Read
  the process output and confirm the footer shows auto mode before sending the
  brief; if it does not, send shift+tab (`send_input` bytes `[27,91,90]`) — the
  cycle is manual → accept-edits → plan → auto, so three presses — and re-read.

- **Send those three as three separate calls.** Measured: one call carrying the
  bytes three times advanced the mode by a single step, landing on accept-edits.
  Sending them together and trusting the count leaves the agent one press short of
  auto mode, which is the stalled state the check above exists to catch.

- **Close an implementor the moment its work is merged.** A finished implementor is
  a running session holding a whole slice in context: it costs, it is a live target
  for a stray keystroke, and its presence in `list_processes` invites a later agent
  to reuse it instead of spawning the fresh one the next slice is owed. A standing
  reviewer is the exception; it is meant to outlive the work.

## Briefing

- **Any prompt longer than one short sentence goes in a file, with a one-line
  `send_input` naming its absolute path.** Every prompt to every agent — brief,
  review request, follow-up — not only the first. Sent inline, a long prompt arrives
  truncated and a multi-line one submits at every newline. Either way the agent acts
  on a fragment and nothing in its output says so.

- **Point an implementor at the rules file its own harness loads, and never at
  both.** Where a tool generates per-harness rules files from shared sources, those
  files are usually byte-identical — measured at 51,770 bytes each, zero diff lines.
  Each harness loads its own automatically. A brief opening "read `<the other
  one>` first" buys a duplicate copy of ~13k tokens and nothing else; measured on
  a run where the agent complied without noticing it already had the content. Say
  "the project rules your harness has already loaded" rather than naming a file.

- **State the comment calibration as a ceiling and name the floor.** An agent told
  only a ceiling fills it. Give both: most declarations should carry no comment at
  all, and the ceiling is what a class, a guard, an exception or a non-obvious test
  *may* earn. Point at two files as voice references rather than describing the
  voice.

- **Demand a mutation list.** For each significant guard, the production change the
  implementor made to watch a named test go red. Say in the brief that you will
  re-run one, because that is what makes the list real.

## The integrator's job

The orchestrator reviews every diff, independently re-runs at least one claimed
mutation, runs the merge gate itself, merges, ticks the acceptance criteria, and
writes the changelog entry in the same merge window. Implementors do none of those.

- **Implementor judgment calls go to the human in chat, never into issue comments.**
  The comment trail is an archive; nobody re-reads it. A decision parked there is a
  decision nobody made. Confirmed rules land in the project's rules files or an ADR;
  the issue gets only the completion record.

- **Do not record an implementor's reported "lesson learned" without measuring it.**
  One reported that a test runner's `--filter` with quoted spaces matched nothing
  and *"reports passed with zero tests run"*. Measured: spaces match fine and exit 0,
  the underscored form is what fails, and a zero-match run exits 1 and reports
  failed — wrong in both directions. Whatever cost that agent a cycle is still
  unexplained. **A false rule in a guidelines file is worse than no rule, because
  the next agent obeys it.**

## A green suite is not evidence

This is the whole of the integrator's job, and it holds whether or not anyone is
watching.

- **Reproduce the failure before you trust the fix.** A check that only exercises
  the environment where the thing already worked proves nothing. Three failures in
  one night shared that shape: a build-cache reproduction that passed because dev
  dependencies were installed, an environment test that passed because the framework
  reads a dotfile rather than the shell, and a config fix that passed locally
  because the developer's environment already had the value — then killed CI at the
  install step.
- **Re-run the implementor's load-bearing claim yourself**, preferring one that
  initially surprised you. Not the whole list; the one the decision rests on.
- **Caches make a dry run lie.** Measured: after editing a static-analysis tool's
  config, its `--dry-run` reported zero changes from cache, which would have had a
  *correct* implementor claim reported as wrong. The shape recurs in any tool that
  caches by file hash. Clear the cache when you are testing what a rule does.
- **A figure nobody measured is the figure that is wrong.** In one report every
  number was measured except one written with a tilde, and that one was out by 8x —
  and it was the one a resume estimate depended on. Treat hedged precision as
  unmeasured.
- **Look at the running app when a surface changed.** Two defects in one milestone
  came only from the browser, with the suite green — a refusal naming the wrong
  month, and a heading that jumped between pages. Nothing else catches a `0×0`
  rendering regression.
- **Verify the environment before blaming the code.** A merged branch that added a
  dependency leaves the primary checkout's installed packages behind the lockfile,
  because the gate ran in a worktree and pulling does not install. It presents as an
  impossible error in unrelated code.

## Parallel agents in git worktrees

These setup mistakes silently corrupt a run. Get them right before the agent does
any work, and treat the orchestrator's post-merge run as the real gate.

- **Do not provision the next slice's worktree until the current one has been
  reviewed, corrected, and is green in CI** — not the local suite, CI. This holds
  even when the next slice is formally unblocked. Measured: a worktree cut while
  review was still running went stale within minutes, and the review then found a
  real defect in exactly the code the next slice builds on. Every merge-forward
  after an early cut is work the early cut created, and an implementor that had
  started coding would have built on the defect. What a "blocked by" field permits
  is about *dependency*, not *readiness*.

- **Wait for the whole review before correcting anything.** Fixing the first finding
  while the reviewer is still working re-opens the same files for the rest and
  invalidates its picture of the code mid-run. Arm an idle timer on the reviewer and
  do work that touches nothing under review.

- **Never symlink the dependency directory into a worktree — run a real install.**
  A generated autoloader computes its base directory from the file's *real* path, so
  through a symlink it resolves to the **primary** checkout. Namespaces then map to
  the primary source: the worktree's new classes never load and its edits are
  ignored at runtime. Tests run against the primary code and go falsely green, so
  the agent's self-verification is worthless. **Clone-copy instead** (`cp -Rc`, which
  is copy-on-write on APFS and costs seconds), then run the installer, which finds
  everything present and only regenerates the autoloader against the worktree's own
  path. Then verify: a class must resolve *inside* the worktree. Directories with no
  equivalent base-path assumption may be symlinked.

- **A fresh worktree may be based behind the trunk — merge it in first.** Work
  merged after the worktree was created is invisible to it, and the branch conflicts
  hard at integration.

- **Build artifacts are per-worktree, and the tests they break look unrelated.**
  A git-ignored build directory is absent in a fresh worktree, so every test
  fetching a full page 500s on a missing-manifest exception while component-level
  tests pass. Measured: 13 component tests passed while 4 of 7 page tests failed —
  so the failing set reads as an arbitrary scatter rather than as a missing build,
  and an agent loses time before finding the exception text.

- **Provision worktrees where the harness expects them.** Claude Code's worktree
  tool prompts for approval on a path outside `.claude/worktrees/`, **even in auto
  mode**. Measured: two implementors spawned into an arbitrary sibling directory
  stopped at that prompt on their first action and stayed there — which looks
  exactly like an agent that is working.

- **The orchestrator's own shell drifts into worktrees.** A `cd <worktree> && …`
  inside one call re-roots the session, and a later `cd <primary>` does not reliably
  persist. Measured: an orchestrator created a fix branch believing it was in the
  primary — the checkout ran in an implementor's worktree, switched that agent's
  branch under it mid-slice, and appended a test to the wrong tree's file. Use
  absolute paths, or a subshell `(cd $W && …)` which leaves the parent's directory
  alone, and check `pwd` before any relative-path operation. Implementor sessions
  are protected by their own harness isolation; the orchestrator's is not.

## Reading an agent's terminal

- **Unsubmitted text in an agent *you* spawned is ghost text.** The CLI's own
  next-prompt suggestion renders identically to a typed draft, and stopping to ask
  about each one costs a round trip for nothing.

- **In a session you did not spawn, the rendered output cannot distinguish the
  two.** The CLI dims its suggestion, but neither the rendered nor the raw output
  carries that attribute through — measured, the raw call returned the text with its
  SGR sequences stripped.

  **There is a safe way to find out.** Send your own message with `submit: false`,
  then read the prompt line. Ghost text is redrawn, so it vanishes the moment real
  input arrives and you see only your own text; typed input is buffered, so yours
  arrives appended and you see both. Then send `[13]` to submit, or clear it if what
  you found belongs to the human. Ctrl+U (`[21]`) does **not** clear ghost text — no
  effect at all is itself weak evidence that it is a suggestion.

  **Never submit text you did not write.** If it turns out to be a person's draft,
  surface it and ask. If you must decide without them, decide on the merits and say
  in your message that the decision was yours.

## Wake-ups

- **A long timer body arrives truncated to its tail.** Measured across three
  wake-ups: bodies of a few hundred words were delivered as their final sentence or
  two, so what arrived was a throwaway clause (`If still working, set another
  timer.`) while the actual task at the head was gone. A wake-up that says only its
  own footnote is worse than none, because it reads like a complete instruction.
  Write one or two sentences naming a file that holds the detail.

- **An agent waiting on its own background monitor reads as idle.** Idle means "this
  agent's turn ended", not "its work finished". So an idle-triggered timer returns
  *already satisfied* and schedules nothing, and an orchestrator treating that as its
  wake-up never wakes. Use a plain delay timer for any agent running work in the
  background, and keep idle timers for agents that block in the foreground.
  **Whichever you use, read the return value** — "already satisfied" means no timer
  exists.

## Agents compact each other

A session cannot compact itself: `/compact` typed into your own terminal is a
message to you, not a command the harness runs on you. So a long run ends in a
context wall unless another agent drives it — and the orchestrator is the session
that most needs it, because it accumulates every diff, every review and every merge.

Give the duty to the standing reviewer: already spawned, long-lived, and idle
between slices, which is exactly when the orchestrator is fullest. At each boundary:

1. `send_input` to the orchestrator: `/handoff`, so it writes a summary while it
   still has the context to write one.
2. `send_input`: `/compact <directions>`, with that summary in the directions.
3. A **second, separate** `send_input`: `Continue — read <state file> first.`

The third cannot be folded into the second. A compaction consumes the message that
triggers it, so a `/compact` carrying "and then continue" leaves the orchestrator
summarised and silent, waiting for a turn that never comes.

**Write the directions explicitly, naming what to keep and what to drop.** Left to
its own judgment the summariser keeps the narrative and drops the state — the
findings of a finished review survive while the sequence and the process ids do not.
Keep the scope, the gate, the standing decisions, what is in flight and which agents
hold which ids; drop findings already acted on and the detail of work already merged.

**Compact a standing reviewer too, during dead time** — while waiting on CI, never
when a brief is already waiting — and direct it the same way: keep the role, the
method and the output shape, drop the findings already delivered. For a reviewer the
findings are the one part already spent, and they are exactly what an undirected
summariser keeps.

**The permission this needs may not be in the repository.** Local settings files are
often git-ignored, so an agent allowed to call `send_input` in one checkout is not in
the next. Grant the reviewer its Solo permissions **before** the run starts, not at
the first boundary — measured, where the duty failed its first rehearsal for exactly
this and a human had to intervene. A reviewer blocked on a permission prompt is
indistinguishable from one that is thinking.

**Keep a state file the compacted session reads on its first turn.** The handoff is
written for a reader; the state file is written for the next turn of the same agent,
and only the second survives being summarised. See the `unattended-run` skill.
