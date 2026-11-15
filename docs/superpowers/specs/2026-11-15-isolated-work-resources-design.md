# planter — Isolated, Named Resources per Work

**Date:** 2026-11-15
**Status:** Approved

## Intent

Two workshop workspaces often run at the same time. Today the worker and
reviewer panes start whatever the task needs — a Docker Compose stack, a
simulator, a browser — in the shared default place: the default Compose
project, the booted simulator, the user's own browser context. Two
workspaces then collide, and planter:sweep cannot tell which of the leftovers
belongs to the work it is sweeping.

Every resource a work starts outside its checkout runs in an instance of its
own, named after the work, so that concurrent workspaces never share one and
a sweep removes only what carries its own name.

## Goals

- One name per work: `<repo>-<branch>` lowercased, with every character
  outside `[a-z0-9_-]` turned into `-` (`feat/ABC-123-x` in repo `shop` →
  `shop-feat-abc-123-x`). Call it the **work name**. The repo is in it
  because one issue often opens the same branch in two repos at once, and
  those two works must not share a Compose project (final-review finding).
- Compose is isolated even when an agent forgets to ask for it.
- Simulators, hand-started containers, and browser task spaces are created
  fresh under the work name, never borrowed from the user's defaults.
- planter:sweep treats the work name as ownership evidence, alongside the
  plan file's `Resources` section.

## Non-Goals

- Automatic host-port remapping. Compose project names separate containers
  and networks, not fixed `ports:` bindings; those are handled by the rule
  below, not by planter.
- planter:tenant (ad-hoc delegation outside a workshop workspace) is
  unchanged.
- Browser profiles. ego-browser's isolation unit is the task space; profiles
  stay the user's choice.

## Anti-Goals

- Removing anything in a sweep because it was *used*, rather than because it
  carries the work name or is recorded in `Resources`.
- Profile-wide browser clears, or claiming a user-owned task space.

## Approaches

**A. Rule in the prompts only.** collab tells the worker and reviewer to
isolate and name everything they start; sweep matches the name. Smallest
diff, but one `docker compose up` without `-p` lands in the default project
and collides.

**B. Environment injection plus rule (chosen).** workshop exports
`PLANTER_WORK=<work name>` and `COMPOSE_PROJECT_NAME=<work name>` into every
pane, the same way it already carries `CLAUDE_CONFIG_DIR` / `CODEX_HOME`.
Compose is isolated by default; the rule in collab covers the rest, keyed on
`$PLANTER_WORK`; sweep finds resources by that name. Touches three skills and
the restart path in collab Step 0, but the common case needs no discipline
from the agent.

**C. A helper script shipped in the plugin.** Dropped: the plugin is
skills-only, and a script is one more thing every pane must find on `PATH`.

## Design

### workshop

- **Step 2** defines the work name next to the branch, and notes that
  everything the work starts outside the checkout carries it.
- **Step 5 (herdr)** exports `PLANTER_WORK` and `COMPOSE_PROJECT_NAME` into
  every pane before `herdr agent start`, through the existing
  `herdr pane run <pane_id> "export …"` path. Unlike the profile variables,
  these are always exported.
- **Step 5 (Herdr app)** puts the same two assignments into the `<env>`
  prefix of every terminal's argv, always.
- **Step 6** reports the work name.

### collab

- **Step 0, restart path**: a pane restarted by hand has no environment, so
  the planner re-exports the two variables (herdr: `pane run export` before
  `agent start`; Herdr app: the `<env>` prefix). The planner reads
  `$PLANTER_WORK` from its own environment; if unset (planner started outside
  workshop), it derives the work name from the branch.
- **Worker prompt (Step 6) and reviewer prompt (Step 7)** carry the
  isolation rule: anything started for this work outside the checkout is a
  fresh instance named `$PLANTER_WORK`, never the default one.
  - Compose: the project name comes from the environment; never pass another
    `-p`. Fixed host ports: if the compose file reads them from variables,
    set those per work; otherwise write an override file beside the plan
    file (`~/.claude/plans/<KEY>-<slug>.compose.yml`), pass it with `-f`,
    and record it in `Resources`.
  - Plain `docker run`: `--name "$PLANTER_WORK-<service>"` and
    `--label planter.work="$PLANTER_WORK"`.
  - iOS: `xcrun simctl create "$PLANTER_WORK" <device type> <runtime>` and
    boot that UDID; never the already-booted device. Android:
    `avdmanager create avd -n "$PLANTER_WORK" …` with a port of its own.
  - Browser (ego-browser): `taskSpace("$PLANTER_WORK")`; never
    `claimTaskSpace` / `takeOverTaskSpace` on a user-owned space.
  - `Resources` keeps recording exact ids as today.

### sweep

- **Step 3, removable**: in addition to the `working_dir` label match and
  `Resources`, anything carrying the work name — the Compose project
  `<work name>`, containers labeled `planter.work=<work name>`, a simulator
  (`xcrun simctl list devices -j`) or AVD named `<work name>`, an ego-browser
  task space of that name among the agent's own spaces, and the override
  compose file beside the plan file.
- **Step 5, remove**: `docker compose -p <work name> down` (still no `-v`),
  `xcrun simctl delete <udid>` / `avdmanager delete avd -n <work name>`,
  `task.finish()` on the task space, the override file.
- **Kept** is unchanged: volumes, images, and anything that existed before
  the work or lacks the name.

### README

Feature rows for workshop, collab, and sweep mention the work name and the
isolation rule in a clause each.

## Constraints

- No value typed into a pane may contain anything the shell reinterprets:
  the work name is `[a-z0-9_-]` by construction, but still quote it the way
  the profile variables are quoted.
- Compose project names must be lowercase `[a-z0-9_-]` — a dot is rejected,
  which is why the flattening maps every other character, not only `/`.
- A pane's shell does not inherit the planner's environment; every export is
  explicit, on both front ends and on every restart path.

## Left to the Builder

- Exact sentence placement inside each skill; keep each rule in one home and
  reference it from the others, as the skills do today.
- Which device type and runtime a simulator gets: the project's own
  convention, found by the worker, not fixed here.
- Whether the reviewer prompt needs the full rule or a one-line reference to
  the worker's — the reviewer rarely starts resources.
