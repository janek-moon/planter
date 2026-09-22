# planter 🌱

**English** · [한국어](README.ko.md)

<img width="1536" height="1024" alt="Image" src="https://github.com/user-attachments/assets/b71fc93e-2d49-4869-ac97-e5e1fe1e8769" />

Delegate tasks from a Claude Code or Codex session to named **tmux**, **cmux**, or
**herdr** sessions — find or create the session, run `claude` (default), `codex`, or a
plain shell in it, monitor the screen, and report the result back.

On herdr it also starts work: hand it a Jira or Linear issue and it opens a
worktree in a new workspace named after the issue, with planner, worker, and
reviewer panes waiting in the checkout.

## Install

Claude Code:

```
/plugin marketplace add janek-moon/planter
/plugin install planter@planter
```

Codex:

```
codex plugin marketplace add janek-moon/planter
codex plugin add planter@planter
```

The same skills load in both; start a new session after installing.

## Skills

| Skill | Purpose |
|---|---|
| `planter:tenant` | Entry point: detects tmux/cmux/herdr, finds or creates the named session, routes to a runner |
| `planter:tenant-claude` | Runs the task under the `claude` CLI (default runner) |
| `planter:tenant-codex` | Runs the task under the `codex` CLI |
| `planter:tenant-shell` | Runs a plain shell command with sentinel-based completion detection |
| `planter:issue-worktree` | herdr only: takes a Jira/Linear issue, files one, or works without any issue, then opens a worktree on a conventionally named branch, names the workspace after the work, and lays out planner/worker/reviewer panes |

## Usage

Just ask Claude Code or Codex:

- "Run the tests in the **build** session" / "build 세션에서 테스트 돌려줘"
- "Create a session called **deploy** and have claude fix the lint errors there"
- "Delegate `npm run build` to the build session as a plain shell command"
- "Use codex for this one"
- "Fire and forget — just send it"
- "Start ABC-123" — opens a `feat/ABC-123-…` worktree in a herdr workspace named after the issue
- "File an issue for this and open a worktree for it"
- "Open a worktree for this, no issue needed"

## How it works

1. **Detect** — `$TMUX` / `$CMUX_WORKSPACE_ID` / `$HERDR_ENV` / live-server probes pick the backend; nothing is assumed.
2. **Resolve** — the named session/workspace is reused if it exists, created and named if not.
3. **Inspect** — the target screen is captured before any keystroke is sent; occupied panes are never overwritten.
4. **Run** — the runner launches claude/codex or injects the sentinel-wrapped shell command.
5. **Monitor** — the screen is polled until the completion signal fires and output stabilizes, then the result is summarized. On herdr, the agent state (`idle`/`working`/`blocked`) reported by herdr is used instead of screen polling. Fire-and-forget skips this on request.

### Starting work (herdr only)

1. **Resolve** — the issue key is looked up in whichever tracker MCP is connected, and a key that does not resolve stops the run. With no key you are asked which you want: file an issue (read back to you first) or work without one, leaving the tracker untouched.
2. **Name** — the branch follows the repo's own recent branches (prefix, key casing, slug or none); the base comes from `origin/HEAD`, not a hardcoded `main`.
3. **Open** — one `herdr worktree create` makes the checkout, the workspace, and its label; an existing worktree is reopened instead of duplicated.
4. **Lay out** — planner, worker, and reviewer panes, all in the checkout, left empty for `planter:tenant` to fill.

## Safety rules

- Never boots a tmux/cmux/herdr server on its own.
- Only drives herdr from inside a herdr pane.
- Never types over a pane occupied by another program.
- Never answers trust/permission/login dialogs in the delegated session without explicit user authorization.
- Never files a tracker issue without reading it back first, and never files one to cover an issue key that failed to resolve.

## Requirements

- tmux, cmux, and/or herdr
- Claude Code or Codex with plugin support; `claude` / `codex` CLIs on PATH for those runners
- The issue-or-not choice in `planter:issue-worktree` is a click-to-answer dialog (AskUserQuestion in Claude Code, `request_user_input` in Codex). Codex shows it in Plan mode, or in Default mode once the `default_mode_request_user_input` feature is enabled; otherwise it asks in plain text
- For `planter:issue-worktree`: herdr and a git repo — plus a Jira or Linear MCP when the work comes from an issue

## Manual verification checklist

- [x] Codex: `codex plugin add planter@planter` installs from the marketplace and all five skills appear in the model prompt
- [x] tmux: delegate to a NEW named session (claude runner, monitored)
- [x] tmux: delegate to an EXISTING session (reused, not recreated)
- [ ] cmux: delegate to a new named workspace
- [x] codex runner end-to-end (verified to the approval/usage-limit checkpoint)
- [x] shell runner: sentinel DONE and FAIL paths
- [x] herdr: split + label a pane, shell sentinel DONE/FAIL, claude and codex via `herdr agent`
- [x] herdr: worktree workspace created from a branch + label, reopened by branch, removed
- [x] issue-worktree: worktree workspace opens with planner/worker/reviewer panes in the checkout
- [ ] issue-worktree: Linear and Jira lookup, issue creation path
- [ ] fire-and-forget: injection confirmed, no monitoring afterwards

## License

[MIT](LICENSE)
