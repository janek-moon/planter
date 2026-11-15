# Isolated, Named Resources per Work — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every Docker Compose stack, container, simulator, or browser task space a workshop work starts runs in its own instance named after the work, and planter:sweep removes exactly those.

**Architecture:** The skills are markdown instructions, so each task edits sentences in one `SKILL.md`. workshop defines the **work name** and exports `PLANTER_WORK` / `COMPOSE_PROJECT_NAME` into every pane; collab re-exports them on restart and puts the isolation rule into the worker and reviewer prompts; sweep adds "carries the work name" as ownership evidence and the matching removal commands.

**Tech Stack:** Claude Code / Codex plugin skills (`skills/*/SKILL.md`), herdr CLI (`herdr pane run`, `herdr agent start`), Herdr app (`hc`, `herdr-app do terminal.open` argv), `docker compose`, `xcrun simctl`, `avdmanager`, ego-browser `taskSpace()`.

**Spec:** `docs/superpowers/specs/2026-11-15-isolated-work-resources-design.md`

**Working directory:** `/Users/janek/Documents/dev/planter`, branch off `main` (`feat/isolated-work-resources`).

## Global Constraints

- Work name = branch lowercased with every `/` turned into `-` (`feat/ABC-123-x` → `feat-abc-123-x`); it is the same string as the worktree folder name in `~/.herdr/worktrees/<repo>/`.
- The two variables are exactly `PLANTER_WORK` and `COMPOSE_PROJECT_NAME`, both set to the work name, exported into **every** pane on **both** front ends, **always** (not only when a profile variable is set).
- Values typed into a pane are quoted with `printf %q`, as the profile variables are.
- sweep still never passes `-v` to `docker compose down`, never removes volumes or images, and removes nothing that neither carries the work name nor is listed in `Resources`.
- One rule lives in one skill; the others reference it (`planter:workshop Step 2`, `planter:collab Step 6`).
- Version bump: `0.15.0` → `0.16.0` in `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json`, as every feature PR does.
- Verification is `grep` on the skill text plus the manual scenario in Task 5; the repo has no test harness for skills.

## Review Focus

1. A planner started outside workshop (no `PLANTER_WORK` in its env) must derive the work name from the branch, not skip the exports — Task 2 Step 3.
2. The Herdr app `<env>` prefix must appear even when neither `CLAUDE_CONFIG_DIR` nor `CODEX_HOME` is set; today the prefix is empty in that case — Task 1 Step 5.
3. A compose file with fixed host `ports:` and no variable knob must get the override file, not a second `-p` or the default project — Task 2 Step 5.
4. sweep must not finish a user-owned ego-browser task space that happens to share the name — Task 3 Step 3 limits it to the agent's own spaces.
5. A branch with uppercase or `/` in it must never reach `COMPOSE_PROJECT_NAME` unflattened — Task 1 Step 3 defines the name once and the exports reference it.
6. A pane whose shell never received the export (agent started before this change, or a codex daemon from an earlier session) must still get the name — Task 2 Step 4 writes it literally into the prompt.

---

### Task 1: workshop — define the work name and export it into every pane

**Files:**
- Modify: `skills/workshop/SKILL.md:61-75` (Step 2), `:91` (herdr Step 3 path), `:144` (herdr Step 5 exports), `:156` (Herdr app Step 3 path), `:173` (Herdr app Step 5 `<env>`), `:190` (Step 6 report)

**Interfaces:**
- Produces: the term **work name** and the two variables `PLANTER_WORK`, `COMPOSE_PROJECT_NAME`, referenced by collab and sweep as "planter:workshop Step 2".

- [ ] **Step 1: Write the failing check**

```bash
cd /Users/janek/Documents/dev/planter
grep -c 'PLANTER_WORK' skills/workshop/SKILL.md        # expect 0 now, ≥3 after
grep -c 'Work name' skills/workshop/SKILL.md            # expect 0 now, 1 after
```

- [ ] **Step 2: Run it to confirm both print 0**

- [ ] **Step 3: Define the work name in Step 2**

After the `Workspace label:` paragraph (line 71), add a paragraph starting `Work name:` that states: the branch lowercased with every `/` turned into `-` (`feat/ABC-123-x` → `feat-abc-123-x`); it names the worktree folder, and everything the work starts outside the checkout — Compose project, containers, simulators, browser task space — carries it (planter:collab Step 6, planter:sweep Step 3). It is `[a-z0-9-]` by construction, which Compose accepts as a project name.

- [ ] **Step 4: Export always on the herdr side (Step 5, line 144)**

Edit the profile-export paragraph so it says: export `PLANTER_WORK` and `COMPOSE_PROJECT_NAME`, both set to the work name, into every pane you start, before any profile variable, in one command: `herdr pane run <pane_id> "export PLANTER_WORK=$(printf %q "<work name>") COMPOSE_PROJECT_NAME=$(printf %q "<work name>")"`. Then the existing profile text follows, with "Neither variable set: nothing to export" reworded to "Neither profile variable set: only the two work-name exports".

