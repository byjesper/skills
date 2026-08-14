---
name: to-prd
description: Turn the current conversation into a PRD, publish it to the project issue tracker, and mirror it in-repo under docs/prd/ — no interview, just synthesis of what you've already discussed.
disable-model-invocation: true
---

This skill takes the current conversation context and codebase understanding and produces a PRD. Do NOT interview the user — just synthesize what you already know.

The issue tracker and triage label vocabulary should have been provided to you — run `/setup-matt-pocock-skills` if not.

## Process

1. Explore the repo to understand the current state of the codebase, if you haven't already. Use the project's domain glossary vocabulary throughout the PRD, and respect any ADRs in the area you're touching.

2. Sketch out the seams at which you're going to test the feature. Existing seams should be preferred to new ones. Use the highest seam possible. If new seams are needed, propose them at the highest point you can. The fewer seams across the codebase, the better - the ideal number is one.

Check with the user that these seams match their expectations.

3. Write the PRD using the template below.

4. **Publish to BOTH places and keep them in sync.** A PRD lives in two mirrored locations: the **issue tracker is canonical**; an **in-repo copy under `docs/prd/`** is a durable mirror alongside `docs/adr/` and `DESIGN.md`.

   a. **Publish to the issue tracker first**, applying the `ready-for-agent` triage label — no additional triage. Capture the new item's **identifier and URL** from the create output.

      **Add it to the project board and give it an issue type.** A PRD is the one item most likely to be read by someone who did not write it, so neither is optional.

      The tracker's mechanics — create and edit commands, the board's name, its custom fields and their vocabulary — are repo-specific. Read them from the tracker doc named in the project's agent guidelines (conventionally `docs/agents/issue-tracker.md`, written by `/setup-matt-pocock-skills`). Never assume a type, field, or status name exists; read the project's enabled values first.

      Two things hold across trackers whatever the local vocabulary:

      - **Type** — a PRD describes a new capability, so it takes whatever the tracker calls a *feature*. Reserve the *bug* type for a PRD whose whole subject is fixing broken behaviour.
      - **Status** — a PRD tracking issue is a rollup, not a pickable slice. Give it an in-progress-style status rather than a ready/pickable one, so nobody grabs the parent instead of a child. If the board defines further custom fields (area, priority, blocked-by), set them too — a card with a blank status is a card nobody can plan around.

      <github-example>
      Where the tracker is GitHub, driven by `gh`:

      - **Project** — pass `--project "<title>"` to `gh issue create`. It resolves org- and user-owned projects by title.
      - **Type** — there is **no** `--type` flag on `gh issue create` or `gh issue edit` (checked at gh 2.92). Set it afterwards over REST, which takes the type's **name**:

        ```
        gh api --method PATCH repos/<owner>/<repo>/issues/<n> -f type=Feature
        ```

        Read the org's enabled types rather than assuming the name exists:

        ```
        gh api graphql -f query='query{organization(login:"<org>"){issueTypes(first:20){nodes{name isEnabled}}}}'
        ```
      </github-example>

      Verify both stuck before moving on to the mirror.

   b. **Write the in-repo mirror** at `docs/prd/<issue-number>-<slug>.md` — zero-padded to match the `docs/adr/NNNN-*.md` convention (e.g. `docs/prd/0002-<slug>.md`). The file opens with YAML front matter, then the title, then a line linking the canonical issue, then the body:

      ```
      ---
      status: ready-for-agent   # draft | ready-for-agent | in-progress | shipped | superseded
      issue: <n>                # canonical tracker item identifier
      created: <YYYY-MM-DD>     # the item's creation date
      labels: [ready-for-agent] # the item's triage labels
      ---

      # <PRD title>

      > **Tracking issue:** [#<n>](<issue-url>) — canonical. This file mirrors it; keep both in sync.
      ```

      `status` starts at `ready-for-agent` (matching the applied label) and is bumped as the PRD progresses. If `docs/prd/README.md` does not exist, create it as an index table mirroring `docs/adr/README.md` (columns: #, Title, Status, Tracking issue); add a row for the new PRD. Commit the repo copy (pure docs — follow the repo's branch rules for docs).

   c. **Formatting mirrors the issue — keep one line per paragraph.** The in-repo copy keeps the issue's exact **one-line-per-paragraph** formatting (most trackers, GitHub included, render single newlines as line breaks). Where the repo has a hard-wrap convention for prose, this is a **deliberate exception** to it — that convention applies to hand-authored docs like `DESIGN.md` and ADRs. Because a PRD mirror is machine-copied from its canonical issue, keeping it byte-identical to the body keeps the two a **trivial diff apart** and makes syncing reliable. Do **not** hard-wrap these files.

   d. **Mirror the issue's comments too** when they carry decisions or completion records, under a `## Tracking-issue comments` section at the end (`### @author — YYYY-MM-DD` per comment). Skip pure chatter.

   e. **On any later edit to a PRD, update BOTH places in the same task** — the tracking issue and the `docs/prd/` file must never drift. Whichever you change, mirror the change to the other.

<prd-template>

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

Exception: if a prototype produced a snippet that encodes a decision more precisely than prose can (state machine, reducer, schema, type shape), inline it within the relevant decision and note briefly that it came from a prototype. Trim to the decision-rich parts — not a working demo, just the important bits.

## Testing Decisions

A list of testing decisions that were made. Include:

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

</prd-template>
