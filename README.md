# planter 🌱

Delegate tasks from a Claude Code session to named **tmux** or **cmux**
sessions — find or create the session, run `claude` (default), `codex`, or a
plain shell in it, monitor the screen, and report the result back.

## Install

```
/plugin marketplace add janek-moon/planter
/plugin install planter@planter
```

## Skills

| Skill | Purpose |
|---|---|
| `planter:delegate` | Entry point: detects tmux/cmux, finds or creates the named session, routes to a runner |
| `planter:delegate-claude` | Runs the task under the `claude` CLI (default runner) |
| `planter:delegate-codex` | Runs the task under the `codex` CLI |
| `planter:delegate-shell` | Runs a plain shell command with sentinel-based completion detection |

## Usage

Just ask Claude Code:

- "Run the tests in the **build** session" / "build 세션에서 테스트 돌려줘"
- "Create a session called **deploy** and have claude fix the lint errors there"
- "Delegate `npm run build` to the build session as a plain shell command"
- "Use codex for this one"
- "Fire and forget — just send it"

## How it works

1. **Detect** — `$TMUX` / `$CMUX_WORKSPACE_ID` / live-server probes pick the backend; nothing is assumed.
2. **Resolve** — the named session/workspace is reused if it exists, created and named if not.
3. **Inspect** — the target screen is captured before any keystroke is sent; occupied panes are never overwritten.
4. **Run** — the runner launches claude/codex or injects the sentinel-wrapped shell command.
5. **Monitor** — the screen is polled until the completion signal fires and output stabilizes, then the result is summarized. Fire-and-forget skips this on request.

## Safety rules

- Never boots a tmux/cmux server on its own.
- Never types over a pane occupied by another program.
- Never answers trust/permission/login dialogs in the delegated session without explicit user authorization.

## Requirements

- tmux and/or cmux
- Claude Code with plugin support; `claude` / `codex` CLIs on PATH for those runners

## Manual verification checklist

- [x] tmux: delegate to a NEW named session (claude runner, monitored)
- [x] tmux: delegate to an EXISTING session (reused, not recreated)
- [ ] cmux: delegate to a new named workspace
- [x] codex runner end-to-end (verified to the approval/usage-limit checkpoint)
- [x] shell runner: sentinel DONE and FAIL paths
- [ ] fire-and-forget: injection confirmed, no monitoring afterwards

## License

[MIT](LICENSE)
