---
name: workshop
description: Use when a task needs its own place to live — from an issue or not — "ABC-123 작업 시작해줘", "이 이슈로 worktree 열어줘", "이슈 만들고 브랜치 파줘", "이슈 없이 worktree만 열어줘", "start work on ABC-123", "open a worktree for this" — takes a Jira/Linear issue key, files one, or works without any issue, then opens a git worktree in a new herdr workspace or Herdr app space named after the work
---

# Starting Work in Its Own herdr Workspace

## Overview

Turn a piece of work into a place to do it: resolve the issue, file one, or skip the tracker entirely, then open a git worktree on a conventionally named branch and label the herdr workspace with the work, and start an idle agent in each of its three panes. It stops there — prompting those agents is planter:collab's or planter:tenant's job.

## Step 0: herdr or the Herdr app only

Requires `$HERDR_ENV` = `1` (herdr) or `$HC_TERMINAL_ID` set with `herdr-app status` exiting 0 (the Herdr app — see planter:tenant for telling them apart). There is no tmux/cmux equivalent. If neither holds, STOP and report. Steps 1, 2, and 6 are the same for both; in the Herdr app, Steps 3–5 follow the **Herdr app** section below instead.

## Step 1: Resolve the issue

**Key given** (`ABC-123`, `XYZ-456`): look it up through whichever tracker MCP is connected — Linear and/or Jira. Both connected: query both, the one that returns the issue wins; if both return one, ask which. Nothing connected, or the key is not found: STOP and report. Never invent a title, and never create a new issue to cover a key that did not resolve.

**No key**: ask which of the two the user wants as a choice dialog — AskUserQuestion in Claude Code, `request_user_input` in Codex — whenever that tool is available this turn, otherwise in plain prose. Ask even when the user already said which — this is the last point before a tracker gets touched, and answering is one click or one word. Offer "File an issue" only when a tracker MCP is actually connected; when none is, say so and continue without an issue.

The two options:
- **File an issue** — ask what the work is and the destination (Linear team / Jira project), then follow **Filing issues** below. Continue with the key of the issue this workspace will carry.
- **No issue** — do not touch a tracker at all. Ask for the work in a sentence, and carry that instead of an issue.

Keep from the issue: key, title, type, URL. Working without one, keep just that sentence.

## Filing issues

Every issue planter files follows this section, here and in planter:sweep.

**Shape by size.** An epic is the work as a whole, a story is a use case, a task is one detailed feature.

| The work is | File |
|---|---|
| one detailed feature | a task |
| one use case with several features | a story and its tasks |
| several use cases | an epic, its stories, and their tasks |

**Map to the tracker.**

| Level | Linear | Jira |
|---|---|---|
| Epic | Project | Epic |
| Story | Issue | Story |
| Task under a story | Sub-issue (`parentId`) | Sub-task, with `parent` |
| Task on its own | Issue | Task |

Jira: read the project's real types first (`getJiraProjectIssueTypesMetadata`), because names and levels differ per project. When the project cannot express the hierarchy (a type is missing, or the type takes no parent), tell the user and ask.

**What goes in an issue.** These apply to every field you write: description, comments, and Linear's "Agent notes".
- Epic: the purpose and the scope.
- Story: what the user can do once it is done.
- Task: the capability needed and its completion criteria.
- To identify a problem, also include what was observed: reproduction steps, the error and where it appeared, an external contract the work must meet.

An issue describes the work needed and stops there. How to build it — files to change, function design, code, library choices, implementation order — is analyzed in the workspace that does the work (planter:collab Step 4). This overrides the Linear MCP's advice to keep implementation plans in "Agent notes".

**Confirm, then create.** Read back the whole tree: each issue's type, title, and body, the destination, assignee yourself. In the same read-back, settle which single issue this workspace carries; its key names the branch and label. Create only after the user confirms. Create parents first and link children by the returned keys. If a create fails partway, report the keys already made and reuse them on retry, so nothing is filed twice.

## Step 2: Derive the names

