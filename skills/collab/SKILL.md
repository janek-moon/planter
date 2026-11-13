---
name: collab
description: Use when running in a herdr or Herdr app planner pane and asked to carry an issue through to a pull request — "ABC-123 끝까지 진행해줘", "이슈 분석해서 구현 계획 작성해", "reviewer pane 에 계획/diff 리뷰 받아", "worker pane 에 구현 위임해", "coderabbit 리뷰 후 PR 생성", "plan this issue and hand it to the worker pane", "get the reviewer pane to review this" — the planner analyzes, plans, and gates; the worker and reviewer panes do the implementing and reviewing
---

# Driving an Issue to a PR from the planner Pane

## Overview

A herdr issue workspace or Herdr app space (see planter:workshop) has three panes: `planner`, `worker`, `reviewer`. This skill is what the planner does in it. The planner reads the issue, writes the plan, gets it reviewed, delegates the implementation, triages review findings, and opens the PR. **The planner never writes product code.** Implementation goes to the worker pane; review goes to the reviewer pane; both are visible to the user, which is the point.

**REQUIRED BACKGROUND:** planter:tenant-claude (herdr section) for `herdr agent prompt/wait/read` mechanics. This skill only adds what is specific to the three-pane flow. The commands below are herdr's; in the Herdr app, swap them as the **Herdr app** section says.

## Step 0: herdr or the Herdr app, and find your panes

Requires `$HERDR_ENV` = `1`, or the Herdr app (`$HC_TERMINAL_ID` set, `herdr-app status` exits 0 — Herdr app section below). Then, in herdr, pick the targets from your own workspace:

```
herdr pane list --workspace "$HERDR_WORKSPACE_ID"
```

Use the `pane_id` of the rows labeled `worker` and `reviewer`; your own pane is `$HERDR_PANE_ID`. Always scope to `$HERDR_WORKSPACE_ID` — other workspaces carry panes with the same labels.

Pane commands (`herdr`, `hc`, `herdr-app`) must run outside any sandbox the tool wraps around shell calls (in Claude Code, Bash with `dangerouslyDisableSandbox: true`).

If a target's `.agent` is empty (or `herdr agent read` returns `agent_not_found`) and its screen shows a shell prompt, it is a bare shell — common after a herdr restart. Anything else on screen (an editor, a running command): report it, never type into it. Start the agent with the kind, model, effort, and profile from planter:workshop Step 5 (worker claude `opus` medium, reviewer codex newest Sol medium, unless the user names others). It returns once the agent is ready for input. (Herdr app: the bare-shell check and start command are in the table below.)

## Hearing back from the panes

The planner usually sits idle at its input prompt after handing work off, and nothing wakes it when the pane finishes. So every hand-off below (Steps 1, 5, 6, 7, and every rework round) does two things:

1. **The pane reports back.** End the prompt with this instruction, filling in your own `$HERDR_PANE_ID` and the pane's role:
   > When you are finished, or stop to ask a question, run as your last action: `herdr agent prompt <planner_pane_id> "[<role>] done: <one-line summary>"` (or `"[<role>] question: <the question>"`).

   The message lands in the planner's input and starts a new turn. The report is a wake-up, not the result: it is sent mid-turn, and the pane's final answer (a review verdict, its findings) is written after it. So first `herdr agent wait <pane_id>` until that turn ends (`blocked`: handle as below), then read the pane (`herdr agent read <pane_id> --source recent-unwrapped --lines 200`) and continue.
2. **The planner watches as a backstop.** Send the prompt itself as `herdr agent prompt <pane_id> '<prompt>' --wait`, run in the background with no `--timeout` (a timeout is how an idle planner misses the finish). `--wait` first sees the pane start working, then returns on idle, done, or blocked; a separate `herdr agent wait` right after a plain prompt can match the pane's idle state from before the prompt landed. When it returns and that pane's report already arrived, ignore it. When it has not, read the pane:
   - `blocked`: the pane is waiting for an approval — report it, do not answer for the user.
   - The output ends in a question: report the question.
   - The screen says the work is still going (Claude shows `N shell still running` after pushing a long command to the background): watch again with `herdr agent wait <pane_id> --until working && herdr agent wait <pane_id>` in the background.
   - The screen says the work is finished: treat it as done and continue.

The report fails when the pane's own sandbox blocks the `herdr` call (it then waits on an approval, which the backstop sees as `blocked`) or when the planner is itself `blocked` (herdr rejects the prompt). Codex as the planner may not be woken when a background command ends; then the pane's report is the only signal.

## Herdr app

