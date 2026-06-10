---
name: delegate
description: Use when the user asks to delegate, hand off, or run a task in another terminal session, window, pane, surface, or workspace — e.g. "run this in the build session", "build 세션에서 돌려줘", "delegate this to a new session" — on tmux or cmux
---

# Delegating Work to a tmux/cmux Session

## Overview

Hand a task to a named session under tmux or cmux: detect the live multiplexer from real state, find or create the named target, inspect it, then route to a runner skill. Never assume environment state — verify each step with a real command.

## Step 1: Detect the multiplexer

| Check | Meaning |
|---|---|
| `$TMUX` set | running inside tmux |
| `$CMUX_WORKSPACE_ID` set | running inside a cmux terminal |
| `tmux ls` exits 0 | a tmux server is alive |
| `cmux ping` exits 0 | the cmux app is alive |

Priority: backend the user explicitly named > innermost mux you are running inside (`$TMUX` wins when both are set) > whichever live server responds. If nothing responds, STOP and report that delegation is impossible — never start a tmux/cmux server yourself.

## Step 2: Find or create the target

tmux:
- Exists? `tmux has-session -t <name> 2>/dev/null`
- Create: `tmux new-session -d -s <name> -c <cwd>`
- Finer targets: `-t <session>:<window>.<pane>`. Indexes shift when windows/panes close — resolve them immediately before sending, never from memory.

cmux:
- Exists? Match the title in `cmux list-workspaces`; surface-level targets via `cmux tree`.
- Create a workspace: `cmux new-workspace --name <name> --cwd <cwd>`
- Add a surface to an existing workspace: `cmux new-surface --workspace <ref>` then `cmux rename-tab --surface <ref> <name>`
- Prefer UUID refs for cmux targets (`--id-format uuids`); short refs like `surface:<n>` shift when surfaces close — re-resolve them at send time.

## Step 3: Inspect before injecting

Capture the target screen FIRST:
- tmux: `tmux capture-pane -p -t <target>`
- cmux: `cmux read-screen --workspace <ref> --surface <ref>`

If a foreground program (editor, REPL, running job) occupies the target, STOP — report what is on screen and ask before touching it.

## Step 4: Route to a runner skill

| Request | Runner |
|---|---|
| default, or "claude" | planter:delegate-claude |
| "codex" | planter:delegate-codex |
| plain shell command / "shell" | planter:delegate-shell |

## Monitoring contract (used by all runners)

Default: after injection, poll the target screen every 15–30 seconds. The task is done when the runner's completion signal fires AND two consecutive captures are identical. Then report a summary of the final output (use scrollback: `tmux capture-pane -p -S -200 -t <target>` / `cmux read-screen --workspace <ref> --surface <ref> --scrollback`). If 10 minutes pass without completion, report interim status before continuing.

Fire-and-forget (only when the user asks): confirm the payload landed with one capture, then stop — no polling, no result report.

## Common Mistakes

- Sending keys without capturing the screen first.
- Assuming a session still exists because it existed earlier — re-check at send time.
- Starting a mux server when none responds — report instead.
- Reusing remembered pane indexes — they shift; resolve fresh.
