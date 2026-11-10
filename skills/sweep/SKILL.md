---
name: sweep
description: Use when work in a herdr or Herdr app workspace is finished and its leftovers should go — "머지했어 정리해줘", "작업 끝났어 리소스 정리해", "워크트리 지워줘", "후속 이슈 만들고 정리해줘", "PR merged, clean up", "tear down this workspace", "remove the worktree and containers for this work"
---

# Sweeping Up Finished Work

## Overview

Run in the planner pane of a finished workspace. Finish what the work left behind (follow-up issues, spec edits), remove what belongs to it (docker, simulators, the worktree, the branch), and close the workspace last. **Nothing is removed unless this work demonstrably owns it, and every failed command stops the sweep before the next destructive step.**

**REQUIRED BACKGROUND:** planter:collab for pane mechanics (Step 0, Hearing back from the panes, the Herdr app table).

## Step 1: Pin the PR

```
gh pr view <branch> --json number,url,state,mergedAt,headRefName,headRefOid,isCrossRepository
```

This runs only in a linked worktree on a branch other than the base. In the main checkout, or on the base branch: STOP and report.

- `gh` answers that no pull request was found: there is no PR. Continue on the no-PR path in Step 2. Any other `gh` error: STOP and report.
- `headRefName` differs from the branch, or `isCrossRepository` is true: STOP and report.
- `OPEN`: STOP and report.
- `MERGED`: continue.
- `CLOSED` without a merge: continue only if the user said to clean up this closed PR, and keep the branch in Step 7. Its commits exist nowhere else.

## Step 2: Prove the work is preserved

Do this before anything is removed or switched.

1. Worker and reviewer panes have finished. Read the screen as well as the state: a pane can read `idle` while a background command runs (Claude shows `N shell still running`). Working, `blocked`, `unknown`, or a background command: STOP and report.
2. With a PR: `git status --porcelain` shows no modified or staged tracked file (any: STOP and report), and `git rev-parse HEAD` equals `headRefOid` (different: STOP and report). A squash merge leaves the branch unmerged in git's eyes, so this match is the proof, not `git branch -d`.
3. Without a PR: `git fetch origin` (fails: STOP and report), then list what the base branch does not have: commits from `git log origin/<base>..HEAD --oneline` (`<base>` as planter:collab Step 2 defines it) and modified or staged tracked files from `git status --porcelain`. These go in the Step 4 warning.
4. Record the branch name and that commit id in the plan file (`~/.claude/plans/<KEY>-<slug>.md`, or `<slug>.md` without an issue; create it if the work has none).

## Step 3: List what is there

**Follow-ups:** the plan file's `Follow-ups` section and anything the user asked for.

**Removable, because this work owns it:**
- Entries in the plan file's `Resources` section, by exact id.
- Docker Compose projects whose `com.docker.compose.project.working_dir` label is this checkout path or a folder inside it: their containers and networks only.
- Agent tool state (`.omc/`, `.serena/`), build output, installed dependencies (`node_modules/`), and caches, as whole folders, because their contents keep changing. Find them in `git status --porcelain --ignored --untracked-files=all` and write them as absolute paths inside this checkout.

**The user's call, listed file by file as absolute paths:** every other untracked or ignored path. An untracked source file may be work nobody committed, and a local database may be unrebuildable. If the user keeps any of them, the sweep ends after Step 6 with the worktree, branch, and workspace in place.

**Kept:** Docker volumes and images, and any simulator or service that existed before this work. Ownership unclear: report it, do not remove it.

## Step 4: One confirmation

Show the removal list and the follow-ups, including the full read-back that planter:issue-worktree's **Filing issues** requires. Remove and file only after the user confirms.

Without a PR, when Step 2 listed anything, show those commits and files first with a plain warning: this work is not on the base branch, and once swept nothing tracks it anymore. Say whether the commits are also on `origin`; the sweep never deletes a remote branch. The user must confirm this warning specifically. If they decline, the sweep ends after Step 6 with the worktree, branch, and workspace in place.

