---
name: agy
description: Delegate work to Google's Antigravity CLI (agy, fast Gemini) - research, second-opinion review of code or plans, scoped implementation, general tasks, or a quick one-shot question (image generation goes through the imagine skill unless the user names agy) - and manage its background jobs (wait, status, result, cancel, continue). Use when the user says "agy", "ask/have agy ...", "second opinion", or when an independent model or cheap parallel worker would help.
argument-hint: '<mode> [--prompt-file <path>|--stdin|--prompt "task"] [flags]'
allowed-tools: Read, Glob, Grep, Bash(node:*), Bash(git:*), Bash(gh:*)
---

# agy

Hand a task to agy through the bundled companion, collect the result, then judge it yourself.

```bash
node "<skill-dir>/companion/agy-companion.mjs" <mode> [flags] --prompt-file <path>
```

Run it **unsandboxed**: agy needs a localhost port and its OAuth token file. In Codex, request escalated permissions. Failures: `references/troubleshooting.md`.

## Pick a mode

| Mode | Use for | Runs | Default model / profile |
| --- | --- | --- | --- |
| `staffer` | anything else (image generation only when the user names agy; name the output path) | background | flash-medium / unrestricted |
| `research` | survey or deep dive on a topic or codebase | background | flash-high / unrestricted |
| `review` | independent critique of code or a plan - compose the task per `references/code-review.md` or `references/general-review.md` | background | flash-medium / unrestricted |
| `implement` | a scoped change agy makes in the working tree | background | flash-high / unrestricted |
| `ask` | one quick question, zero tools; also the smoke test (`--prompt "reply with OK"`) | foreground | flash-low / always restricted |

Coordinating several assignments or a long task: read `references/lead.md`.

## Pick a model

The mode default (flash) fits most work. Switch with `--model` when the task calls for it; `models` shows the ids on offer and each account's quota left.

| `--model` | Best at | Cost to you | Choose it when |
| --- | --- | --- | --- |
| `flash` (default) | broad, fast agentic work: research, review, implement, quick asks | Gemini 5h + weekly windows, rotated across accounts | anything without a reason to switch; `--effort low` for lookups, `high` for review/implement |
| `argon` | frontier reasoning: deep multi-step investigation, subtle bugs, architecture, long documents (256k+), charts/images, legal/finance/science writing | its own weekly-only pool, much smaller than flash's; can be slow or time out | the task is hard or costly to get wrong, and `models` shows argon's weekly comfortably above what the task needs |
| Claude Sonnet 5.5 | fast routine coding and documents; what the fallback runs | the fallback reserve (Claude group) | never ask for it: agy-pick switches to Sonnet itself once Gemini is out; `--model` refuses Claude and GPT ids |
| Claude Opus 5.5 | ambiguous, long-running work needing sustained judgment | same Claude reserve | not used by this skill |

Measured on one agy-pick review (high effort, 2026-10-22): argon took 13 min and 72k output tokens for 3 focused, well-evidenced findings; flash took 14 min and 114k for 5 broader ones. Wall time is similar on agentic work; argon spends fewer tokens and goes deeper.

## Write the task

- agy sees only the task text, not this conversation. Include the goal, relevant context and decisions, and what an acceptable result looks like.
- Keep the user's explicit authorizations verbatim ("open a draft PR", "run the e2e tests"). The templates deny commits, pushes, and side-effectful or paid calls unless the task asks for them.
- Use `--prompt-file` or `--stdin` for anything longer than a line.

## Flags

`--model <id>` (an id from `node "<skill-dir>/companion/agy-companion.mjs" models`, which lists what agy offers now; a family such as `gemini-4-argon` or a unique part such as `argon` takes `--effort`) or `--effort low|medium|high` (flash); `--restricted` (allowlisted tools only, may come back empty - `references/setup.md`); `--timeout <dur>` (60m default, 120m max; ask 2m); `--continue` / `--conversation <id>` reuse a conversation and inherit its model and profile; `--json` on review only, when machine-readable findings are requested.

## Collect the result

Background modes print a job id and the full `cd '<repo>' && node '<companion>' wait <id> --timeout 9m` command; run that command as printed, in the same unsandboxed context, one wait per job. In Claude Code use either Bash `run_in_background: true`, or run it in the foreground with Bash timeout 600000. Never pipe `wait`/`result` through `tail`/`head` or `2>&1`: stdout is the result, relay it verbatim. Exit codes: 0 done, 2 still running (wait again), 3 error, 4 canceled, 5 resumable timeout (ask the user before continuing), 1 bad command. Do not call `observe` unless the user asks for progress. Full protocol - continue, restart, cancel, recovery: `references/jobs.md`.

## Rules

- Return agy's output verbatim, before your own assessment. The `[agy-staff]` and `agy-pick:` stderr lines are metadata for you, not the user.
- Do not fix review findings, extend agy's edits, or commit its changes unless the user asks you to.
- After `implement` (or any run that edited files), inspect `git diff` before building on it.
- Reviewing untrusted code unrestricted exposes you to prompt injection; prefer `--restricted` or an isolated checkout.
