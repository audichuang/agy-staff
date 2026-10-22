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

## Write the task

- agy sees only the task text, not this conversation. Include the goal, relevant context and decisions, and what an acceptable result looks like.
- Keep the user's explicit authorizations verbatim ("open a draft PR", "run the e2e tests"). The templates deny commits, pushes, and side-effectful or paid calls unless the task asks for them.
- Use `--prompt-file` or `--stdin` for anything longer than a line.

## Flags

`--model <id>` or `--effort low|medium|high`; `--restricted` (allowlisted tools only, may come back empty - `references/setup.md`); `--timeout <dur>` (60m default, 120m max; ask 2m); `--continue` / `--conversation <id>` reuse a conversation and inherit its model and profile; `--json` on review only, when machine-readable findings are requested.

## Collect the result

Background modes print a job id and the full `cd '<repo>' && node '<companion>' wait <id> --timeout 9m` command; run that command as printed, in the same unsandboxed context, one wait per job. In Claude Code use either Bash `run_in_background: true`, or run it in the foreground with Bash timeout 600000. Never pipe `wait`/`result` through `tail`/`head` or `2>&1`: stdout is the result, relay it verbatim. Exit codes: 0 done, 2 still running (wait again), 3 error, 4 canceled, 5 resumable timeout (ask the user before continuing), 1 bad command. Do not call `observe` unless the user asks for progress. Full protocol - continue, restart, cancel, recovery: `references/jobs.md`.

## Rules

- Return agy's output verbatim, before your own assessment. The `[agy-staff]` and `agy-pick:` stderr lines are metadata for you, not the user.
- Do not fix review findings, extend agy's edits, or commit its changes unless the user asks you to.
- After `implement` (or any run that edited files), inspect `git diff` before building on it.
- Reviewing untrusted code unrestricted exposes you to prompt injection; prefer `--restricted` or an isolated checkout.