Branch — `<type>/<KEY>-<slug>`, same shape for both trackers. Without an issue the key drops out and the slug carries the whole name (`feat/parallel-upload-retry`). The key-casing rule below then has nothing to derive from; the rest still applies:
- Type is the kind of work, not just the issue type: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`. An issue typed as a bug means `fix`; otherwise read the title — or, with no issue, the sentence from Step 1 — and ask when it does not clearly map to one of these.
- Key keeps its uppercase form (`feat/ABC-123`, `feat/XYZ-456`).
- Slug is a short English phrase for what the work is, about three or four words (`fix/ABC-123-i18n-selector-drift`). A Korean title is not transliterated — write the English phrase yourself. Dropping the slug is only an option when a key carries the name (`feat/ABC-123`); without an issue the slug is the whole name, and `feat/` is not a valid branch name. If the work will not condense into an English phrase, ask the user for one rather than shipping a transliteration.
- Linear issues carry a `gitBranchName` (`<username>/abc-123-<title>`). Do not adopt that format on your own — but a repo may genuinely be full of such branches, because they were made from Linear. That is a repo convention to weigh in the check below, not a reason to build the name that way yourself.
- Confirm against the repo before committing to a name: `git branch -a --format='%(committerdate:short) %(refname:short)' --sort=-committerdate | head -20`. Match the shape you see — prefix style, key casing, whether a slug is used at all.
- Repos are often mixed (one repo here carries both `feat/ABC-123` and a Linear-generated `<username>/abc-124-...` in the same month). When the recent branches disagree on shape, say what you found and ask which to use. Only the no-transliteration rule stands regardless: an existing branch with a Korean slug is not a reason to write another one.

Workspace label: `<KEY> <title>`, matching how issue workspaces already read in the sidebar. Use the title alone if the user asks for that. Without an issue, the label is the work in a few words — the sentence from Step 1, trimmed to what fits a sidebar.

Base: `git symbolic-ref --short refs/remotes/origin/HEAD` — it already resolves per repo (`origin/develop`, `origin/development`, `origin/main`), so do not assume `main`. If the ref is unset, ask instead of guessing. A base the user named wins.

## Step 3: Open the worktree

Bring the base up to date first, every time — `origin/<base>` is a remote-tracking ref, and only a fetch moves it:

```
git fetch origin
```

Fetch fails (offline, auth): STOP and report — do not create the branch from a base you could not refresh.

Also note whether the branch already exists locally (`git rev-parse --verify --quiet refs/heads/<branch>`). An existing branch is checked out as it is — `--base` only seeds a new one — so it needs the rebase in Step 4 whichever way the worktree opens.

```
herdr worktree create --cwd "$PWD" --branch <branch> --base <base> --label "<label>" --no-focus
```

- Read `.result.workspace.workspace_id`, `.result.workspace.worktree.checkout_path`, and `.result.root_pane.pane_id` from the JSON. The checkout lands under `~/.herdr/worktrees/<repo>/<branch-slug>`.
- `--label` becomes the workspace name — no separate rename call.
- `--no-focus` leaves the user where they are. Pass `--focus` only when they asked to jump there.
- Any error other than the one below: STOP and report the JSON error as-is. Do not retry with a different branch name.
- Error `worktree_create_failed` with `already exists`: the worktree is already on disk. Reuse it — `herdr worktree open --cwd "$PWD" --branch <branch> --label "<label>" --no-focus` returns `already_open: true` and renames the workspace to the new label. The existing branch is rebased onto the fetched base in Step 4, once its panes have been checked.

## Step 4: Lay out the three panes

Every issue workspace gets the same three: `planner` top left, `worker` below it, `reviewer` down the right side. The root pane becomes the planner, so only two splits are needed (`--ratio` defaults to 0.5, which is the shape these workspaces already have):

```
herdr pane rename <root_pane_id> planner
herdr pane split --pane <root_pane_id> --direction right --cwd "<checkout_path>" --no-focus
herdr pane rename <pane_id from that response> reviewer
herdr pane split --pane <root_pane_id> --direction down --cwd "<checkout_path>" --no-focus
herdr pane rename <pane_id from that response> worker
```

Each split returns the new pane at `.result.pane.pane_id` — rename that one, not the root. `--cwd` matters because only the root pane inherits the checkout path; without it the splits open wherever the caller was.

Reuse path (`already_open: true`): list what is there with `herdr pane list --workspace <workspace_id>` before touching anything. Label only what is missing, and run only the splits needed to get back to three — never a second set. If three panes exist but carry no labels, `herdr pane layout --pane <root_pane_id>` gives their positions: top left is the planner, below it the worker, the full-height one on the right the reviewer.

Then, when the branch existed before this run (Step 3), rebase it onto the fetched base — only after seeing the panes, never under a running agent:
1. Every pane must be empty or hold an agent that is `idle` or `done` (`herdr pane get <pane_id>`). `working`, `blocked`, or `unknown`: STOP and report.
2. No rebase may already be in progress: `git -C <checkout_path> rev-parse --git-path rebase-merge` and `--git-path rebase-apply` must not exist as directories. Either one does: STOP and report — that state is not yours.
3. `git -C <checkout_path> status --porcelain --untracked-files=all` must print nothing. Any output: STOP and report.
4. The branch is already on origin (`git ls-remote --exit-code --heads origin <branch>`): ask before rebasing — rewriting published commits means the next push must be `git push --force-with-lease`. Declined: leave the branch as it is and say so.
5. `git -C <checkout_path> rebase --no-autostash <base>`. Any nonzero exit — a conflict or anything else: `git -C <checkout_path> rebase --abort`, then STOP and report. The check in 2 makes any rebase state now this run's own.

Never stash, clean, or resolve anything yourself.

## Step 5: Start the agents

Each pane gets its agent, launched but not prompted. Defaults:

| Pane | Kind | Effort | Flags after `--` |
|------|------|--------|------------------|
| planner | claude | high | `--effort high` |
| worker | claude | medium | `--effort medium` |
| reviewer | codex | medium | `-c model_reasoning_effort=medium` |

The user can override kind and effort per pane in the request ("reviewer는 claude effort high로", "worker codex high"). Map the effort onto the kind's flag: claude `--effort <low|medium|high|xhigh|max>`, codex `-c model_reasoning_effort=<level>`. An effort the kind does not accept fails `agent start` — report it and ask, do not guess a nearby level. If `command -v codex` fails: a default codex reviewer runs as claude with the same effort, and you say so; a codex the user asked for by name means STOP and report.

Which panes: on a fresh workspace, all three. On the reuse path, only a pane whose `.agent` is empty and whose screen (`herdr pane read <pane_id> --source visible --lines 5`) shows a shell prompt. A running agent stays as it is; anything else is reported, never typed into. If the user named a kind or effort for a pane that already runs an agent, say that the running one was kept and ask whether to restart it — never stop it yourself.

The agents run under the same profile as the session running this skill. A pane's shell does not inherit the caller's environment, so check your own for `CLAUDE_CONFIG_DIR` and `CODEX_HOME` and export every one that is set into every pane you start, whatever its kind — the planner needs the reviewer's profile too, to restart it later (planter:collab Step 0). Let `printf %q` quote the value so nothing in it is reinterpreted on either side: `herdr pane run <pane_id> "export CODEX_HOME=$(printf %q "$CODEX_HOME")"`. In the pane that will run codex under a non-default `CODEX_HOME`, start its daemon first, ending with a sentinel unique to this call (`<tok>` = e.g. `date +%s%N`): `herdr pane run <pane_id> '(cd ~ && codex app-server daemon start >/dev/null); echo daemon-<tok>-$?'`. Read the pane for up to 30 s for a whole line `daemon-<tok>-0` — the echoed command shows `-$?`, so it cannot match. Any other number, or nothing within 30 s: STOP and report. Neither variable set: nothing to export. A profile carried by a secret (`CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`) is never typed into a pane — the value would sit on screen and in shell history; ask the user how that profile is launched instead.

```
herdr agent start <role>-<workspace_id lowercased> --kind <kind> --pane <pane_id> -- <flags>
```

The workspace id in the name keeps it unique — every issue workspace has a planner, worker, and reviewer. Start the three one after another; each call returns once its agent is ready for input. On `timeout` — typically a trust dialog for the fresh checkout — read the pane and report what it says, as planter:tenant-claude describes; do not answer it.

## Herdr app: Steps 3–5

The app has no worktree command, so git makes the checkout and the app gets a space for it. Same rules as above where not restated: fetch first, never retry under another branch name, rebase an existing branch only after the checks in Step 4, never stash or resolve anything yourself.

**Step 3 — checkout.** `git fetch origin` (fails: STOP). The path follows herdr's layout so both front ends find the same checkouts: `~/.herdr/worktrees/<repo>/<branch lowercased, every / turned into ->` (`feat/ABC-123-x` → `feat-abc-123-x`), where `<repo>` is the main checkout's folder name (`basename "$(dirname "$(git rev-parse --path-format=absolute --git-common-dir)")"`).
- `git worktree list --porcelain` already shows the branch checked out: reuse that path as it is.
- Otherwise the branch exists locally: `git worktree add <path> <branch>`. It does not: `git worktree add --no-track -b <branch> <path> <base>` — without `--no-track` the new branch would track the base (`origin/main`), and a later push or pull would aim at the base branch.
- Any error: STOP and report it as-is.

**Step 3 — space.** Look in `herdr-app ls` for a space whose `.folder` is the checkout path. Found: reuse it, and rename it to the label (`herdr-app do space.rename --id <space> --name "<label>"`). Not found:

```
herdr-app do space.new --name "<label>" --folder "<checkout_path>" --tab main --argv '<planner argv>'
```

`<planner argv>` is the JSON array `["<$SHELL>", "-lic", "<env><planner command>; exec \"$SHELL\" -l"]` from Step 5 — build it with a JSON encoder, not by hand. It returns `{ "space", "tab", "terminal" }` and switches the window to the new space — there is no no-focus option. Unless the user asked to jump there, switch back to the space you run in (`.labels["herdr.space"]` of `hc get "$HC_TERMINAL_ID" --json`, read before `space.new`) once Step 5 has opened its splits — they focus the new space again: `herdr-app do space.select --id <that space>`.

**Step 4 — panes.** Same shape as herdr: `planner` top left, `worker` below it, `reviewer` down the right side. The space's first terminal is the planner. A terminal added with `hc run --label herdr.space=…` lands wherever the window's layout puts it (three columns), so the other two are opened as splits of the planner pane instead, in Step 5. The app keeps no terminal name for them, so every role is marked with an hc label — `hc label <t> planter.role=<planner|worker|reviewer>` — and found by it: walk the space's terminals in `herdr-app ls` and read `.labels["planter.role"]` from `hc get <t> --json`. Label the planner as soon as `space.new` returns.

Reuse path: list the space's terminals in `herdr-app ls` before touching anything, and create only what is missing — never a second worker or reviewer. The rebase checks of Step 4 apply unchanged, with `hc get <t> --json` `.status.state` in place of `herdr pane get` (no `.status.app` means a bare shell).

**Step 5 — agents.** Defaults, overrides, and the codex fallback are the table above. Command per kind: claude `claude --effort <level>`, codex `codex --no-daemon -c model_reasoning_effort=<level>` (`--no-daemon` is required under hc; the daemon setup above does not apply here). `<env>` carries the profile: `CLAUDE_CONFIG_DIR=<value> CODEX_HOME=<value> ` for whichever of the two your own environment sets, each value quoted with `printf %q`, into every terminal whatever its kind. A secret-carried profile is still never put on a command line.

- Planner: a new space starts it through `--argv` above. On the reuse path, a planner terminal at a shell prompt (`hc read <terminal> --lines 5`) gets `hc send <terminal> '<env><command>'`. Do not type into the shell of a space you just made: its startup files may still be loading, and the typed Enter then lands on claude's trust dialog and picks "No, exit".
- Reviewer, then worker — each a split of the planner pane (its `.pane` id in `herdr-app ls`). `terminal.open` splits the focused pane, so focus the planner pane first, every time:
  ```
  herdr-app do pane.focus --pane <planner_pane>
  herdr-app do terminal.open '{"place": "right", "folder": "<checkout_path>", "argv": ["<$SHELL>", "-lic", "<env><reviewer command>; exec \"$SHELL\" -l"]}'
  herdr-app do pane.focus --pane <planner_pane>
  herdr-app do terminal.open '{"place": "down", "folder": "<checkout_path>", "argv": ["<$SHELL>", "-lic", "<env><worker command>; exec \"$SHELL\" -l"]}'
  ```
  Build each JSON object with a JSON encoder, not by hand. Each call returns `{ "pane", "terminal" }`; label that terminal with its role right away. Right first, then down, gives the full-height reviewer on the right and the worker under the planner. The login shell loads the user's `PATH` (a bare `codex` fails to spawn without it), and `exec "$SHELL" -l` leaves a shell behind if the agent exits.
  Then switch back to your own space with `space.select` (Step 3).

After each start, `hc wait <t> --timeout 60` returns `idle` once the agent is ready for input; confirm `hc get <t> --json` shows the expected `.status.app` (`claude-code` or `codex`). Anything else — typically a trust dialog for the fresh checkout — read the screen (`hc read <t> --lines 50`) and report it; do not answer it. On the reuse path, a terminal that already runs an agent (`.status.app` set) is kept as it is, as above; if its `.status.app` is not the kind that role should run, say so and ask — never stop it yourself.

## Step 6: Report

Workspace (or space) id and label, checkout path, branch, the issue URL when there is one, and each pane with the agent, kind, effort, and profile it runs. The agents sit idle — the work itself starts when the user runs planter:collab in the planner pane, or hands a task over through planter:tenant.

## Common Mistakes

- Creating an issue because a given key did not resolve — report the miss instead.
- Reaching for Linear's `gitBranchName` because it is there — build the name from the repo's convention instead.
- Transliterating a Korean title or sentence into the slug — write a short English phrase instead.
- Dropping the slug when there is no key: `feat/` alone is a name git refuses.
- Removing a worktree to retry: `herdr worktree remove` deletes the checkout but leaves the branch behind, so the retry collides with the same branch name.
- Opening the worktree with `--focus` and yanking the user out of what they were doing.
- Creating the worktree without `git fetch origin` first — the branch starts from a stale base.
- Stashing or resolving a rebase conflict yourself on the reuse path — abort and report instead.
- Running `pane run` exports into a pane that already runs an agent — the text lands in its input box.
