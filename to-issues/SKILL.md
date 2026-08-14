---
name: to-issues
description: Break a plan, spec, or PRD into independently-grabbable issues on the project issue tracker using tracer-bullet vertical slices.
disable-model-invocation: true
---

# To Issues

Break a plan into independently-grabbable issues using vertical slices (tracer bullets).

The issue tracker and triage label vocabulary should have been provided to you — run `/setup-matt-pocock-skills` if not.

## Process

### 1. Gather context

Work from whatever is already in the conversation context. If the user passes an issue reference (issue number, URL, or path) as an argument, fetch it from the issue tracker and read its full body and comments.

### 2. Explore the codebase (optional)

If you have not already explored the codebase, do so to understand the current state of the code. Issue titles and descriptions should use the project's domain glossary vocabulary, and respect ADRs in the area you're touching.

Look for opportunities to prefactor the code to make the implementation easier. "Make the change easy, then make the easy change."

### 3. Draft vertical slices

Break the plan into **tracer bullet** issues. Each issue is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

<vertical-slice-rules>

- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Any prefactoring should be done first
- **Design for parallelism.** Shape the slices — and the dependency graph — so that as many as possible can be built concurrently by separate agents in isolated git worktrees. Prefer slices that touch disjoint files/directories (so parallel worktrees don't conflict at merge), isolate the unavoidable bottleneck (usually a foundation slice everything depends on) into its own issue so it clears the critical path fastest, and favour framework auto-discovery over a shared registry file that every slice would have to edit (a shared file serialises otherwise-parallel work). Keep the true critical path (the strict blocked-by chain) as short as possible.

</vertical-slice-rules>

### 4. Quiz the user

Present the proposed breakdown as a numbered list. For each slice, show:

- **Title**: short descriptive name
- **Blocked by**: which other slices (if any) must complete first
- **User stories covered**: which user stories this addresses (if the source material has them)

Ask the user:

- Does the granularity feel right? (too coarse / too fine)
- Are the dependency relationships correct?
- Should any slices be merged or split further?
- Does the parallelism shape look right — which slices run concurrently in worktrees, and is the critical path as short as it can be?

Iterate until the user approves the breakdown.

### 5. Publish the issues to the issue tracker

For each approved slice, publish a new issue to the issue tracker. Use the issue body template below. These issues are considered ready for AFK agents, so publish them with the correct triage label unless instructed otherwise.

**Also set the issue type and add it to the project board.** Trackers handle these two unevenly, so do not assume one command covers both. Add the issue to the board at creation time where the tracker allows it rather than editing it in afterwards; an issue that never reaches the board is invisible to whoever works from it.

The tracker's mechanics — create and edit commands, the board's name, its custom fields and their vocabulary — are repo-specific. Read them from the tracker doc named in the project's agent guidelines (conventionally `docs/agents/issue-tracker.md`, written by `/setup-matt-pocock-skills`). Never assume a type, field, or status name exists; read the project's enabled values first.

**Choosing the type.** Whatever the tracker calls them, choose by what the issue *is*, not by which layer it touches. The common set:

- **Feature** — new user- or operator-facing capability
- **Bug** — something behaves wrong today
- **Task** — internal work: refactors, tooling, measurement, regeneration, research spikes, release chores

A slice inherits nothing from its parent. A `Feature` PRD usually decomposes into mostly `Task` slices, and that is correct — the parent is the capability, the children are the work.

**Board fields.** If the board defines custom fields (status, area, priority, blocked-by), set them for each new issue too — a card with a blank status is a card nobody can plan around. In particular, a slice with open blockers must **not** land in a "ready/pickable" status, or an AFK agent will take work that cannot yet be done.

Verify both the board membership and the type stuck before reporting the issue published.

**Publish issues in dependency order** (blockers first) so you can reference real issue identifiers in the "Blocked by" field. Every slice's "Blocked by" section must name its real blockers as `Blocked by #nn` (one per line, with a short reason), or "None - can start immediately". These cross-references are what let a reader — and a parallel-agent orchestrator — see the dependency graph at a glance.

**Link each new issue to its parent** (when the source was an existing parent issue, e.g. a PRD) using the tracker's native sub-issue or child relationship where it has one, not merely a `#nn` mention in the body.

<github-example>
Where the tracker is GitHub, driven by `gh`:

- **Project** — `gh issue create --project "<title>"` resolves org- and user-owned projects by title.
- **Type** — there is **no** `--type` flag on `gh issue create` or `gh issue edit` (checked at gh 2.92). Set it after creation over the REST API, which takes the type's **name**:

  ```
  gh api --method PATCH repos/<owner>/<repo>/issues/<n> -f type=Task
  ```

  The valid names are the org's enabled types — read them, don't guess:

  ```
  gh api graphql -f query='query{organization(login:"<org>"){issueTypes(first:20){nodes{name isEnabled}}}}'
  ```

  The default set is **Bug** / **Feature** / **Task**.

- **Verify** — `gh issue view <n> --json number,title,projectItems`, plus the `type` from the REST read.
- **Sub-issues** — the REST API keys on the child's internal **id**, not its issue number:

  ```
  # after `gh issue create` returns the child's URL/number:
  child_id=$(gh api "repos/<owner>/<repo>/issues/<child_number>" --jq '.id')
  gh api --method POST "repos/<owner>/<repo>/issues/<parent_number>/sub_issues" -F sub_issue_id="$child_id"
  ```

  Verify afterwards with `gh api repos/<owner>/<repo>/issues/<parent_number>/sub_issues --jq '.[].number'`.

</github-example>

<issue-template>
## Parent

A reference to the parent issue on the issue tracker (if the source was an existing issue, otherwise omit this section).

## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

Avoid specific file paths or code snippets — they go stale fast. Exception: if a prototype produced a snippet that encodes a decision more precisely than prose can (state machine, reducer, schema, type shape), inline it here and note briefly that it came from a prototype. Trim to the decision-rich parts — not a working demo, just the important bits.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Blocked by

- A reference to the blocking ticket (if any)

Or "None - can start immediately" if no blockers.

</issue-template>

### 6. Add a "Build order" section to the parent issue

Once **all** sub-issues exist (so you can name real identifiers), append a `## Build order` section to the end of the parent issue's body. This is the one permitted edit to the parent — do not otherwise alter, and never close, the parent.

Match the repo's existing house style if a prior decomposed PRD has one. Otherwise use this format:

- A one-line intro naming the sub-issue range (e.g. "decomposed into N sub-issues (#nn–#nn), each carrying its own Blocked-by references").
- Then bullets that group the ordering picture, foregrounding parallelism:
  - **Start immediately, in parallel:** the slices with no blockers that touch disjoint areas (safe to run in concurrent worktrees).
  - **Critical path:** the strict blocked-by chain, written as `#a → #b → #c`.
  - **Closure:** the final slice(s) blocked by everything else.

Keep the Build order consistent with each sub-issue's own "Blocked by" field — they must not disagree.

**Keep the parent's in-repo mirror in sync.** If the parent PRD is mirrored under `docs/prd/` (see the `to-prd` skill), add the same `## Build order` section to that file in the same task, so the canonical issue and the mirror never drift. This is a pure-docs change — follow the repo's branch rules for docs.

After editing, re-read the parent issue and confirm the Build order rendered as intended.
