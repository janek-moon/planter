---
name: issue-worktree
description: Use when work starts from an issue and needs its own place to live — "ABC-123 작업 시작해줘", "이 이슈로 worktree 열어줘", "이슈 만들고 브랜치 파줘", "start work on ABC-123", "open a worktree for this ticket" — takes a Jira/Linear issue key (or creates the issue), opens a git worktree in a new herdr workspace, and names the workspace after the issue
---

# Starting an Issue in Its Own herdr Workspace

## Overview

Turn an issue into a place to work: resolve (or create) the issue, open a git worktree on a conventionally named branch, and label the herdr workspace with the issue. It stops there — running anything inside that workspace is planter:tenant's job.

## Step 0: herdr only

Requires `$HERDR_ENV` = `1`. Worktree workspaces are a herdr feature; there is no tmux/cmux equivalent. If the check fails, STOP and report.

## Step 1: Resolve the issue

**Key given** (`ABC-123`, `XYZ-456`): look it up through whichever tracker MCP is connected — Linear and/or Jira. Both connected: query both, the one that returns the issue wins; if both return one, ask which. Nothing connected, or the key is not found: STOP and report. Never invent a title, and never create a new issue to cover a key that did not resolve.

**No key**: ask for the title and the destination (Linear team / Jira project), then read back what you are about to file — type, title, destination, assignee (yourself) — and create it only after the user confirms. Continue with the key it returns.

Keep from the issue: key, title, type, URL.

## Step 2: Derive the names

Branch — `<type>/<KEY>-<slug>`, same shape for both trackers:
- Type is the kind of work, not just the issue type: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`. A bug type means `fix`; otherwise read the title, and ask when it does not clearly map to one of these.
- Key keeps its uppercase form (`feat/ABC-123`, `feat/XYZ-456`).
- Slug is a short English phrase for what the work is, about three or four words (`fix/ABC-123-i18n-selector-drift`). A Korean title is not transliterated — either write the English phrase yourself or leave the slug off entirely.
- Linear issues carry a `gitBranchName` (`<username>/abc-123-<title>`). Do not adopt that format on your own — but a repo may genuinely be full of such branches, because they were made from Linear. That is a repo convention to weigh in the check below, not a reason to build the name that way yourself.
- Confirm against the repo before committing to a name: `git branch -a --format='%(committerdate:short) %(refname:short)' --sort=-committerdate | head -20`. Match the shape you see — prefix style, key casing, whether a slug is used at all.
- Repos are often mixed (one repo here carries both `feat/ABC-123` and a Linear-generated `<username>/abc-124-...` in the same month). When the recent branches disagree on shape, say what you found and ask which to use. Only the no-transliteration rule stands regardless: an existing branch with a Korean slug is not a reason to write another one.

Workspace label: `<KEY> <title>`, matching how issue workspaces already read in the sidebar. Use the title alone if the user asks for that.

Base: `git symbolic-ref --short refs/remotes/origin/HEAD` — it already resolves per repo (`origin/develop`, `origin/development`, `origin/main`), so do not assume `main`. If the ref is unset, ask instead of guessing. A base the user named wins.

## Step 3: Open the worktree

```
herdr worktree create --cwd "$PWD" --branch <branch> --base <base> --label "<label>" --no-focus
```

- Read `.result.workspace.workspace_id`, `.result.workspace.worktree.checkout_path`, and `.result.root_pane.pane_id` from the JSON. The checkout lands under `~/.herdr/worktrees/<repo>/<branch-slug>`.
- `--label` becomes the workspace name — no separate rename call.
- `--no-focus` leaves the user where they are. Pass `--focus` only when they asked to jump there.
- Any error other than the one below: STOP and report the JSON error as-is. Do not retry with a different branch name.
- Error `worktree_create_failed` with `already exists`: the worktree is already on disk. Reuse it — `herdr worktree open --cwd "$PWD" --branch <branch> --label "<label>" --no-focus` returns `already_open: true` and renames the workspace to the new label.

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

Leave the panes as empty shells. The names say what each is for; planter:tenant starts an agent in one when the user asks (`herdr agent start <name> --kind claude|codex --pane <pane_id>`).

## Step 5: Report

Workspace id and label, checkout path, branch, issue URL, and the three pane names. Do not start an agent — hand the workspace to planter:tenant if the user wants work running in it.

## Common Mistakes

- Creating an issue because a given key did not resolve — report the miss instead.
- Reaching for Linear's `gitBranchName` because it is there — build the name from the repo's convention instead.
- Transliterating a Korean title into the slug — leave the slug off instead.
- Removing a worktree to retry: `herdr worktree remove` deletes the checkout but leaves the branch behind, so the retry collides with the same branch name.
- Opening the worktree with `--focus` and yanking the user out of what they were doing.
