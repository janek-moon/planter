---
name: issue-worktree
description: Use when a task needs its own place to live — from an issue or not — "ABC-123 작업 시작해줘", "이 이슈로 worktree 열어줘", "이슈 만들고 브랜치 파줘", "이슈 없이 worktree만 열어줘", "start work on ABC-123", "open a worktree for this" — takes a Jira/Linear issue key, files one, or works without any issue, then opens a git worktree in a new herdr workspace named after the work
---

# Starting Work in Its Own herdr Workspace

## Overview

Turn a piece of work into a place to do it: resolve the issue, file one, or skip the tracker entirely, then open a git worktree on a conventionally named branch and label the herdr workspace with the work, and start an idle agent in each of its three panes. It stops there — prompting those agents is planter:issue-to-pr's or planter:tenant's job.

## Step 0: herdr only

Requires `$HERDR_ENV` = `1`. Worktree workspaces are a herdr feature; there is no tmux/cmux equivalent. If the check fails, STOP and report.

## Step 1: Resolve the issue

**Key given** (`ABC-123`, `XYZ-456`): look it up through whichever tracker MCP is connected — Linear and/or Jira. Both connected: query both, the one that returns the issue wins; if both return one, ask which. Nothing connected, or the key is not found: STOP and report. Never invent a title, and never create a new issue to cover a key that did not resolve.

**No key**: ask which of the two the user wants as a choice dialog — AskUserQuestion in Claude Code, `request_user_input` in Codex — whenever that tool is available this turn, otherwise in plain prose. Ask even when the user already said which — this is the last point before a tracker gets touched, and answering is one click or one word. Offer "File an issue" only when a tracker MCP is actually connected; when none is, say so and continue without an issue.

The two options:
- **File an issue** — ask for the title and the destination (Linear team / Jira project), read back what you are about to file (type, title, destination, assignee: yourself), and create it only after the user confirms. Continue with the key it returns.
- **No issue** — do not touch a tracker at all. Ask for the work in a sentence, and carry that instead of an issue.

Keep from the issue: key, title, type, URL. Working without one, keep just that sentence.

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

The agents run under the same profile as the session running this skill. A pane's shell does not inherit the caller's environment, so check your own for `CLAUDE_CONFIG_DIR` and `CODEX_HOME` and export every one that is set into every pane you start, whatever its kind — the planner needs the reviewer's profile too, to restart it later (planter:issue-to-pr Step 0). Let `printf %q` quote the value so nothing in it is reinterpreted on either side: `herdr pane run <pane_id> "export CODEX_HOME=$(printf %q "$CODEX_HOME")"`. In the pane that will run codex under a non-default `CODEX_HOME`, start its daemon first, ending with a sentinel unique to this call (`<tok>` = e.g. `date +%s%N`): `herdr pane run <pane_id> '(cd ~ && codex app-server daemon start >/dev/null); echo daemon-<tok>-$?'`. Read the pane for up to 30 s for a whole line `daemon-<tok>-0` — the echoed command shows `-$?`, so it cannot match. Any other number, or nothing within 30 s: STOP and report. Neither variable set: nothing to export. A profile carried by a secret (`CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`) is never typed into a pane — the value would sit on screen and in shell history; ask the user how that profile is launched instead.

```
herdr agent start <role>-<workspace_id lowercased> --kind <kind> --pane <pane_id> -- <flags>
```

The workspace id in the name keeps it unique — every issue workspace has a planner, worker, and reviewer. Start the three one after another; each call returns once its agent is ready for input. On `timeout` — typically a trust dialog for the fresh checkout — read the pane and report what it says, as planter:tenant-claude describes; do not answer it.

## Step 6: Report

Workspace id and label, checkout path, branch, the issue URL when there is one, and each pane with the agent, kind, effort, and profile it runs. The agents sit idle — the work itself starts when the user runs planter:issue-to-pr in the planner pane, or hands a task over through planter:tenant.

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
