# planter — Session Delegation Plugin Design

**Date:** 2026-06-10
**Status:** Approved

## Overview

planter is a Claude Code plugin (MIT, open source) that delegates work to named
terminal sessions running under tmux or cmux. When the user asks to hand a task
to a specific session ("build 세션에서 테스트 돌려줘"), the plugin detects which
multiplexer is available, finds or creates the named session, launches the
requested runner (claude by default, codex or a plain shell on request), injects
the task, and monitors progress until it can report results.

## Goals

- Delegate a task to a named session on tmux **or** cmux, detected from the
  actual runtime environment (never assumed).
- Reuse an existing session/surface/pane when the requested name exists;
  otherwise create one and name it.
- Default runner is `claude`; `codex` and plain `shell` are selectable.
- Default behavior is monitor-and-report; fire-and-forget on request.
- Package each concern as its own skill inside one plugin, installable from a
  git repo via marketplace.json.

## Non-Goals

- No support for multiplexers other than tmux and cmux.
- No daemon or background service; all orchestration happens through skill
  instructions executed by the calling Claude session.
- No automated test harness for the skills themselves (they are markdown
  instructions); verification is a manual scenario checklist.

## Repository Layout

```
planter/
├── .claude-plugin/
│   ├── plugin.json          # name: planter, license: MIT
│   └── marketplace.json     # allows /plugin marketplace add <repo-url>
├── skills/
│   ├── delegate/SKILL.md          # entry point (router)
│   ├── delegate-claude/SKILL.md   # claude runner (default)
│   ├── delegate-codex/SKILL.md    # codex runner
│   └── delegate-shell/SKILL.md    # plain shell runner
├── LICENSE                  # MIT
├── README.md                # English; install, usage, scenarios
└── .gitignore
```

## Skill: `delegate` (entry point)

Trigger: the user asks to delegate/hand off work to a (named) session, pane,
surface, or workspace.

Workflow:

1. **Detect environment** — check real state, never assume:
   - `$TMUX` set → running inside tmux.
   - `$CMUX_WORKSPACE_ID` set → running inside a cmux terminal.
   - Neither set → probe live servers with `tmux ls` and `cmux ping`.
   - Backend priority: user-specified backend > innermost mux the current
     session runs inside (`$TMUX` wins over `$CMUX_*` when both are set) >
     whichever live server responds. If none respond, stop and report that
     delegation is impossible (do not attempt to boot a mux).
2. **Resolve target session** by the requested name:
   - tmux: `tmux has-session -t <name>`; finer targets as
     `session:window.pane`.
   - cmux: match workspace titles from `cmux list-workspaces`; surface-level
     targets via `cmux tree` / `cmux list-pane-surfaces`.
3. **Create if missing**, with the requested name:
   - tmux: `tmux new-session -d -s <name> -c <cwd>`.
   - cmux: `cmux new-workspace --name <name> --cwd <cwd>`; if the user pointed
     inside an existing workspace, `cmux new-surface` + `cmux rename-tab`.
4. **Inspect before injecting** — capture the target screen first
   (`tmux capture-pane -p -t <target>` / `cmux read-screen`). If a foreground
   program (editor, REPL, running job) occupies the pane, do not inject; report
   to the user and ask how to proceed.
5. **Route to a runner skill** — `delegate-claude` by default;
   `delegate-codex` or `delegate-shell` when the user asks for codex / a plain
   shell command.

## Runner Skills

Common rules:

- Send text with literal-safe mechanisms: `tmux send-keys -l` for the payload
  followed by a separate `Enter` key event; `cmux send` + `cmux send-key`.
- Multi-line prompts are sent as a single literal payload; never rely on shell
  interpolation inside the target pane.
- After injection, re-capture the screen to confirm the payload landed before
  claiming the task was delegated.

### `delegate-claude` (default)

- If the target pane already runs claude (detected from the captured screen),
  reuse it; otherwise launch `claude` and wait for the input prompt to render.
- Inject the task prompt, then Enter.
- Completion heuristic: working indicators ("esc to interrupt", spinner)
  disappear and the input prompt returns to idle.

### `delegate-codex`

- Same pattern with the `codex` CLI: launch or reuse, wait for prompt, inject,
  detect idle prompt for completion.

### `delegate-shell`

- Inject the shell command directly, appending a sentinel:
  `<cmd> && echo PLANTER_DONE_<id> || echo PLANTER_FAIL_<id>` where `<id>` is a
  short unique token chosen at delegation time.
- Completion: sentinel string appears in capture output. This is the most
  reliable detector and requires no prompt heuristics.

## Monitoring (default) and fire-and-forget

- Default: poll the target screen every 15–30 seconds. Consider the task done
  when the runner-specific completion signal fires **and** two consecutive
  captures are identical (output stabilized). Then summarize the final output
  back to the user.
- Soft timeout: after 10 minutes without completion, report interim status and
  decide (with the user if interactive) whether to keep waiting.
- Fire-and-forget: when the user says so ("보내기만 해", "fire and forget"),
  verify injection landed, then stop — no polling, no result report.

## Error Handling

- No live mux server → report; never try to start tmux/cmux.
- Session create/name collision failures → report the actual command error.
- Target pane occupied by another program → never overwrite; surface to the
  user.
- Capture shows the runner crashed or printed an error → report the captured
  output verbatim rather than a guess.

## Packaging & Distribution

- `.claude-plugin/plugin.json`: name `planter`, version `0.1.0`, MIT license,
  description, keywords.
- `.claude-plugin/marketplace.json`: single-plugin marketplace pointing at the
  repo root so users can `/plugin marketplace add <github-url>` then
  `/plugin install planter@planter`.
- MIT `LICENSE`, English `README.md` (install, usage examples per runner,
  fire-and-forget, verification checklist), `.gitignore`.
- Local git history prepared; pushing to GitHub is done by the user.

## Verification Checklist (manual)

1. tmux: delegate to a new named session, claude runner, monitored.
2. tmux: delegate to an existing session (reuse, no duplicate creation).
3. cmux: delegate to a new named workspace, claude runner.
4. codex runner end-to-end on either mux.
5. shell runner with sentinel completion detection.
6. fire-and-forget: injection verified, no monitoring afterwards.
