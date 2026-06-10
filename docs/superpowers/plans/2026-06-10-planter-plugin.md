# planter Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `planter` Claude Code plugin: four skills that delegate tasks to named tmux/cmux sessions (claude/codex/shell runners), packaged as an MIT open-source repo.

**Architecture:** One entry-point skill (`delegate`) detects the live multiplexer, finds/creates the named session, and routes to one of three runner skills (`delegate-claude`, `delegate-codex`, `delegate-shell`). Skills are markdown instructions; every skill is verified RED→GREEN with subagent scenarios against a real tmux server (cmux spot-checked if the app is running).

**Tech Stack:** Claude Code plugin format (`.claude-plugin/plugin.json` + `skills/*/SKILL.md`), tmux 3.x CLI, cmux CLI, `claude` / `codex` CLIs.

**Spec:** `docs/superpowers/specs/2026-06-10-planter-design.md`

**Working directory:** `/Users/janek/Documents/dev/planter` (git repo already initialized, branch `main`).

**Test-session naming:** every test session is named `planter-test-*` so cleanup is `tmux kill-session -t <name>` per session; never touch other sessions.

---

### Task 1: Plugin scaffold (metadata, license, gitignore)

**Files:**
- Create: `.gitignore`
- Create: `LICENSE`
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Write `.gitignore`**

```
.DS_Store
.omc/
```

- [ ] **Step 2: Write `LICENSE` (MIT)**

