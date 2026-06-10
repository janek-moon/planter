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
- Working indicator: codex shows a spinner/working status; done when the composer is idle again — confirm with two identical captures, and read the content: a follow-up question from codex also yields identical captures; report questions to the user instead of declaring the task done.
- Codex may render approval prompts for commands; report them to the user, do not answer them yourself.

## Common Mistakes

- Treating the login screen as a ready composer and typing the task into it.
- Submitting multi-line prompts line-by-line (same early-submit failure as claude).
