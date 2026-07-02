---
name: agy-run
description: Use once agent-routing (or the user, saying "让 agy 做" / "用 agy" / "dispatch to gemini") has decided a task goes to the Gemini worker. Covers dispatch syntax, background runs, the DENIED escalation protocol, and result verification for the local `agy-run` command. Does NOT cover whether to delegate — see agent-routing for that.
---

# agy-run — dispatch work to a cheap Gemini subagent

`agy-run` runs Antigravity CLI (`agy`) headlessly with a Gemini 3.5 Flash (Low) worker. It is a
local command, already on PATH. It fixes agy's non-TTY stdout bug, binds the working directory,
enforces output discipline, and auto-falls-back to Claude Sonnet 4.6 on usage limits.

For whether a given task belongs here vs. Claude Code itself vs. codex-run, see the
`agent-routing` skill — this skill only covers mechanics once that call is made.

## Dispatch

```bash
agy-run [-C workdir] [-m model] [-f fallback] [-t seconds] "<task prompt>"
```

- Always pass `-C <absolute project dir>` — the worker does NOT inherit your cwd reliably.
- Default timeout 300s is a ceiling, not a wait; short tasks return in tens of seconds.
- For tasks likely >1 min, use a background Bash run and continue other work; read the output
  when notified. Output streams incrementally, so polling shows progress.
- Write task prompts like orders to a junior: exact files/globs, exact deliverable format,
  "paste real command output verbatim".

## Permission / DENIED protocol

The agy side denies `rm` and `git push` (deny rules in `~/.gemini/antigravity-cli/settings.json`);
everything else is auto-approved. If the reply contains a line starting with `DENIED:`, the
worker hit a deny rule. Do not tell it to work around the denial. Instead: decide whether the
command is genuinely needed; if yes, run it yourself through your own Bash tool so the human
approves it through the normal permission prompt.

## Verify before trusting

Small models fabricate. For any load-bearing result:
- Have the worker include verifiable anchors (file paths + line numbers, `git rev-parse HEAD`).
- Spot-check 1-2 claims with your own Read/Grep before acting on the result.
- If output smells templated or too smooth, re-run with `-m "Gemini 3.5 Flash (High)"` or do it
  yourself.

## Failure modes

- Empty output / `authentication failed`: agy auth expired — tell the user to run `agy` once
  interactively to re-login.
- `[agy-run: answered by fallback model …]` header: the cheap model hit its usage limit; the
  answer came from Claude Sonnet 4.6 — costlier, mention it to the user.
- Exit 124: hard timeout; re-dispatch with a bigger `-t` or split the task.