- [ ] **Step 5: Export always on the Herdr app side (Step 5, line 173)**

Change the `<env>` sentence so `<env>` is `PLANTER_WORK=<work name> COMPOSE_PROJECT_NAME=<work name> ` always, followed by `CLAUDE_CONFIG_DIR=<value> CODEX_HOME=<value> ` for whichever profile variable your own environment sets; each value quoted with `printf %q`.

- [ ] **Step 6: Point the two path sentences at the term**

Line 91: `<branch-slug>` → `<work name>`. Line 156: replace the inline derivation with "`~/.herdr/worktrees/<repo>/<work name>` (Step 2)", keeping the `<repo>` definition.

- [ ] **Step 7: Report it (Step 6, line 190)**

Add "the work name" after "branch" in the report list.

- [ ] **Step 8: Run the check**

```bash
grep -c 'PLANTER_WORK' skills/workshop/SKILL.md          # ≥ 3
grep -c 'Work name' skills/workshop/SKILL.md              # 1
grep -n 'branch-slug\|branch lowercased, every' skills/workshop/SKILL.md   # no output
```

- [ ] **Step 9: Commit**

```bash
git add skills/workshop/SKILL.md
git commit -m "feat(workshop): define the work name and export it into every pane"
```

---

### Task 2: collab — re-export on restart, isolation rule in the prompts

**Files:**
- Modify: `skills/collab/SKILL.md:26` (Step 0 restart), `:52` (Herdr app table row), `:104` (Step 6 worker prompt), `:110` (Step 7 reviewer prompt), `:131-` (Common Mistakes)

**Interfaces:**
- Consumes: work name, `PLANTER_WORK`, `COMPOSE_PROJECT_NAME` (Task 1).
- Produces: the **isolation rule** paragraph in Step 6, which sweep references as "planter:collab Step 6"; the override file path `~/.claude/plans/<KEY>-<slug>.compose.yml` (`<slug>.compose.yml` without an issue), which sweep removes.

- [ ] **Step 1: Write the failing check**

```bash
grep -c 'PLANTER_WORK' skills/collab/SKILL.md     # 0 now, ≥ 4 after
grep -c 'compose.yml' skills/collab/SKILL.md       # 0 now, 1 after
```

- [ ] **Step 2: Run it to confirm 0**

- [ ] **Step 3: Restart path re-exports (Step 0, line 26 and table line 52)**

Line 26, before "Start the agent with the kind…": a restarted pane has no environment, so first run planter:workshop Step 5's export of `PLANTER_WORK` and `COMPOSE_PROJECT_NAME` (and the profile variables) into it. The work name is `$PLANTER_WORK` from your own environment; unset, derive it as planter:workshop Step 2 says from `git branch --show-current`. Table line 52: the Herdr app cell gains "with planter:workshop's `<env>` prefix, work-name variables included".

- [ ] **Step 4: Worker prompt carries the isolation rule (Step 6, line 104)**

Replace the sentence beginning "Tell the worker that anything it newly creates…" with a paragraph that keeps the `Resources` record and adds the rule. The prompt states the work name literally — "the work name is `<work name>`; `$PLANTER_WORK` holds it in your shell" — because a pane whose agent predates the export, or a codex daemon started earlier, has no such variable, and an empty `$PLANTER_WORK` would silently yield `--name "-svc"` or `taskSpace("")`. The rule:
- Everything started for this work outside the checkout is a fresh instance named `$PLANTER_WORK`; never the default Compose project, the booted simulator, or a user-owned browser space.
- Compose: the project name is already in the environment; never pass another `-p`. Fixed host `ports:` read from variables → set them per work; no variable → write `~/.claude/plans/<KEY>-<slug>.compose.yml` remapping them, pass it with `-f` after the project's files, record it in `Resources`.
- `docker run`: `--name "$PLANTER_WORK-<service>" --label planter.work="$PLANTER_WORK"`.
- iOS: `xcrun simctl create "$PLANTER_WORK" <device type> <runtime>`, boot that UDID. Android: `avdmanager create avd -n "$PLANTER_WORK" …` on a port of its own.
- Browser (ego-browser): `taskSpace("$PLANTER_WORK")`; never `claimTaskSpace`/`takeOverTaskSpace` on a user-owned space.
- `Resources` keeps the exact id of each (compose project, container, UDID/AVD name, task space id, override file path); planter:sweep also finds anything carrying the name by itself.

- [ ] **Step 5: Reviewer prompt references it (Step 7, line 110)**

Append one clause: anything the reviewer itself starts follows the isolation rule in Step 6.

- [ ] **Step 6: Common Mistakes**

Add two bullets: running `docker compose` with a `-p` other than the environment's; booting the user's already-booted simulator or claiming their browser space instead of creating `$PLANTER_WORK`'s own.

- [ ] **Step 7: Run the check**

