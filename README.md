# agent-dispatch

Headless dispatch tools for running cheap/alternate models as subagents from any orchestrator
(Claude Code, Codex, CI, plain shell), plus the routing policy for deciding which one to use.

```bash
agy-run   -C ~/projects/myrepo "Find every file that mentions FeedUnit and list them."
codex-run -C ~/projects/myrepo "Second opinion: does this migration plan handle concurrent writes?"
```

Three components, each a few dozen lines of bash / one skill file. No server, no MCP bridge, no
extra auth — they drive CLIs you already have, under your existing subscriptions.

| Component | Worker | Use for |
|---|---|---|
| [`agy-run`](agy-run) | Gemini 3.5 Flash (Low), via Antigravity CLI (`agy`) | bulk read/search/summarize, terminology sweeps, mechanical multi-file edits, boilerplate, first-pass doc audits |
| [`codex-run`](codex-run) | GPT-5.5, via OpenAI Codex CLI (`codex exec`) | planning second opinions, mechanical multi-file edits, boilerplate, first-pass doc audits |
| [`skills/agent-routing`](skills/agent-routing/SKILL.md) | — (policy only) | decides whether a task stays with the orchestrator itself or goes to one of the two workers above |

## Why this exists

Both underlying CLIs have headless modes, but neither is orchestration-ready out of the box.

**`agy --print`** (all defects reproduced on agy 1.0.14):

| # | Defect | Fix in agy-run |
|---|--------|----------------|
| 1 | [issue #76](https://github.com/google-antigravity/antigravity-cli/issues/76): **silently drops stdout** when stdout is not a TTY (pipes, subprocesses — i.e. every orchestrator) | wraps the call in `script -qec` (pseudo-TTY), strips the CRLF it introduces |
| 2 | runs in agy's **own scratch workspace**, not your cwd — commands execute against the wrong directory/repo | binds the target dir with `--add-dir` (`-C` flag, defaults to cwd) |
| 3 | weak models **fabricate command output** instead of running commands | every prompt gets a discipline preamble: paste real output verbatim, never guess |
| 4 | usage limit on the cheap model kills the run | detects quota/rate-limit errors, retries once on a fallback model, loudly labeled |

**`codex exec`** is natively headless (proper non-TTY stdout, `-C` for workdir, `--sandbox` for
write scoping) — `codex-run` only adds the discipline preamble, a hard timeout, and defaults
(model `gpt-5.5`, sandbox `workspace-write`) so callers don't repeat them every time.

## Permission model

**agy**: print mode **auto-approves every tool call**, including `rm` (verified empirically).
`agy-run` assumes a deny-list in `~/.gemini/antigravity-cli/settings.json`:

```json
{
  "permissions": {
    "deny": [
      "command(rm)",
      "command(git push)"
    ]
  }
}
```

A denied command surfaces in the reply as `DENIED: Permission denied for command(rm /tmp/x). …` —
the preamble instructs the worker to report this verbatim instead of working around it. Your
orchestrator then escalates: it re-runs the command itself under **its own** user-facing approval
flow. Net effect: everything is auto-approved except the destructive pair, which becomes "ask the
human upstream" without any interactive channel into the subprocess. Adjust the deny list to
taste (`command(sudo)`, `command(docker rm)`, …).

**codex**: `codex exec` runs with `approval: never` (it's non-interactive by design) and instead
scopes writes via `--sandbox`. `codex-run` defaults to `workspace-write` (can edit files under
`-C`, nothing outside it); pass `-s read-only` for analysis/second-opinion tasks that must not
touch disk.

## Install

```bash
cp agy-run codex-run ~/.local/bin/ && chmod +x ~/.local/bin/agy-run ~/.local/bin/codex-run
```

Requirements:
- `agy-run`: `agy` (authenticated), `script` (util-linux; BSD `script` on macOS has different
  flags — see Caveats), `timeout` (coreutils).
- `codex-run`: `codex` (authenticated via `codex login`, ChatGPT/Codex subscription), `timeout`.

## Usage

```bash
agy-run   [-C workdir] [-m model] [-f fallback_model] [-t timeout_seconds] "<prompt>"
codex-run [-C workdir] [-m model] [-s sandbox] [-t timeout_seconds] "<prompt>"
```

| Flag | agy-run default | codex-run default | Meaning |
|------|------------------|--------------------|---------|
| `-C` | `$PWD` | `$PWD` | Directory the worker operates in |
| `-m` | `Gemini 3.5 Flash (Low)` | `gpt-5.5` | Primary model |
| `-f` | `Claude Sonnet 4.6 (Thinking)` | — | Fallback on usage limit (agy-run only) |
| `-s` | — | `workspace-write` | Sandbox: `workspace-write` \| `read-only` \| `danger-full-access` (codex-run only) |
| `-t` | `300` | `300` | Hard timeout in seconds (a ceiling — returns as soon as the worker finishes) |

`agy-run` streams output through in real time (`tee`), so orchestrators polling a background
process see progress. `codex-run` prints Codex's own event stream, which already streams.

### From Claude Code

Run either with the Bash tool; for long tasks use a background run and get notified on
completion. Installable skills ship in [`skills/`](skills/) — copy each `skills/<name>/` to
`~/.claude/skills/<name>/` and Claude picks the pattern up automatically. Suggested permission
allowlist entries so dispatch never prompts: `Bash(agy-run:*)`, `Bash(codex-run:*)`.

Read [`skills/agent-routing/SKILL.md`](skills/agent-routing/SKILL.md) first — it's the policy
skill that decides whether a task stays with Claude Code, goes to `agy-run`, or goes to
`codex-run`. The other two skills only cover dispatch mechanics once that call is made.

### From Codex / anything else

Both are plain commands that print the answer on stdout and exit non-zero on failure. Call them
from AGENTS.md-driven workflows, Makefiles, CI — anywhere.

## Caveats

- **agy-run's fallback trigger is text-matched** (`quota|rate limit|429|…`) against the worker's
  output; a task whose *content* legitimately contains those words could false-positive the retry.
- macOS: BSD `script` uses `script -q /dev/null command...` syntax; adapt `run_once` in `agy-run`.
- The discipline preamble reduces but cannot eliminate fabricated output from either worker — for
  load-bearing results, have the orchestrator spot-check (e.g. ask for `git rev-parse HEAD` and
  compare).
- agy-run's deny rules match the command head (`command(rm)` blocks any `rm …` invocation). They
  are enforced by agy, not by this script — keep them in agy's settings, not here.
- codex-run's sandbox is enforced by Codex itself; `danger-full-access` bypasses it entirely —
  avoid unless the task needs network or paths outside the workdir.

## License

MIT
