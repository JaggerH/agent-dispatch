---
name: codex-run
description: Use once agent-routing (or the user, saying "让 codex 做" / "用 codex" / "dispatch to codex" / "second opinion from GPT") has decided a task goes to the Codex worker. Covers dispatch syntax and output verification for the local `codex-run` command. Does NOT cover whether to delegate — see agent-routing for that.
---

# codex-run — dispatch work to a GPT-5.5 subagent via Codex CLI

`codex-run` runs OpenAI's Codex CLI headlessly (`codex exec`) against GPT-5.5, authenticated via
the ChatGPT/Codex subscription (`codex login`, already done). It is a local command, already on
PATH. Codex's `exec` subcommand is natively headless — no pseudo-TTY hack needed — so the wrapper
only adds a discipline preamble, a hard timeout, and sane defaults (model `gpt-5.5`, sandbox
`workspace-write`).

For whether a given task belongs here vs. Claude Code itself vs. agy-run, see the
`agent-routing` skill — this skill only covers mechanics once that call is made.

## Dispatch

```bash
codex-run [-C workdir] [-m model] [-s sandbox] [-t seconds] "<task prompt>"
```

- Always pass `-C <absolute project dir>` — defaults to `$PWD`, which may not be the target repo.
- `-s` sandbox: `workspace-write` (default, can edit files under `-C`), `read-only` (analysis /
  second-opinion tasks that must not touch disk), `danger-full-access` (avoid unless the task
  needs network or paths outside the workdir).
- Default timeout 300s is a ceiling, not a wait; short tasks return in tens of seconds.
- For tasks likely >1 min, use a background Bash run and continue other work; read the output
  when notified.
- Write task prompts like orders to a junior: exact files/globs, exact deliverable format,
  "paste real command output verbatim".
- For a planning second opinion, paste the actual proposal/design text into the prompt and ask
  for a critique or alternative — don't ask Codex to re-derive context it doesn't have.

## Verify before trusting

Any model can rationalize a plausible-sounding but wrong answer. For any load-bearing result:
- Have the worker include verifiable anchors (file paths + line numbers, `git rev-parse HEAD`).
- Spot-check 1-2 claims with your own Read/Grep before acting on the result.
- Treat a second opinion as input to your own judgment, not a verdict to defer to — if Codex and
  your own plan disagree, resolve the disagreement explicitly rather than picking whichever came
  last.

## Failure modes

- `codex-run: missing prompt` / usage error: check quoting — the prompt must be the final
  positional argument.
- Empty output / auth error: Codex session expired — tell the user to run `codex login` again
  interactively (opens a browser auth flow; WSL2 forwards it to the Windows browser).
- Exit 124: hard timeout; re-dispatch with a bigger `-t` or split the task.
- Startup lines like `failed to load skill ... missing YAML frontmatter` are Codex trying (and
  failing) to load unrelated Claude-format skills from `~/.agents/skills/` — harmless, ignore.