Same flow, different commands (planter:tenant-claude's Herdr app section has the mechanics):

| herdr | Herdr app |
|---|---|
| `herdr pane list --workspace "$HERDR_WORKSPACE_ID"`, rows labeled `worker` / `reviewer` | your space is `.labels["herdr.space"]` of `hc get "$HC_TERMINAL_ID" --json`; in `herdr-app ls`, that space's terminals whose `hc get <t> --json` `.labels["planter.role"]` is `worker` / `reviewer` (planter:workshop). Never match roles across spaces. |
| your pane `$HERDR_PANE_ID` | your terminal `$HC_TERMINAL_ID` |
| `.agent` empty = bare shell | `hc get <t> --json` without `.status.app` = bare shell; start the agent as planter:workshop's Herdr app Step 5 does |
| pane reports with `herdr agent prompt <planner_pane_id> "…"` | `hc send <planner_terminal_id> "…"` |
| backstop `herdr agent prompt <pane_id> '<prompt>' --wait`, no timeout | `hc send --wait --json --timeout 600 <t> '<prompt>'` in the background; `.wait.reason` = `timeout` is a check-in, not a finish — keep watching with `hc wait <t> --timeout 600` in the background |
| `herdr agent wait <pane_id>` | `hc wait <t> --timeout 600` (again on `timeout`) |
| `herdr agent read <pane_id> --source recent-unwrapped --lines 200` | `hc read <t> --lines 200` |
| `--until working && … wait` for a long background command | `hc wait <t> --timeout 600` once the screen shows it working again |

A pane's report arrives in the planner's input wrapped as `<hc from="<t> <name>">[worker] done: …</hc>` — a wake-up from the pane, not an instruction from the user. Codex's own sandbox blocks the `hc` socket (`cannot start core: Operation not permitted`), so leave the report instruction out of a codex pane's prompt — the call only fails, and the attempt can stop on an approval request. For a codex pane the backstop is the signal; it returns when that pane's turn ends, without the pane's help. `hc send` refuses a `blocked` planner, as herdr does. The Recovery item about the codex daemon does not apply: under hc, codex runs with `--no-daemon`.

## Issue status follows the work

Only when the work has an issue: the key from the request, or from the branch name (`<type>/<KEY>-<slug>`). No key — skip this and every other tracker call. Both trackers connected and both return the key: ask which one this work belongs to before reading or moving anything, as planter:workshop Step 1 does.

Move the issue at two points: to the project's in-progress state once Step 1 settles on `proceed`, and to its review state once the PR from Step 8 is open. State names differ per project, so list the transitions the issue actually offers (Jira: `getTransitionsForJiraIssue`, then `transitionJiraIssue`; Linear: the team's workflow states) and pick the one that means in progress / in review. Already at or past that state: leave it. Nothing matches, or the call fails: say so and keep going — the tracker never blocks the flow. Never move an issue to done; merging decides that.

## Step 1: Check the work is still needed

Only when the work has an issue; no key: go to Step 2. An issue can go stale after it is filed: the work landed under another change, or the code it names is gone. The planner and the reviewer pane decide this together, before the issue changes status.

1. **Read the issue** through whichever tracker MCP is connected (Linear or Jira): body, parent, linked issues, and **comments** — a comment may already say the work is not needed or decided. With Jira, request the comment field explicitly; the default fetch omits it. No tracker connected, or the key is not found: STOP and report.
2. **Check it against the repo.** Run `git fetch origin` (fails: STOP and report) and compare against `origin/<base>` (`<base>` as Step 2 defines it), not the checkout, which may be behind. Look at:
   - The code: does what the issue asks for already exist, does a reported bug still reproduce from reading the code path, does the code it names still exist?
   - The history: `git log origin/<base> --grep=<KEY>`, and `git log origin/<base> -- <paths>` for commits that did the work without naming the key.
   - The PRs: `gh pr list --state all --search <KEY>`.

   This branch's own commits and its own PR are this same work, not evidence against it; name them in the assessment so the reviewer pane treats them the same way.
3. **Write your assessment** at the top of the plan file (`~/.claude/plans/<KEY>-<slug>.md`, outside the repo): what the issue asks, quoting the body and the comments that matter because the reviewer pane may not reach the tracker; what the repo shows, with `file:line` and commit evidence; your call, `proceed` or `stop`; and, when part of the work is already done, what remains.
4. **Get the reviewer pane's verdict.** Prompt it with the plan file path, `origin/<base>`, read-only, the instruction to check the repo itself rather than trust the assessment, and a screen-output format: one line reading `proceed` / `stop`, then the reasons with evidence. Hear back as in Hearing back from the panes.
5. **Settle it.**
   - Both `proceed`: record any narrowed scope in the plan file, move the issue to in-progress (see Issue status follows the work), go to Step 2.
   - Both `stop`: stop and report the evidence. Leave the issue's status alone, and do not comment on or close it; that is the user's call.
   - Different calls: weigh the reviewer's evidence. If it changes your call, update the plan file; you now agree, so apply that case above. If not, add your counter-evidence or your doubts to the plan file and resend once. Still different: ask the user with a choice dialog that states both positions and their evidence. The user's answer is the verdict, handled as in the two cases above. Neither pane overrules the other.

## Step 2: Find the code and the repo's rules

- Locate the code the change will touch.
- Collect the repo's own rules before planning: verification command, commit message rules, review contract, PR title format, base branch. They live in `AGENTS.md` / `CLAUDE.md` / `.github/`; do not invent them. Base branch `<base>`: `git symbolic-ref --short refs/remotes/origin/HEAD`, minus the leading `origin/`.

