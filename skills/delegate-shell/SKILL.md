---
name: delegate-shell
description: Use when a task delegated to a tmux/cmux session is a plain shell command, or the user asks for a plain shell instead of an AI agent in the target session
---

# Running a Shell Command in a Delegated Session

## Overview

Inject a shell command into the target session with a sentinel suffix so completion and success/failure are detected from the screen, not guessed.

**REQUIRED BACKGROUND:** planter:delegate defines target resolution and the monitoring contract.

## Step 1: Verify a shell prompt

Capture the target (`tmux capture-pane -p -t <target>` / `cmux read-screen --workspace <ref> --surface <ref>`). The last line must be a shell prompt. If another program is running, STOP and report.

## Step 2: Inject with a sentinel

Pick a unique id (e.g. `id=$(date +%s)`) — resolve it YOURSELF before building the payload; the string you send must contain the literal id, never an unresolved `$(...)`. Wrap the command:

```
<command> && echo "PLANTER_DONE_<id>" || echo "PLANTER_FAIL_<id>"
```

Send it literally, then Enter as a separate event:

- tmux: `tmux send-keys -t <target> -l '<wrapped command>'` then `tmux send-keys -t <target> Enter`
- cmux: `cmux send --surface <ref> '<wrapped command>'` then `cmux send-key --surface <ref> enter`

The payload for `cmux send` must stay single-line with no `\n`/`\r`/`\t` two-character escape sequences — cmux interprets those as Enter/Tab keys and would submit mid-string.

Re-capture once and confirm the typed command is visible at the prompt (or output has started) before reporting it as delegated.

## Step 3: Detect completion

Poll per the monitoring contract, reading scrollback:

```
tmux capture-pane -p -J -S -200 -t <target> | grep -E '^PLANTER_(DONE|FAIL)_<id>$'
```

```
cmux read-screen --workspace <ref> --surface <ref> --scrollback | grep -E '^PLANTER_(DONE|FAIL)_<id>$'
```

**Anchor the match to the whole line.** The typed command itself also contains the sentinel text; only the echoed result appears alone on its own line. `-J` joins wrapped lines, so a wrapped fragment of the long typed command cannot start a physical line and false-match.

## Step 4: Report

Report DONE/FAIL plus the output lines between the command and the sentinel. On FAIL, include the error output verbatim — do not paraphrase.

Fire-and-forget: skip Steps 3–4 (see planter:delegate).

## Common Mistakes

- Matching the sentinel without line anchors and "detecting" completion instantly from the echoed command line.
- Joining the command and Enter in one send — quoting bugs swallow the newline.
- Forgetting that `$(...)` inside the payload expands in the TARGET shell — intentional only if the user's command wants that.
