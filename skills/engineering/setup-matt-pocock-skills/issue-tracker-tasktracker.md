# Issue tracker: Tasktracker

This file teaches how the matt-skills vocabulary maps onto `tt`. The other engineering skills (`to-issues`, `to-prd`, etc.) speak in abstract phrases — "publish to the issue tracker", "fetch the relevant ticket". This file translates each phrase an agent will actually encounter into the concrete `tt` actions to take. Phrases that arise only from the rejected `triage` skill are not mapped here.

> Load the tt skill (`tt` skill, named `tt`) — it teaches the agent loop, planning, task concepts, and command syntax in full. This doc covers only the integration surface; it does not re-teach `tt`.

> **The `triage` skill is not recommended with this tracker.** `triage` routes work through a label-based state machine (`needs-triage` → `needs-info` → `ready-for-agent` → `ready-for-human` → `wontfix`). `tt` routes work through assignee (`any` / `human`) plus dependency/blocking relationships, with `tt request` resolving the queue automatically. The label machinery does not map onto that model and would only add noise. Skip `/triage` for repos using tt; represent routing decisions directly as `tt assign` and `tt block` instead.

The `tt` CLI must be on the path. For command syntax run `tt --help` or `tt <cmd> --help`.

## Conventions

- **Issues and PRDs**: Issues live as `tt` tasks. PRDs live as files at `tmp/PRD-<feature-slug>.md` for the lifespan of the associated task set — kept as a reference for cross-checking how the PRD was decomposed into tasks. The `feature` task's prompt summarizes the PRD's top-level elements (goal, acceptance criteria); it is the durable record. The PRD file is not maintained after decomposition, but is not discarded while the task set exists.
- **Task types**: Use `feature` (a group type) for the parent task of a unit of work. Use `issue` (a leaf type) for each actual unit of work. Run `tt types` for the full listing. A `feature` may contain sub-`feature`s, so for large breakdowns you can nest group tasks with `issue`s as the leaves. Default to a single `feature` + `issue`s; introduce subgroups only when a flat list would be unwieldy.
- **No orphan leaf tasks**: Every `issue` lives under a `feature` parent (`-p <parent_id>`).
- **Default assignee**: `tt create` defaults `assignee=any`, so newly created tasks are immediately ready for agents to claim via `tt request`. Override with `-a human` only when a task genuinely needs a human.

## Finding the feature_id

Several operations below take `<feature_id>`. Find it with:

- `tt search "<title or keyword>"` — locate a feature by name.
- `tt list` — recent tasks, newest first.
- `tt list-tree <id>` — confirm the subtree once you have a candidate.

Convention: when you create a `feature` task, record its id in the PRD header (e.g. a `Feature: f-123` line) so the next agent picking up the PRD knows where to attach child issues.

## Skill-phrase translations

The abstract phrases used by the other engineering skills map to `tt` actions as follows.

### "publish to the issue tracker"

Write the PRD to `tmp/PRD-<feature-slug>.md`. Create the parent `feature`
group task describing the overall goal and acceptance criteria, then create
each child `issue` task with `-p <feature_id>`. The `issue` prompt holds the
unit-of-work description and acceptance criteria; the `feature` prompt holds
the umbrella goal. Set sibling dependencies with
`tt block <blocked> <blocker>` where A must complete before B. Verify the
tree with `tt list-tree <feature_id>`.

The "apply the ready-for-agent triage label" step that often follows publish
is a no-op — tasks are born `assignee=any` by default, so they're immediately
ready for agents to claim.

Do not close or modify the parent `feature` task as part of publishing — only
add the child `issue`s.

For large efforts, cluster slices under sub-`feature`s (each created with
`-p <parent_feature_id>`) rather than one flat list of `issue`s. Keep
`issue`s as the leaves; `tt request` traverses the nested tree correctly.

### "fetch the relevant ticket"

`tt view <id>` to read a task's details, comments, and relations. When you
are picking up work rather than just inspecting, prefer
`tt request <feature_id>` — it claims and returns the next ready task
instead of you manually selecting one.

### "fetch the PRD"

Read `tmp/PRD-<feature-slug>.md`. The PRD is a handoff document — it may have
been produced in another context (a different session, `/to-prd`, or a human).
It persists for the lifespan of the task set as a reference for cross-checking
the decomposition. The `feature` task's prompt summarizes the top-level
elements; the PRD file holds the full content. Recover the slug from the
feature task's prompt if not given directly.

### "escalate to a human" (mid-work)

When you hit a wall mid-task and need human input, `tt escalate <id> "<reason>"`
releases the task and reassigns it to a human in one step.