```
MIT License

Copyright (c) 2026 Janek

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 3: Write `.claude-plugin/plugin.json`**

```json
{
  "name": "planter",
  "description": "Delegate tasks to named tmux/cmux sessions: find or create the session, run claude, codex, or a plain shell, then monitor and report back",
  "version": "0.1.0",
  "author": {
    "name": "Janek",
    "email": "eunho.moon@gmail.com"
  },
  "homepage": "https://github.com/janek-moon/planter",
  "repository": "https://github.com/janek-moon/planter",
  "license": "MIT",
  "keywords": ["tmux", "cmux", "delegation", "session", "claude", "codex", "shell"]
}
```

- [ ] **Step 4: Write `.claude-plugin/marketplace.json`**

```json
{
  "name": "planter",
  "owner": {
    "name": "Janek",
    "email": "eunho.moon@gmail.com"
  },
  "plugins": [
    {
      "name": "planter",
      "source": "./",
      "description": "Delegate tasks to named tmux/cmux sessions: find or create the session, run claude, codex, or a plain shell, then monitor and report back"
    }
  ]
}
```

- [ ] **Step 5: Validate both JSON files**

Run: `jq . .claude-plugin/plugin.json && jq . .claude-plugin/marketplace.json`
Expected: both pretty-print without error (exit 0).

- [ ] **Step 6: Commit**

```bash
git add .gitignore LICENSE .claude-plugin
git commit -m "feat: scaffold planter plugin (metadata, MIT license)"
```

---

### Task 2: `delegate` skill (entry point / router)

**Files:**
- Create: `skills/delegate/SKILL.md`

- [ ] **Step 1: RED — baseline scenario without the skill**

Dispatch a `general-purpose` subagent with EXACTLY this prompt (no mention of any skill):

> You are running inside tmux on macOS. Delegate this task to a terminal session named `planter-base-a`: run `echo hello-from-base`. Do whatever you would naturally do, then report the steps you took.

Document verbatim in working notes: Did it check `$TMUX` or probe `tmux ls`? Did it run `tmux has-session` before creating? Did it capture the pane BEFORE sending keys? Did it verify the command actually ran? Expected baseline failures (any of): skips capture-before-send, recreates an existing session, sends command and reports success without re-capturing.

- [ ] **Step 2: Clean up baseline session**

Run: `tmux kill-session -t planter-base-a 2>/dev/null; true`

- [ ] **Step 3: GREEN — write `skills/delegate/SKILL.md`**

````markdown
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

Default: after injection, poll the target screen every 15–30 seconds. The task is done when the runner's completion signal fires AND two consecutive captures are identical. Then report a summary of the final output (use scrollback: `tmux capture-pane -p -S -200 -t <target>` / `cmux read-screen --scrollback`). If 10 minutes pass without completion, report interim status before continuing.

Fire-and-forget (only when the user asks): confirm the payload landed with one capture, then stop — no polling, no result report.

## Common Mistakes

- Sending keys without capturing the screen first.
- Assuming a session still exists because it existed earlier — re-check at send time.
- Starting a mux server when none responds — report instead.
- Reusing remembered pane indexes — they shift; resolve fresh.
````

- [ ] **Step 4: Verify — same scenario WITH the skill**

Dispatch a fresh `general-purpose` subagent:

> Read /Users/janek/Documents/dev/planter/skills/delegate/SKILL.md and follow it exactly. Scenario: the user asks to delegate a task (runner not yet specified) to a tmux session named `planter-test-a` with working directory `/tmp`. Perform Steps 1–3 of the skill, then STOP before invoking any runner skill (they do not exist yet). Report: which multiplexer you detected and why, whether the session existed, the exact commands you ran in order, and which runner skill Step 4 would route to for a default request.

Verify after it returns:
- Run: `tmux has-session -t planter-test-a; echo $?` → Expected: `0`
- Subagent report shows: detection via `$TMUX`/probe BEFORE session commands; `has-session` BEFORE `new-session`; `capture-pane` ran; routing answer is `planter:delegate-claude`.

If any check fails: identify the gap or loophole in the skill text, fix SKILL.md, re-run this step with a fresh subagent (REFACTOR loop).

- [ ] **Step 5: Clean up**

Run: `tmux kill-session -t planter-test-a 2>/dev/null; true`

- [ ] **Step 6: Commit**

```bash
git add skills/delegate
git commit -m "feat: add delegate skill (mux detection, session resolution, routing)"
```

---

### Task 3: `delegate-shell` skill

**Files:**
- Create: `skills/delegate-shell/SKILL.md`

- [ ] **Step 1: RED — baseline without the skill**

Dispatch a `general-purpose` subagent:

> You are inside tmux on macOS. A session named `planter-base-b` may or may not exist; create it if needed (cwd /tmp). Send it the shell command `sleep 3 && ls /tmp | head -3`, wait until it finishes, and report whether it succeeded and what it printed. Report the exact commands you used.

Document verbatim: how did it detect completion? Expected baseline failures: fixed `sleep` guesses instead of a completion signal, no success/failure distinction, matching output without anchoring.

- [ ] **Step 2: Clean up baseline session**

Run: `tmux kill-session -t planter-base-b 2>/dev/null; true`

- [ ] **Step 3: GREEN — write `skills/delegate-shell/SKILL.md`**

````markdown
---
name: delegate-shell
description: Use when a task delegated to a tmux/cmux session is a plain shell command, or the user asks for a plain shell instead of an AI agent in the target session
---

# Running a Shell Command in a Delegated Session

## Overview

Inject a shell command into the target session with a sentinel suffix so completion and success/failure are detected from the screen, not guessed.

**REQUIRED BACKGROUND:** planter:delegate defines target resolution and the monitoring contract.

## Step 1: Verify a shell prompt

Capture the target (`tmux capture-pane -p -t <target>` / `cmux read-screen`). The last line must be a shell prompt. If another program is running, STOP and report.

## Step 2: Inject with a sentinel

Pick a unique id (e.g. `id=$(date +%s)`). Wrap the command:

```
<command> && echo "PLANTER_DONE_<id>" || echo "PLANTER_FAIL_<id>"
```

Send it literally, then Enter as a separate event:

- tmux: `tmux send-keys -t <target> -l '<wrapped command>'` then `tmux send-keys -t <target> Enter`
- cmux: `cmux send --surface <ref> '<wrapped command>'` then `cmux send-key --surface <ref> Enter`

Re-capture once to confirm the command line landed before reporting it as delegated.

## Step 3: Detect completion

Poll per the monitoring contract, reading scrollback:

```
tmux capture-pane -p -S -200 -t <target> | grep -E '^PLANTER_(DONE|FAIL)_<id>$'
```

**Anchor the match to the whole line.** The typed command itself also contains the sentinel text; only the echoed result appears alone on its own line.

## Step 4: Report

Report DONE/FAIL plus the output lines between the command and the sentinel. On FAIL, include the error output verbatim — do not paraphrase.

## Common Mistakes

- Matching the sentinel without line anchors and "detecting" completion instantly from the echoed command line.
- Joining the command and Enter in one send — quoting bugs swallow the newline.
- Forgetting that `$(...)` inside the payload expands in the TARGET shell — intentional only if the user's command wants that.
````

- [ ] **Step 4: Verify — end-to-end DONE path**

Dispatch a fresh `general-purpose` subagent:

> Read /Users/janek/Documents/dev/planter/skills/delegate/SKILL.md and /Users/janek/Documents/dev/planter/skills/delegate-shell/SKILL.md and follow them exactly. Delegate the shell command `sleep 3 && ls /tmp | head -3` to a tmux session named `planter-test-b` (create if missing, cwd /tmp). Monitor to completion and report the result, plus the exact commands you ran in order.

Verify: report shows anchored sentinel grep (`^PLANTER_DONE_...$`), capture-before-send, separate Enter send, and the actual `ls` output lines.

- [ ] **Step 5: Verify — FAIL path**

Dispatch a fresh `general-purpose` subagent with the same prompt but the command `ls /nonexistent-planter-dir` and session `planter-test-b` (now it exists — must be REUSED, not recreated). Verify: report says the command FAILED, includes the real `ls` error line, and the subagent did not run `tmux new-session` again.

- [ ] **Step 6: Clean up**

Run: `tmux kill-session -t planter-test-b 2>/dev/null; true`

- [ ] **Step 7: Commit**

```bash
git add skills/delegate-shell
git commit -m "feat: add delegate-shell skill (sentinel-based completion)"
```

---

### Task 4: `delegate-claude` skill

**Files:**
- Create: `skills/delegate-claude/SKILL.md`

- [ ] **Step 1: RED — baseline without the skill**

Dispatch a `general-purpose` subagent:

> You are inside tmux on macOS. Start the `claude` CLI in a new tmux session named `planter-base-c` (cwd /Users/janek/Documents/dev/planter), have it answer the prompt "Say PLANTER_OK and nothing else", capture its answer, then exit it. Report the exact commands you used. You are authorized to accept a workspace-trust dialog if one appears.

Document verbatim. Expected baseline failures: types the prompt before claude's input box renders (keystrokes land in the shell), uses one `send-keys 'prompt' Enter` call with quoting issues, declares success without re-capturing the answer.

- [ ] **Step 2: Clean up baseline session**

Run: `tmux kill-session -t planter-base-c 2>/dev/null; true`

- [ ] **Step 3: GREEN — write `skills/delegate-claude/SKILL.md`**

````markdown
---
name: delegate-claude
description: Use when a task delegated to a tmux/cmux session should run under the claude CLI — the default planter runner when the user names no other tool
---

# Running claude in a Delegated Session

## Overview

Launch or reuse a `claude` instance in the target session, inject the task prompt, then watch the screen for completion.

**REQUIRED BACKGROUND:** planter:delegate defines target resolution and the monitoring contract.

## Step 1: Launch or reuse

Capture the target screen. If claude is already running (input box, or "esc to interrupt"), reuse it. Otherwise:

- tmux: `tmux send-keys -t <target> 'claude' Enter`
- cmux: `cmux send --surface <ref> 'claude'` then `cmux send-key --surface <ref> Enter`

Re-capture every 2–3 s (up to ~30 s) until the input box renders. If it never does, report the captured screen verbatim.

A trust/permission dialog may render first ("Do you trust the files in this folder?"). Do NOT answer dialogs yourself unless the user authorized it — report what the dialog says.

## Step 2: Inject the prompt

Single-line prompt — literal payload, Enter as a separate event:
- tmux: `tmux send-keys -t <target> -l '<prompt>'` then `tmux send-keys -t <target> Enter`
- cmux: `cmux send --surface <ref> '<prompt>'` then `cmux send-key --surface <ref> Enter`

Multi-line prompt — use bracketed paste, never per-line Enter (each Enter would submit early):
- tmux: `printf '%s' "$PROMPT" | tmux load-buffer - && tmux paste-buffer -p -t <target>` then Enter
- cmux: `cmux set-buffer "$PROMPT"` then `cmux paste-buffer --surface <ref>` then Enter

Re-capture to confirm the text sits in the input box BEFORE sending Enter.

## Step 3: Completion and report

Working: spinner / "esc to interrupt" visible. Done: working indicators gone, input box idle — confirm with two identical captures per the monitoring contract. Report the final response from scrollback (`tmux capture-pane -p -S -200 -t <target>`).

If the screen shows a permission request from the delegated claude, report it to the user instead of pressing keys on it.

## Common Mistakes

- Sending the prompt before the input box renders — keystrokes land in the shell.
- Multi-line prompts via repeated send-keys + Enter — submits the first line alone.
- Auto-answering trust/permission dialogs without user authorization.
````

- [ ] **Step 4: Verify — end-to-end**

Dispatch a fresh `general-purpose` subagent:

> Read /Users/janek/Documents/dev/planter/skills/delegate/SKILL.md and /Users/janek/Documents/dev/planter/skills/delegate-claude/SKILL.md and follow them exactly. Delegate this task to a tmux session named `planter-test-c` (create if missing, cwd /Users/janek/Documents/dev/planter) using the claude runner: the prompt is "Say PLANTER_OK and nothing else". Monitor to completion, report claude's answer, then send `/exit` to close claude. Exception granted by the user for this test: if a workspace-trust dialog appears, you may accept it. Report the exact commands you ran in order.

Verify: report shows wait-for-input-box loop before injection, `-l` literal send with separate Enter, re-capture before Enter, and the captured answer contains `PLANTER_OK`.

- [ ] **Step 5: Clean up**

Run: `tmux kill-session -t planter-test-c 2>/dev/null; true`

- [ ] **Step 6: Commit**

```bash
git add skills/delegate-claude
git commit -m "feat: add delegate-claude skill (default runner)"
```

---

### Task 5: `delegate-codex` skill

**Files:**
- Create: `skills/delegate-codex/SKILL.md`

- [ ] **Step 1: RED — baseline without the skill**

Dispatch a `general-purpose` subagent:

> You are inside tmux on macOS. Start the `codex` CLI in a new tmux session named `planter-base-d` (cwd /tmp), have it answer "Reply with exactly PLANTER_OK", and report what happened with the exact commands you used. If codex shows a login screen, stop and say so.

Document verbatim. Expected baseline failures: typing into a login/auth screen, no ready-signal wait.

- [ ] **Step 2: Clean up baseline session**

Run: `tmux kill-session -t planter-base-d 2>/dev/null; true`

- [ ] **Step 3: GREEN — write `skills/delegate-codex/SKILL.md`**

````markdown
---
name: delegate-codex
description: Use when the user asks to run a delegated task with codex (OpenAI Codex CLI) in a tmux/cmux session
---

# Running codex in a Delegated Session

## Overview

Same flow as planter:delegate-claude with the `codex` CLI: launch or reuse, wait for the composer, inject, watch for idle.

**REQUIRED BACKGROUND:** planter:delegate (target resolution, monitoring contract); planter:delegate-claude (injection mechanics — identical, including bracketed paste for multi-line prompts).

## Codex specifics

- Launch with `codex` (append user-requested flags verbatim).
- Ready signal: the composer input box renders.
- If a login/auth screen appears instead, STOP and report — never start an auth flow on the user's behalf.
- Working indicator: codex shows a spinner/working status; done when the composer is idle again — confirm with two identical captures.
- Codex may render approval prompts for commands; report them to the user, do not answer them yourself.

## Common Mistakes

- Treating the login screen as a ready composer and typing the task into it.
- Submitting multi-line prompts line-by-line (same early-submit failure as claude).
````

- [ ] **Step 4: Verify — end-to-end (or auth-stop path)**

Dispatch a fresh `general-purpose` subagent:

> Read these three files and follow them exactly: /Users/janek/Documents/dev/planter/skills/delegate/SKILL.md, /Users/janek/Documents/dev/planter/skills/delegate-claude/SKILL.md, /Users/janek/Documents/dev/planter/skills/delegate-codex/SKILL.md. Delegate this task to a tmux session named `planter-test-d` (create if missing, cwd /tmp) using the codex runner: the prompt is "Reply with exactly PLANTER_OK". Monitor to completion and report codex's answer, then exit codex. If a login/auth screen appears, follow the skill. Report the exact commands you ran in order.

Verify (either outcome is a pass):
- Authenticated codex: answer contains `PLANTER_OK`, ready-signal wait happened before injection.
- Login screen: subagent STOPPED and reported it without typing into it.

- [ ] **Step 5: Clean up**

Run: `tmux kill-session -t planter-test-d 2>/dev/null; true`

- [ ] **Step 6: Commit**

```bash
git add skills/delegate-codex
git commit -m "feat: add delegate-codex skill"
```

---

### Task 6: README, cmux spot-check, final review

**Files:**
- Create: `README.md`

- [ ] **Step 1: cmux spot-check (only if the app is alive)**

Run: `cmux ping && echo ALIVE || echo DOWN`

If `ALIVE`: dispatch a `general-purpose` subagent:

> Read /Users/janek/Documents/dev/planter/skills/delegate/SKILL.md and /Users/janek/Documents/dev/planter/skills/delegate-shell/SKILL.md and follow them exactly. The user explicitly chose the cmux backend. Delegate the shell command `echo hello-from-cmux` to a cmux workspace named `planter-test-cmux` (create it, cwd /tmp). Monitor to completion, report the result and the exact commands you ran.

Verify: workspace created via `cmux new-workspace --name planter-test-cmux`, sentinel detected in `cmux read-screen` output. Then clean up: close the workspace with `cmux close-workspace --workspace <ref>` (resolve the ref from `cmux list-workspaces`).

If `DOWN`: skip — note in the final report that cmux was verified CLI-only.

- [ ] **Step 2: Write `README.md`**

````markdown
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

- [ ] tmux: delegate to a NEW named session (claude runner, monitored)
- [ ] tmux: delegate to an EXISTING session (reused, not recreated)
- [ ] cmux: delegate to a new named workspace
- [ ] codex runner end-to-end
- [ ] shell runner: sentinel DONE and FAIL paths
- [ ] fire-and-forget: injection confirmed, no monitoring afterwards

## License

[MIT](LICENSE)
````

- [ ] **Step 3: Final structure check**

Run: `ls -R skills .claude-plugin && jq -e .name .claude-plugin/plugin.json`
Expected: 4 skill dirs each containing exactly `SKILL.md`; jq prints `"planter"`.

- [ ] **Step 4: Confirm no stray test sessions**

Run: `tmux ls | grep planter- || echo CLEAN`
Expected: `CLEAN`

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: add README with install, usage, and verification checklist"
```

---

## Self-Review Notes

- Spec coverage: detection/priority (Task 2), session resolution + create/name (Task 2), inspect-before-inject (Tasks 2–4), runners claude/codex/shell (Tasks 4/5/3), monitoring + fire-and-forget contract (Task 2 skill text, exercised in Tasks 3–5), packaging/marketplace/MIT (Task 1), README + manual checklist (Task 6). Local-git-only distribution per design (push is the user's step).
- Verification of "existing session reuse" is covered by Task 3 Step 5 (second run against `planter-test-b`).
- Fire-and-forget has no automated scenario (it is the absence of monitoring); it stays on the README manual checklist per spec.
