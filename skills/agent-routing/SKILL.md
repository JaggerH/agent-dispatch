---
name: agent-routing
description: Use before starting any multi-step or bulk task, or when the user says "谁来做" / "让谁做" / "delegate this" / "who should handle this" — decides whether Claude Code does the work itself, delegates to agy-run (Gemini), or delegates to codex-run (GPT-5.5). Establishes the three-way division of labor across agents callable from Claude Code.
---

# agent-routing — who does the work

Claude Code can execute a task itself, or hand it to one of two local subagent CLIs. This skill
is the routing table. It does NOT cover dispatch syntax or protocols for the workers — those live
in `agy-run` and `codex-run` respectively. Read this first to decide *who*; read the worker's own
skill for *how*.

## The three roles

### 1. Claude Code (self) — planning, verification, consistency

Owns: architectural decisions, precise refactors, anything whose result can't be cheaply
verified, anything touching credentials, and all planning/acceptance work. Most planning and
acceptance work runs through the `comet` skill (open → design → build → verify → archive), which
produces the consistency documents (proposal, design doc, delta spec) that keep multi-session work
coherent. Default owner when a task doesn't clearly fit the other two.

### 2. agy-run — Gemini 3.5 Flash (Low), bulk mechanical work

Delegate: bulk file reading/searching/summarizing, terminology-consistency sweeps, mechanical
multi-file edits, generating boilerplate, first-pass doc audits — anything where volume is high
and per-step judgment is low.

Do NOT delegate: architectural decisions, subtle refactors, anything whose result can't be
cheaply verified, tasks touching credentials.

Mechanics (dispatch syntax, DENIED protocol, verification discipline): see `agy-run` skill.

### 3. codex-run — GPT-5.5, planning second opinion + mechanical work

Delegate: a second opinion during planning (cross-model check before committing to a design or
plan — cheaper than a full zen consult), mechanical multi-file edits, generating boilerplate,
first-pass doc audits. Overlaps with agy-run on the mechanical-work tier; pick codex-run
specifically when you want a GPT-family second opinion woven into a planning step, otherwise
either worker is fine — default to whichever is already warm/cheaper for the session.

Default model: GPT-5.5.

Mechanics: see `codex-run` skill.

## Quick table

| Task shape | Owner |
|---|---|
| Architecture / design decision | Claude Code (self), via `comet` if multi-session |
| Precise refactor, subtle logic change | Claude Code (self) |
| Result can't be cheaply verified | Claude Code (self) |
| Touches credentials | Claude Code (self) |
| Bulk read/search/summarize across many files | agy-run or codex-run |
| Terminology-consistency sweep | agy-run or codex-run |
| Mechanical multi-file edit / boilerplate | agy-run or codex-run |
| First-pass doc audit | agy-run or codex-run |
| Second opinion on a plan/design before committing | codex-run (or zen consult for higher-stakes divergence — see `~/.claude/docs/zen-consult.md`) |

## Escalation

If a delegated task comes back DENIED, fabricated-looking, or the worker is clearly out of its
depth (misjudging scope, looping, producing templated non-answers), pull the task back to Claude
Code rather than re-prompting the worker repeatedly. Delegation exists to save the expensive
model's time, not to force cheap models through work they can't do.
