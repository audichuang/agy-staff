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
| `staffer` | anything else (image generation only when the user names agy; name the output path) | background | hydrogen-medium / unrestricted |
| `research` | survey or deep dive on a topic or codebase | background | neon-high / unrestricted |
| `review` | independent critique of code or a plan - compose the task per `references/code-review.md` or `references/general-review.md` | background | neon-high / unrestricted |
| `implement` | a scoped change agy makes in the working tree | background | neon-high / unrestricted |
| `ask` | one quick question, zero tools; also the smoke test (`--prompt "reply with OK"`) | foreground | hydrogen-low / always restricted |

Coordinating several assignments or a long task: read `references/lead.md`.

## Pick a model

Modes follow Google's split: Neon for deep reasoning (research, review, implement), Hydrogen, faster and half the price, for everyday work (staffer, ask). Carbon, Google's model for the hardest long-horizon projects, is open to Pro since 2026-11-23 but Google puts Neon's cost per task about a third below Carbon's, so Carbon is opt-in. All three draw on the same Gemini 5h + weekly windows, rotated across accounts, so the bigger models spend them faster. Switch with `--model` when the task calls for it; `models` shows the ids on offer and each account's quota left. Argon is not offered here (Neon replaced it 2026-11-19; agy drops it 2026-11-30), and `--model` refuses Argon ids.

Keep the mode's default model unless one of these applies (first match wins; a `--model` without `--effort` runs `medium`):
1. The user names a model or tier: use it.
2. The user asks for the strongest model, or Neon `xhigh` already fell short on this task: `--model carbon --effort high`. Not for work that is merely long: one run stops at 120m (implement runs were cut at 37-60m), so split multi-hour work per `references/lead.md`, and have an implement/staffer task run only the tests its change touches (unit tests or one UI test); you run the full or UI suite afterwards.
3. Hard or costly to get wrong, in any mode (staffer included): Neon at `high` (research/review/implement already are; staffer: `--model neon --effort high`); `--effort xhigh` for the hardest.
4. Routine (a typo, a summary, a one-fact lookup), in any mode: `--model hydrogen`. A survey is not routine.

Then, if Gemini quota is low (`models`), lower `--effort` first; drop to Hydrogen only where a miss is cheap.

| `--model` | Best at | Choose it when |
| --- | --- | --- |
| `carbon` | Google's strongest Gemini 4: long-horizon, many-step projects | rule 2 above; efforts stop at `high` |
| `neon` (research/review/implement default) | deep reasoning: multi-file debugging and refactors, architecture, subtle bugs, security review, long documents, charts, science and finance writing | the task is hard or costly to get wrong; `--effort xhigh` for the hardest, `medium`/`low` to save quota on a light review |
| `hydrogen` (staffer/ask default) | fast everyday agentic work: lookups, routine edits, summaries, quick asks | the task is routine and speed matters, or a miss is cheap and Gemini quota is low; also a lighter `research`/`review` (`--model hydrogen --effort high`) |
| `gemini-pro-orchestrator-neon` | one message planned and checked by Neon, carried out by parallel Hydrogen subagents; fixed setting, `--effort` ignored (`gemini-pro-orchestrator` is the Carbon-planned variant, untested here) | a large implement task that splits into parallel parts, as an experiment: only smoke-tested here (it draws on the Gemini pool, measured 2026-11-19) |
| Claude Sonnet 5.5 | what the fallback runs (Claude group reserve) | never ask for it: agy-pick switches to Sonnet itself once Gemini is out; `--model` refuses Claude and GPT ids |

## Write the task

- agy sees only the task text, not this conversation. Include the goal, relevant context and decisions, and what an acceptable result looks like.
- Keep the user's explicit authorizations verbatim ("open a draft PR", "run the e2e tests"). The templates deny commits, pushes, and side-effectful or paid calls unless the task asks for them.
- Use `--prompt-file` or `--stdin` for anything longer than a line.

## Flags

`--model <id>` (an id from `node "<skill-dir>/companion/agy-companion.mjs" models`, which lists what agy offers now; a family such as `gemini-4-neon` or a unique part such as `neon` takes `--effort`) or `--effort low|medium|high|xhigh` (keeps the mode's default model); `--restricted` (allowlisted tools only, may come back empty - `references/setup.md`); `--timeout <dur>` (60m default, 120m max; ask 2m); `--continue` / `--conversation <id>` reuse a conversation and inherit its model and profile; `--json` on review only, when machine-readable findings are requested.

## Collect the result

Background modes print a job id and the full `cd '<repo>' && node '<companion>' wait <id> --timeout 9m` command; run that command as printed, in the same unsandboxed context, one wait per job. In Claude Code use either Bash `run_in_background: true`, or run it in the foreground with Bash timeout 600000. Never pipe `wait`/`result` through `tail`/`head` or `2>&1`: stdout is the result, read all of it. Exit codes: 0 done, 2 still running (wait again), 3 error, 4 canceled, 5 resumable timeout (ask the user before continuing), 1 bad command. Do not call `observe` unless the user asks for progress. Full protocol - continue, restart, cancel, recovery: `references/jobs.md`.

## Rules

- Relay agy's output in the user's language, completely and faithfully, before your own assessment: every finding with its file:line, none dropped or softened; keep code, commands and error text as printed. The `[agy-staff]` and `agy-pick:` stderr lines are metadata for you, not the user.
- Do not fix review findings, extend agy's edits, or commit its changes unless the user asks you to.
- After `implement` (or any run that edited files), inspect `git diff` before building on it.
- Reviewing untrusted code unrestricted exposes you to prompt injection; prefer `--restricted` or an isolated checkout.
