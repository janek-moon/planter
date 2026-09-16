---
name: tenant
description: Use when the user asks to delegate, hand off, or run a task in another terminal session, window, pane, split, surface, or workspace on tmux, cmux, or herdr — e.g. "run this in the build session", "delegate this to a new session", "build 세션에서 돌려줘", "다른 세션에서 실행해줘", "이 서피스/세션에 요청해줘", "옆 패널에서 돌려", "새 창에서 시켜줘", "세션에 위임해줘/맡겨줘/넘겨줘"
---

# Delegating Work to a tmux/cmux/herdr Session

## Overview

Hand a task to a named session under tmux, cmux, or herdr: detect the live multiplexer from real state, find or create the named target, inspect it, then route to a runner skill. Never assume environment state — verify each step with a real command.

## Step 1: Detect the multiplexer

| Check | Meaning |
|---|---|
| `$TMUX` set | running inside tmux |
| `$CMUX_WORKSPACE_ID` set | running inside a cmux terminal |
| `tmux ls` exits 0 | a tmux server is alive |
| `cmux ping` exits 0 | the cmux app is alive |
| `$HERDR_ENV` = `1` | running inside a herdr pane |

Priority: backend the user explicitly named > innermost mux you are running inside (`$TMUX` wins over `$CMUX_WORKSPACE_ID`) > whichever live tmux/cmux server responds. If nothing responds, STOP and report that delegation is impossible — never start a tmux/cmux/herdr server yourself.

herdr is usable ONLY from inside herdr (`$HERDR_ENV` = `1`) — never drive a herdr session from outside it, even if its server is running. When `$HERDR_ENV` and `$TMUX` (or `$CMUX_WORKSPACE_ID`) are both set and the user named neither, you cannot tell which is innermost: ask the user.

## Step 2: Find or create the target — keep it visible

Create the target **where the user can watch it**: a split in the session you are already inside, not a hidden background session. Only fall back to a detached session when you are NOT running inside the multiplexer.

tmux:
- Exists? Match a pane titled `<name>`: `tmux list-panes -a -F '#{pane_title}\t#{pane_id}'` — reuse its `#{pane_id}` if found.
- Create (visible — when inside tmux, `$TMUX` set): split the current window and capture the new pane id:
  `tmux split-window -d -h -c <cwd> -P -F '#{pane_id}'` → prints e.g. `%30`. Title it so it can be found again: `tmux select-pane -t <pane_id> -T <name>`. `-d` keeps your focus put; use `-v` for a top/bottom split.
- Create (fallback — when NOT inside tmux, no current window to split): `tmux new-session -d -s <name> -c <cwd>`, then tell the user to `tmux attach -t <name>` to watch.
- The target is the pane id (`%N`) or `<session>:<window>.<pane>`. Indexes/ids shift when panes close — resolve them immediately before sending, never from memory.

cmux (surfaces are always shown in the app — keep `--focus`, the default, so it surfaces in front):
- Exists? Match the title in `cmux list-workspaces`; surface-level targets via `cmux tree`.
- Create (visible split — when inside a workspace, `$CMUX_WORKSPACE_ID` set): `cmux new-split <left|right|up|down> --focus true` adds a visible split surface to the current workspace; read its ref back from `cmux tree --id-format uuids`.
- Create a new workspace when none fits: `cmux new-workspace --name <name> --cwd <cwd> --focus true` (focus defaults true → it opens in front). Add a tab to an existing workspace instead with `cmux new-surface --workspace <ref> --focus true` then `cmux rename-tab --surface <ref> <name>`.
- Prefer UUID refs for cmux targets (`--id-format uuids`); short refs like `surface:<n>` shift when surfaces close — re-resolve them at send time.

herdr (every command returns JSON — read ids from the response, never predict them):
- Exists? Match `.label` in `herdr pane list --workspace "$HERDR_WORKSPACE_ID"`, or an agent's `.name` in `herdr agent list` (`.name` exists only for agents started or renamed with a name). Reuse its `pane_id`; the pane's `.agent` says whether an agent already occupies it.
- Create (visible split of the current tab): check the caller's size with `herdr pane layout --pane "$HERDR_PANE_ID"` — split `right` when wide, `down` when narrow/tall, unless the user named a direction:
  `herdr pane split --current --direction <right|down> --cwd "$PWD" --no-focus` → new id at `.result.pane.pane_id` (e.g. `wQ:p3`). Label it: `herdr pane rename <pane_id> <name>`. `--no-focus` keeps the user's focus put.
- Do NOT create a workspace, tab, or worktree unless the user asked for that topology.
- Always target `--current` or an explicit pane id / agent name — omitting the target may hit the pane another client has focused.

## Step 3: Inspect before injecting

Capture the target screen FIRST:
- tmux: `tmux capture-pane -p -t <target>`
- cmux: `cmux read-screen --workspace <ref> --surface <ref>`
- herdr: `herdr pane read <pane_id> --source recent-unwrapped --lines 50` (plain text, not JSON); `herdr pane get <pane_id>` shows whether an agent occupies it (`.agent`, `.agent_status`)

If a foreground program (editor, REPL, running job) occupies the target, STOP — report what is on screen and ask before touching it.

## Step 4: Route to a runner skill

| Request | Runner |
|---|---|
| default, or "claude" | planter:tenant-claude |
| "codex" | planter:tenant-codex |
| plain shell command / "shell" | planter:tenant-shell |

## Monitoring contract (used by all runners)

Default: after injection, poll the target screen every 15–30 seconds. The task is done when the runner's completion signal fires AND two consecutive captures are identical. Then report a summary of the final output (use scrollback: `tmux capture-pane -p -S -200 -t <target>` / `cmux read-screen --workspace <ref> --surface <ref> --scrollback`). If 10 minutes pass without completion, report interim status before continuing.

herdr: do not compare captures — herdr reports state. Agents: wait on `agent_status` (`idle`/`done` = settled, `blocked` = approval or question dialog, `working`); shell: wait for the sentinel with `herdr pane wait-output`. Use `--timeout 600000` (10 minutes); on `timeout`, report interim status, then wait again. Scrollback for the report: `herdr pane read <pane_id> --source recent-unwrapped --lines 200`.

Fire-and-forget (only when the user asks): confirm the payload landed with one capture, then stop — no polling, no result report.

## Common Mistakes

- Sending keys without capturing the screen first.
- Assuming a session still exists because it existed earlier — re-check at send time.
- Starting a mux server when none responds — report instead.
- Driving herdr from outside a herdr pane (`$HERDR_ENV` unset).
- Reusing remembered pane indexes — they shift; resolve fresh.
