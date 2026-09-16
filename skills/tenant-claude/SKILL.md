---
name: tenant-claude
description: Use when a task delegated to a tmux/cmux/herdr session, window, pane, or surface should run under the claude CLI — "run this with claude in a new split", "새 창에서 claude로 시켜줘", "다른 claude에게 맡겨줘" — the default planter runner when the user names no other tool
---

# Running claude in a Delegated Session

## Overview

Launch or reuse a `claude` instance in the target session, inject the task prompt, then watch the screen for completion.

**REQUIRED BACKGROUND:** planter:tenant defines target resolution and the monitoring contract.

## Step 1: Launch or reuse

Capture the target screen. If claude is already running (input box, or "esc to interrupt"), reuse it. Otherwise:

- tmux: `tmux send-keys -t <target> 'claude' Enter`
- cmux: `cmux send --surface <ref> 'claude'` then `cmux send-key --surface <ref> enter`

Re-capture every 2–3 s (up to ~30 s; this faster poll applies to startup only — steady-state monitoring follows the planter:tenant contract) until the input box renders. If it never does, report the captured screen verbatim.

A trust/permission dialog may render first ("Do you trust the files in this folder?"). Do NOT answer dialogs yourself unless the user authorized it — report what the dialog says.

## Step 2: Inject the prompt

Single-line prompt — literal payload, Enter as a separate event:
- tmux: `tmux send-keys -t <target> -l '<prompt>'` then `tmux send-keys -t <target> Enter`
- cmux: `cmux send --surface <ref> '<prompt>'` then `cmux send-key --surface <ref> enter`

Multi-line prompt — use bracketed paste, never per-line Enter (each Enter would submit early):
- tmux: `printf '%s' "$PROMPT" | tmux load-buffer - && tmux paste-buffer -p -t <target>` then Enter
- cmux: `cmux set-buffer "$PROMPT"` then `cmux paste-buffer --surface <ref>` then enter

Use the buffer method for cmux whenever the prompt contains `\n`/`\r`/`\t` character sequences — `cmux send` interprets those as Enter/Tab keys.

Re-capture to confirm the text sits in the input box BEFORE sending Enter.

## Step 3: Completion and report

Working: spinner / "esc to interrupt" visible. Done: working indicators gone, input box idle — confirm with two identical captures per the monitoring contract. An idle input box is not proof of completion on its own — claude may be asking a follow-up question, which also yields identical captures. Read the captured content: if it ends in a question or a request for input, report that to the user instead of declaring the task done. Report the final response from scrollback (`tmux capture-pane -p -J -S -200 -t <target>`).

If the screen shows a permission request from the delegated claude, report it to the user instead of pressing keys on it.

## herdr

herdr recognizes the agent and reports its state, so Steps 1–3 collapse into agent commands — no startup polling, no capture comparison.

1. **Launch or reuse.** If `herdr pane get <pane_id>` shows `.agent` = `claude`, reuse it (target: its `.name` from `herdr agent list`, or the pane id). Otherwise the pane must be at a shell prompt:
   `herdr agent start <name> --kind claude --pane <pane_id>` — name matches `[a-z][a-z0-9_-]{0,31}`; user flags go after `--`. It returns once claude is ready for input. On `timeout`, read the pane (`herdr pane read <pane_id> --source recent-unwrapped --lines 50`) — usually a trust/permission dialog — and report it; do not answer it.
2. **Inject and wait.** `herdr agent prompt <name> '<prompt>' --wait --timeout 600000`. Multi-line prompts go in as-is — herdr submits via bracketed paste, so no buffer workaround.
3. **Branch on `.result.agent.agent_status`:**
   - `blocked` — a permission request or question dialog. Read it and report; do not send keys.
   - `idle` / `done` — read the result: `herdr agent read <name> --source recent-unwrapped --lines 200`. A follow-up question also settles as `idle`; if the response ends in a question, report it instead of declaring done.
   - error `agent_prompt_stalled` (no state change within 5 s) or `timeout` — read the pane and report interim status.

## Common Mistakes

- Sending the prompt before the input box renders — keystrokes land in the shell.
- Multi-line prompts via repeated send-keys + Enter — submits the first line alone.
- Auto-answering trust/permission dialogs without user authorization.