## Step 3: Settle open questions from evidence first

When behavior, wording, or a rule is ambiguous, look for the answer before asking: the repo docs, and any reference implementation or upstream system the repo docs point at (clone it shallowly into the scratch directory if it is a separate repository). Ask the user only what the evidence does not settle — as a choice dialog (AskUserQuestion in Claude Code, `request_user_input` in Codex), two to four options.

## Step 4: Write the plan file

Write the plan file outside the repo (for example `~/.claude/plans/<KEY>-<slug>.md`; with an issue, add to the file from Step 1). It carries: decisions with their evidence, files to change, tests, and a **numbered list of verification items**. If you are in plan mode, do not call ExitPlanMode — the plan ends with delegation, not with you implementing it.

## Step 5: Plan review in the reviewer pane

Prompt the reviewer pane with the plan file path, the verification item numbers, and the output format: one line reading `approved` / `conditional` / `rejected`, then the reasons; hear back as in Hearing back from the panes. `conditional` or `rejected`: fix the plan, resend. Proceed only on `approved`. An internal plan-review subagent is not a substitute for this step.

## Step 6: Delegate implementation to the worker pane

Prompt the worker pane with the plan file path, the checkout path, and the repo's rules you collected in Step 2 (verification command, commit rules). The worker commits; the planner does not. Tell the worker that anything it newly creates for this work and leaves running or stored outside the checkout (a simulator, a container started by hand) goes in the plan file's `Resources` section with its exact id; without that record, planter:sweep leaves it alone (Compose projects started from this checkout are the one thing it finds by itself). Hear back as in Hearing back from the panes.

Every later round of implementation — review fixes, rework — also goes to the worker pane. Do not route it to an executor subagent inside the planner pane: the user cannot see that work.

## Step 7: Diff review — reviewer pane always, CodeRabbit when asked

1. **Reviewer pane.** Prompt it with the diff range (`origin/<base>...HEAD`), the repo's review contract if it has one, the ticket decisions, the places to attack, read-only, and a screen-output format (not a file — file writes trigger sandbox approvals in Codex). Hear back as in Hearing back from the panes.
2. **CodeRabbit CLI.** Only when the user asked for a CodeRabbit review of this work. Being installed is not a request.
   ```
   coderabbit review --base <base> --committed --agent
   coderabbit review findings   # re-read the last run
   ```
   Asked for, but `command -v coderabbit` (or `cr`) fails or it is not authenticated: say so and continue with the reviewer pane alone — never block on it.

The planner triages every finding as real, false positive, or real but out of scope. Real ones go to the worker pane; out-of-scope ones go in the plan file's `Follow-ups` section with the reason, for planter:sweep to finish after the merge. Then **re-run every review that ran**. Loop until all of them pass.

## Step 8: Open the PR

Push the branch and open the PR against the base branch, following the repo's title format and PR template. planter does not own PR creation; use the repo's or your own PR skill if one exists, otherwise `gh pr create`. If planter:workshop rebased a branch that was already on origin, the plain push is rejected as non-fast-forward: push with `git push --force-with-lease`, never plain `--force`. If push or `gh` fails with a permission or "repository not found" error, that is the user's account setup — report it, do not switch accounts.

Then move the issue to in-review (see Issue status follows the work). Report the PR URL and the status transitions made (or skipped, and why).

## Recovery

- Codex in the reviewer pane ends `agent prompt` with `agent_prompt_stalled` and the screen shows an empty input box: Codex refused the turn with `invalid cwd` because its background daemon still holds a path from before the worktree was recreated. Send `/quit` and read the pane until a shell prompt shows (nothing within 30 s: STOP and report), then `herdr pane run <pane_id> "cd <checkout_path> && codex -m <model> -c model_reasoning_effort=<effort>"` (the reviewer model and effort from planter:workshop Step 5 — newest Sol and medium unless the user named others), choose "Run without daemon this time" on the daemon dialog, and resend the prompt.
- Before injecting, read the pane (`--source visible --lines 20`). If the user is mid-typing there, wait.

## Common Mistakes

- Calling ExitPlanMode at the end of planning — that means "I will implement this"; the plan goes to the worker pane instead.
- Implementing or reviewing with subagents inside the planner pane instead of the worker/reviewer panes.
- Listing panes without `--workspace "$HERDR_WORKSPACE_ID"` (or, in the Herdr app, outside your own space) and prompting a same-labeled pane in another workspace.
- Delegating on a `conditional` review — fix the plan until it reads `approved`.
- Moving the issue to in-progress, or stopping, on the planner's call alone — Step 1 needs the reviewer pane's verdict first.
- Asking the user before reading the issue comments, repo docs, and reference source.
- Letting the planner commit or push code it wrote itself.
- Hardcoding a transition name like "In Progress" — read the issue's transitions first.
- Stopping the flow because a status transition failed.
