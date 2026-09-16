---
name: tenant-codex
description: Use when the user asks codex (OpenAI Codex CLI) to do, fix, build, or run a task — "ask codex to fix this", "hand this to codex", "codex에게 요청해줘/시켜줘/맡겨줘", "codex로 돌려줘" — with or without naming a tmux/cmux/herdr session, window, pane, or surface. Not for "codex review"/"codex challenge"/"consult codex" second-opinion requests without execution
---

# Running codex in a Delegated Session

## Overview

Same flow as planter:tenant-claude with the `codex` CLI: launch or reuse, wait for the composer, inject, watch for idle.

**REQUIRED BACKGROUND:** planter:tenant (target resolution, monitoring contract); planter:tenant-claude (injection mechanics — identical, including bracketed paste for multi-line prompts).

## Codex specifics

- Launch with `codex` (append user-requested flags verbatim).
- Ready signal: the composer input box renders.
- An update notice or trust/directory confirmation may render before the composer — these are NOT the composer. Do not type the task into them and do not answer them without user authorization; report what they say (same dialog rule as planter:tenant-claude).
- If a login/auth screen appears instead, STOP and report — never start an auth flow on the user's behalf.
- Working indicator: codex shows a spinner/working status; done when the composer is idle again — confirm with two identical captures, and read the captured content: a follow-up question from codex also yields identical captures; report questions to the user instead of declaring the task done.
- Codex may render approval prompts for commands, or usage-limit/quota notices, after submission; report them to the user, do not answer them yourself.
- If the user requested auto-approval flags (e.g. `--full-auto`), approval prompts will not appear — say so in your report, since that safety checkpoint is absent.

## herdr

Follow planter:tenant-claude's herdr section with `--kind codex` (reuse when `.agent` = `codex`). Codex notices above still apply: an update notice, trust confirmation, or login screen that makes `agent start` time out, and approval or usage-limit prompts that settle as `blocked` — read and report them, never answer them.

## Common Mistakes

- Treating the login screen as a ready composer and typing the task into it.
- Submitting multi-line prompts line-by-line (same early-submit failure as claude).
