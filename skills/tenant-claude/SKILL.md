---
name: tenant-claude
description: Use when a task delegated to a tmux/cmux session should run under the claude CLI — the default planter runner when the user names no other tool
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

## Common Mistakes

- Sending the prompt before the input box renders — keystrokes land in the shell.
- Multi-line prompts via repeated send-keys + Enter — submits the first line alone.
- Auto-answering trust/permission dialogs without user authorization.