Just before each removal, check that its id still matches the list; a mismatch is a STOP.

## Step 5: Remove what the follow-ups do not need

`docker compose -p <project> down` (no `-v`), then each recorded resource. Keep anything a follow-up still uses.

Without a PR, when the user confirmed the Step 4 warning, it listed modified or staged tracked files, and the user chose to keep nothing in Step 3: compare `git status --porcelain` with that list (different: STOP and ask), then discard them with `git -C "<checkout_path>" reset --hard`.

## Step 6: Finish the follow-ups

- **Issues:** planter:issue-worktree's **Filing issues**.
- **Specs:** where the repo's rules (`AGENTS.md`, `CLAUDE.md`) say they live: this repo, another repo, Notion, Confluence, Obsidian. No rule: ask.
- **A spec in this repo, with tracked changes still in the checkout:** STOP and ask where to make the edit. Switching branches here could disturb what the user kept.
- **A spec in this repo:** the worker pane branches from the fresh `origin/<base>` in this checkout, edits, and commits. The reviewer pane reviews the diff as in planter:collab Step 7. The planner opens the PR. Continue once `gh pr view` shows the PR and its `headRefOid` equals the local HEAD, and record that branch and commit id in the plan file.
- Do not wait for a follow-up PR to merge.
- A follow-up failed: STOP and report, with the worktree still in place.

Never move the original issue to done; merging decides that.

## Step 7: Remove the worktree, then the workspace

`<main_checkout>` is the first `worktree` path in `git worktree list --porcelain`. From here on, every shell call runs in it, because the current directory is about to disappear: set the tool's working directory (Codex `workdir`), or start the command with `cd <main_checkout> &&` (Claude Code).

1. Check the worker and reviewer panes again, screen and state, as in Step 2. They may have run follow-ups since. Anything but finished: STOP and report. Then close them (`herdr pane close <pane_id>`; Herdr app `herdr-app do pane.close --pane <pane>`). Never close the planner here.
2. Compare each branch's current commit with the id recorded in the plan file. Different: STOP and report. This catches a commit added after Step 2.
3. Inspect the checkout, not the main checkout you now run in: `git -C "<checkout_path>" status --porcelain --ignored --untracked-files=all`. A modified or staged tracked file: STOP and report. A path that is neither on the approved list nor inside an approved folder: STOP and ask. `git worktree remove` deletes ignored files without asking, so this comparison is what protects them.
4. Delete the approved untracked paths and remove the checkout in one command, each path absolute and quoted. Never pass `--force`; if git refuses, STOP and report.
   ```
   cd <main_checkout> && rm -rf -- "<checkout_path>/<untracked path>" && git worktree remove <checkout_path>
   ```
5. Delete each recorded branch: `git -C <main_checkout> branch -D <branch>`. The commit match in item 2, plus the PR's head match or the confirmed Step 4 warning, makes this safe. A `CLOSED` PR's branch is the exception: skip this item for it.
6. Append the result to the plan file: PR, follow-up PR and issue links, what was removed, what was kept. Closing the workspace erases the screen, so the file is the record.
7. Anything skipped or kept for the user to decide: report it on screen and leave the workspace open.
8. Otherwise, as the last action: `herdr workspace close <workspace_id>`; Herdr app `herdr-app do space.close --id <space> --confirmed true`.

## Common Mistakes

- Using `git worktree remove --force` to get past a refusal — that is the guard against deleting unlisted untracked files.
- Trusting `git worktree remove` to protect ignored files — it deletes them, so compare the list first.
- Passing `rm` a path relative to the checkout after `cd`-ing to the main checkout — it deletes the main checkout's file of the same name.
- Using `git branch -d` as the merge check — it fails after a squash merge.
- Running shell calls from the checkout after removing it — the next shell may fail to start there.
- Removing a simulator because it was used, rather than because this work created it.
- `docker compose down -v` — volumes hold data this work may not own.
- Closing the workspace before writing the result to the plan file.