```bash
grep -c 'PLANTER_WORK' skills/collab/SKILL.md     # ≥ 4
grep -c 'compose.yml' skills/collab/SKILL.md       # 1
grep -n 'claimTaskSpace' skills/collab/SKILL.md    # 1 line, in Step 6
```

- [ ] **Step 8: Commit**

```bash
git add skills/collab/SKILL.md
git commit -m "feat(collab): isolate and name what the panes start, re-export on restart"
```

---

### Task 3: sweep — the work name is ownership evidence

**Files:**
- Modify: `skills/sweep/SKILL.md:42-48` (Step 3 removable/kept), `:60` (Step 5), `:91-` (Common Mistakes)

**Interfaces:**
- Consumes: work name (Task 1), isolation rule and override file path (Task 2).

- [ ] **Step 1: Write the failing check**

```bash
grep -c 'work name' skills/sweep/SKILL.md          # 0 now, ≥ 3 after
grep -c 'planter.work' skills/sweep/SKILL.md       # 0 now, 1 after
```

- [ ] **Step 2: Run it to confirm 0**

- [ ] **Step 3: Removable by name (Step 3, after line 43)**

Add one bullet: anything carrying the work name (planter:workshop Step 2; `$PLANTER_WORK` in your environment, or derived from the branch) because planter:collab Step 6 makes the panes create resources under it — the Compose project named `<work name>` (`docker compose ls -a` — stopped projects are the usual leftovers), containers labeled `planter.work=<work name>` (`docker ps -a --filter label=planter.work=<work name>`), an iOS simulator or AVD named `<work name>` (`xcrun simctl list devices -j`, `avdmanager list avd`), an ego-browser task space whose `listTaskSpaces()` entry has `name === "<work name>"` **and** `ownership !== "user"` (a user-owned space of the same name is never touched; a space `Resources` records by `id` is removable regardless of name), and the override file `~/.claude/plans/<KEY>-<slug>.compose.yml`.

Keep the `working_dir` bullet. In **Kept**, add "anything without the work name that `Resources` does not list".

- [ ] **Step 4: Remove by name (Step 5, line 60)**

Extend the sentence: `docker compose -p <work name> down` (no `-v`), `docker rm -f` the labeled containers, `xcrun simctl delete <udid>` / `avdmanager delete avd -n <work name>`, `task.finish({ keep: [] })` on the task space, `rm` the override file, then each recorded resource.

- [ ] **Step 5: Common Mistakes**

Add: finishing a browser task space because its name matches, without checking it is the agent's own.

- [ ] **Step 6: Run the check**

```bash
grep -c 'work name' skills/sweep/SKILL.md          # ≥ 3
grep -c 'planter.work' skills/sweep/SKILL.md       # 1
grep -n 'down -v' skills/sweep/SKILL.md            # only the existing Common Mistakes line
```

- [ ] **Step 7: Commit**

```bash
git add skills/sweep/SKILL.md
git commit -m "feat(sweep): remove what carries the work name"
```

---

### Task 4: README rows and version bump

**Files:**
- Modify: `README.md:43-45`, `README.ko.md:43-45`, `.claude-plugin/plugin.json:4`, `.codex-plugin/plugin.json:4`

- [ ] **Step 1: README rows**

workshop row: add "and names everything the work will start after the branch". collab row: add "the panes run Docker, simulators, and browsers in instances named after the work". sweep row: add "found by that name". Same three clauses in Korean in `README.ko.md`.

- [ ] **Step 2: Version**

`"version": "0.15.0"` → `"version": "0.16.0"` in both plugin.json files.

- [ ] **Step 3: Check**

```bash
grep -c '0.16.0' .claude-plugin/plugin.json .codex-plugin/plugin.json   # 1 each
grep -c 'named after the work' README.md                                 # ≥ 1
```

- [ ] **Step 4: Commit**

```bash
git add README.md README.ko.md .claude-plugin/plugin.json .codex-plugin/plugin.json
git commit -m "docs: describe isolated, named work resources; bump to 0.16.0"
```

---

### Task 5: Manual scenario check (no commit)

- [ ] **Step 1:** In the Herdr app, run planter:workshop on this repo without an issue ("isolated resources smoke"). Expected: `herdr-app ls` shows three terminals in the new space, and in each `hc read <t>` → `echo $PLANTER_WORK $COMPOSE_PROJECT_NAME` prints `feat-isolated-resources-smoke feat-isolated-resources-smoke` (adjust to the branch workshop derived).
- [ ] **Step 2:** In the worker pane, with any compose file and no `-p`: `docker compose -f <file> config | head -1` prints `name: <work name>` — the environment, not the folder name, named the project.
- [ ] **Step 3:** Run planter:sweep on that workspace with no PR. Expected: the Step 3 list names the work's Compose project line (empty) and nothing from another workspace; the worktree and branch are removed after confirmation.
- [ ] **Step 4:** Record the result in the plan file's final section and in the PR body.
