---
name: issue-to-pr
description: Use when running in a herdr planner pane and asked to carry an issue through to a pull request — "ABC-123 끝까지 진행해줘", "이슈 분석해서 구현 계획 작성해", "reviewer pane 에 계획/diff 리뷰 받아", "worker pane 에 구현 위임해", "coderabbit 리뷰 후 PR 생성", "plan this issue and hand it to the worker pane", "get the reviewer pane to review this" — the planner analyzes, plans, and gates; the worker and reviewer panes do the implementing and reviewing
---

# Driving an Issue to a PR from the planner Pane

## Overview

A herdr issue workspace (see planter:issue-worktree) has three panes: `planner`, `worker`, `reviewer`. This skill is what the planner does in it. The planner reads the issue, writes the plan, gets it reviewed, delegates the implementation, triages review findings, and opens the PR. **The planner never writes product code.** Implementation goes to the worker pane; review goes to the reviewer pane; both are visible to the user, which is the point.

**REQUIRED BACKGROUND:** planter:tenant-claude (herdr section) for `herdr agent prompt/wait/read` mechanics. This skill only adds what is specific to the three-pane flow.

## Step 0: herdr only, and find your panes

Requires `$HERDR_ENV` = `1`. Then pick the targets from your own workspace:

```
herdr pane list --workspace "$HERDR_WORKSPACE_ID"
```

Use the `pane_id` of the rows labeled `worker` and `reviewer`; your own pane is `$HERDR_PANE_ID`. Always scope to `$HERDR_WORKSPACE_ID` — other workspaces carry panes with the same labels.

Pane commands must run outside any sandbox the tool wraps around shell calls (in Claude Code, Bash with `dangerouslyDisableSandbox: true`).

If a target's `.agent` is empty (or `herdr agent read` returns `agent_not_found`), it is a bare shell — common after a herdr restart. Start the agent as planter:tenant-claude describes: `herdr agent start <name> --kind claude --pane <worker pane_id>` for the worker, `--kind codex` for the reviewer. It returns once the agent is ready for input.

## Step 1: Analyze the issue

- Read the issue through whichever tracker MCP is connected (Linear or Jira): body, parent, linked issues, and **comments** — a comment may already say the work is not needed or decided. With Jira, request the comment field explicitly; the default fetch omits it.
- Locate the code the change will touch.
- Collect the repo's own rules before planning: verification command, commit message rules, review contract, PR title format, base branch. They live in `AGENTS.md` / `CLAUDE.md` / `.github/`; do not invent them. Base branch: `git symbolic-ref --short refs/remotes/origin/HEAD`.

## Step 2: Settle open questions from evidence first

When behavior, wording, or a rule is ambiguous, look for the answer before asking: the repo docs, and any reference implementation or upstream system the repo docs point at (clone it shallowly into the scratch directory if it is a separate repository). Ask the user only what the evidence does not settle — as a choice dialog (AskUserQuestion in Claude Code, `request_user_input` in Codex), two to four options.

## Step 3: Write the plan file

Write a plan file outside the repo (for example `~/.claude/plans/<KEY>-<slug>.md`). It carries: decisions with their evidence, files to change, tests, and a **numbered list of verification items**. If you are in plan mode, do not call ExitPlanMode — the plan ends with delegation, not with you implementing it.

## Step 4: Plan review in the reviewer pane

Prompt the reviewer pane with the plan file path, the verification item numbers, and the output format: one line reading `approved` / `conditional` / `rejected`, then the reasons. `conditional` or `rejected`: fix the plan, resend. Proceed only on `approved`. An internal plan-review subagent is not a substitute for this step.

## Step 5: Delegate implementation to the worker pane

Prompt the worker pane with the plan file path, the checkout path, and the repo's rules you collected in Step 1 (verification command, commit rules). The worker commits; the planner does not. Wait with `herdr agent wait` in the background; `blocked` means the worker is waiting for an approval — report it, do not answer for the user.

Every later round of implementation — review fixes, rework — also goes to the worker pane. Do not route it to an executor subagent inside the planner pane: the user cannot see that work.

## Step 6: Diff review, two sources, both must pass

1. **Reviewer pane.** Prompt it with the diff range (`origin/<base>...HEAD`), the repo's review contract if it has one, the ticket decisions, the places to attack, read-only, and a screen-output format (not a file — file writes trigger sandbox approvals in Codex).
2. **CodeRabbit CLI** when `command -v coderabbit` (or `cr`) succeeds:
   ```
   coderabbit review --base <base> --committed --agent
   coderabbit review findings   # re-read the last run
   ```
   Not installed: say so and continue with the reviewer pane alone.

The planner triages every finding as real or false positive, sends the real ones to the worker pane, then **re-runs both reviews**. Loop until both pass.

## Step 7: Open the PR

Push the branch and open the PR against the base branch, following the repo's title format and PR template. planter does not own PR creation; use the repo's or your own PR skill if one exists, otherwise `gh pr create`. If push or `gh` fails with a permission or "repository not found" error, that is the user's account setup — report it, do not switch accounts.

## Recovery

- Codex in the reviewer pane ends `agent prompt` with `agent_prompt_stalled` and the screen shows an empty input box: Codex refused the turn with `invalid cwd` because its background daemon still holds a path from before the worktree was recreated. Send `/quit`, then `herdr pane run <pane_id> "cd <checkout_path> && codex"`, choose "Run without daemon this time" on the daemon dialog, and resend the prompt.
- Before injecting, read the pane (`--source visible --lines 20`). If the user is mid-typing there, wait.

## Common Mistakes

- Calling ExitPlanMode at the end of planning — that means "I will implement this"; the plan goes to the worker pane instead.
- Implementing or reviewing with subagents inside the planner pane instead of the worker/reviewer panes.
- Listing panes without `--workspace "$HERDR_WORKSPACE_ID"` and prompting a same-labeled pane in another workspace.
- Delegating on a `conditional` review — fix the plan until it reads `approved`.
- Asking the user before reading the issue comments, repo docs, and reference source.
- Letting the planner commit or push code it wrote itself.
