# Skills

A mix of the [Claude Code](https://claude.com/claude-code) skills Jesper uses on a daily
basis — some self-made, others curated from elsewhere and modified to fit my needs.

The two reference skills — `filament-5` and `nativephp-desktop` — are model-invokable: they
match on their own trigger vocabulary when the work touches them. The rest set
`disable-model-invocation: true` and are invoked explicitly by name: `/to-prd`, `/handoff`,
and so on.

## Self-made

| Skill | What it does |
|-------|--------------|
| [`filament-5`](./filament-5) | Building admin panels with Filament 5 — resources, forms, tables, infolists, actions, widgets, panels, theming. A short entry point plus nine topic references. |
| [`nativephp-desktop`](./nativephp-desktop) | Shipping Laravel apps as desktop apps with [NativePHP for Desktop](https://nativephp.com) 2.x — windows, menus, system APIs, builds and distribution, v1 upgrades. |

Both are written against pinned versions and say so in the skill itself. `nativephp-desktop`
carries a provenance table naming the exact package versions and the date its claims were
checked; treat filesystem paths and build outputs in it as perishable and re-check them
rather than trusting them after a dependency bump.

## Curated

All five come from [mattpocock/skills](https://github.com/mattpocock/skills). Most are
modified; `wait-what` is carried verbatim.

| Skill | Original | How mine differs |
|-------|----------|------------------|
| [`grill-with-docs`](./grill-with-docs) | [engineering/grill-with-docs](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs) | The earlier self-contained version, carrying the grilling loop and the domain-model rules in one file. Upstream has since split these into `grilling` + `domain-modeling` and reduced this skill to a stub that composes them. |
| [`handoff`](./handoff) | [productivity/handoff](https://github.com/mattpocock/skills/tree/main/skills/productivity/handoff) | Prints the handoff to the screen so I can decide where it goes — a file, a GitHub issue comment, or somewhere else — instead of writing it to the OS temp directory. |
| [`to-prd`](./to-prd) | [engineering/to-spec](https://github.com/mattpocock/skills/tree/main/skills/engineering/to-spec) (renamed upstream) | Keeps PRD wording, and adds the two-location workflow: the tracker issue is canonical, with a durable in-repo mirror under `docs/prd/` alongside `docs/adr/`. Also covers project-board membership and issue types. |
| [`to-issues`](./to-issues) | [engineering/to-tickets](https://github.com/mattpocock/skills/tree/main/skills/engineering/to-tickets) (renamed upstream) | Keeps issue wording, and adds native sub-issue linking, a `## Build order` section on the parent, and explicit guidance on shaping slices for parallel agents in isolated worktrees. |
| [`wait-what`](./wait-what) | [productivity/wait-what](https://github.com/mattpocock/skills/tree/main/skills/productivity/wait-what) | Unmodified. Stops a reply that didn't land and asks for it again in [ASD-STE100](https://www.asd-ste100.org/) Simplified Technical English, using the project's `CONTEXT.md` vocabulary. |

Upstream renamed `to-prd` → `to-spec` and `to-issues` → `to-tickets`. I kept the older names
because mine have diverged enough that they are no longer the same skill.

## Tracker configuration

`to-prd` and `to-issues` both publish to a project's issue tracker. Neither hardcodes a
tracker: they read the board name, its custom fields, and the enabled issue-type names from
the project's own agent guidelines, conventionally `docs/agents/issue-tracker.md`. Run
`/setup-matt-pocock-skills` in a repo to generate that file before using either skill there.
