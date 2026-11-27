---
name: agy
description: Delegate work to Google's Antigravity CLI (agy, fast Gemini) through Orca - research, second-opinion review of code or plans, scoped implementation, general tasks, or a quick one-shot question - and follow the Orca terminal to the result; also routes media work to the gemini-api CLI (voiceover/TTS, transcripts and captions, music, video, analysing a long PDF, recording or YouTube video). Use when the user says "agy", "ask/have agy ...", "請 agy", "second opinion", "產生語音/旁白/逐字稿/字幕", or when an independent model or cheap parallel worker would help. Image generation goes through the imagine skill unless the user names agy or Gemini.
---

# agy

Hand a task to agy in an Orca terminal, collect the result, then judge it yourself. Media
generation and file understanding go to `gemini-api` instead (last section).

agy is started **only** through `orca agent launch` / `orca worktree create --agent antigravity`,
on the account agy is already logged into. Never run `agy -p`, `agy-pick`, `agy-as`, `AGY_BIN`, or
`orca account select`: Antigravity's consumer terms name wrapping agy in headless automation and
automated account rotation as grounds for suspension (agy-accounts `decisions.md`, 2026-11-26). Only the two Macs can launch it (`orca agent installed` lists `antigravity`); ubuntu
cannot.

## Pick the task type

| Type | Use for | Brief template | Default `--model` / `--effort` |
| --- | --- | --- | --- |
| staffer | anything else | `templates/staffer.md` | `gemini-4-hydrogen` / `medium` |
| research | survey or deep dive on a topic or codebase | `templates/research.md` | `gemini-4-neon` / `high` |
| review | independent critique of code or a plan; compose per `references/code-review.md` or `references/general-review.md` | `templates/review.md` | `gemini-4-neon` / `high` |
| implement | a scoped change agy makes in a worktree | `templates/implement.md` | `gemini-4-neon` / `high` |
| ask | one quick question, no tools; also the login/quota probe ("reply with OK") | `templates/ask.md` | `gemini-4-hydrogen` / `low` |

Coordinating several assignments or a long task: read `references/lead.md`.

## Pick a model

Neon is for deep reasoning (research, review, implement); Hydrogen, faster and half the price, for
everyday work. Carbon is Google's strongest for long-horizon projects but costs about a third more
per task than Neon, so it is opt-in. All three draw on the same Gemini 5h + weekly windows of the
one logged-in account. `agy models` (read-only) lists the ids on offer.

Keep the type's default unless one of these applies (first match wins):
1. The user names a model or tier: use it.
2. The user asks for the strongest model, or Neon `xhigh` already fell short on this task:
   `gemini-4-carbon` / `high` (its maximum). Not for work that is merely long: one run gets cut
   around 37-60m, so split multi-hour work per `references/lead.md`, and have an implement/staffer
   task run only the tests its change touches; you run the full or UI suite afterwards.
3. Hard or costly to get wrong, in any type (staffer included): `gemini-4-neon` / `high`;
   `xhigh` for the hardest.
4. Routine (a typo, a summary, a one-fact lookup), in any type: `gemini-4-hydrogen` / `low`. A
   survey is not routine.

There is no quota query: when the user says Gemini quota is low, lower `--effort` first and drop
to Hydrogen only where a miss is cheap.

Never pass a Claude or GPT id even though `agy models` lists them: that spends Antigravity quota on
Claude - start a Claude subagent instead.

## Write the brief

- agy sees only the brief, not this conversation. Copy the type's template, replace `{{TASK}}` with
  the goal and what an acceptable result looks like, `{{CONTEXT}}` with relevant context and
  decisions, `{{WORKSPACE}}` with the worktree's absolute path. For research, review and implement
  also paste `templates/evidence.md` under the task.
- Keep the user's explicit authorizations verbatim ("open a draft PR", "run the e2e tests"). Orca
  starts agy with `--dangerously-skip-permissions`, so the template's guardrails (no commit, push,
  side-effectful or paid calls unless authorized) are the only brakes - keep them.
- Name the deliverable's absolute path in the brief (`<dir>/transcript.md`, `<dir>/review.md`) and
  end with: write the full result there, then print the line `AGY-DONE <id>` on its own. Reading
  the file beats copying from the screen; left to choose, agy has handed back a progress log with
  the real content in another file.
- Media inside an agy task (the user asked agy to transcribe a recording or judge audio): agy may
  not read the file itself, so allow `gemini-api` in the brief with the Doppler command from the
  last section and a call cap ("at most 6 gemini-api calls"); paid calls are otherwise denied.
  For a transcript, list the names and terms you know (attendees, products) so agy can spell them.
- Save the brief to a file (outside the repo, e.g. `mktemp -d`) and pass it with
  `--prompt "$(cat <brief>)"`; Orca sends multi-line text, CJK and quotes intact.

## Launch

The worktree must be one Orca knows. A new checkout plus agent in one step:

```bash
orca worktree create --repo name:<repo> --name <ticket> --base-branch origin/<main> \
  --agent antigravity --surface terminal --model gemini-4-neon --effort high \
  --prompt "$(cat <brief>)" --json
```

An existing Orca worktree (a plain `git worktree add` dir that Orca never registered returns
`selector_not_found`):

```bash
orca agent launch --agent antigravity --worktree path:<abs path> --surface terminal \
  --model gemini-4-neon --effort high --prompt "$(cat <brief>)" --json
```

Then `orca terminal read --terminal <handle>`: if the brief is still sitting in agy's input box,
submit it with `orca terminal send --terminal <handle> --enter --wait-submit 10`. Read-only types
(research, review, ask) can launch in the current worktree. Parallel editing
assignments each need their own worktree. If the launch reply is lost, rerun with
`--retry-launch <operationId>` from the error, never as a fresh launch.

## Collect the result

`result.outcome.handle` in the launch JSON is the terminal. Wait in the background:

```bash
orca terminal wait --terminal <handle> --for tui-idle --json
```

`tui-idle` also fires during agy's thinking pauses. After it returns, `orca terminal read
--terminal <handle>` and count the run as finished only when a line, trimmed, equals
`AGY-DONE <id>` (the screen echoes the brief, so a substring match lies) and the screen no longer
shows `Generating...`; otherwise wait again. Then read the result file.

- Follow-up in the same conversation: `orca terminal send --terminal <handle> --text "<msg>" --enter`. Once the terminal
  is closed, open a new one with a brief that says what is already done.
- Close the terminal when you have the result (`orca terminal close --terminal <handle>`).
- 429, quota, usage limit or a login prompt on screen means this writer failed: hand the task to a
  Claude subagent of matching strength and say so in your report. Do not switch accounts, retry
  on another account, or wait for the quota to reset.

## Rules

- Relay agy's result in the user's language, completely and faithfully, before your own
  assessment: every finding with its file:line, none dropped or softened; code, commands and error
  text as printed.
- Do not fix review findings, extend agy's edits, or commit its changes unless the user asks you to.
- After an implement run (or any run that edited files), inspect `git diff` before building on it.
- Reviewing untrusted code exposes agy (which runs unrestricted) to prompt injection: use an
  isolated checkout.

## Media: gemini-api

For producing or understanding media, call the `gemini-api` CLI directly - no Orca, no agy -
unless the user names agy ("用 agy 轉逐字稿"): then dispatch agy as above and let it use
`gemini-api` inside the brief. It is
a paid Gemini API key billed per call (separate from agy's subscription), so run it only when the
user asked for the asset, and preview an unfamiliar call with `--dry-run` first.

| Need | Command (default model) |
| --- | --- |
| voiceover, narration, TTS | `gemini-api tts` (`gemini-3.8-flash-tts`) |
| transcript or captions of audio/video | `gemini-api transcribe --format md\|text\|json\|srt` |
| questions about a PDF, recording, video, YouTube URL | `gemini-api analyze -i <file or URL>` |
| music | `gemini-api music` |
| video | `gemini-api video` |

- Images go through the imagine skill, unless the user names Gemini (then `gemini-api image`).
- The key lives in Doppler: `doppler run -p api-key -c dev -- sh -c 'GEMINI_API_KEY="$GOOGLE_API_KEY" gemini-api <...>'`.
  Never write it to a file or argv.
- Write outputs with `--out <dir>` into the place the user wants them and report the printed paths.
- Flags, schemas and the rest (`agent`, `files`, `voices`): `gemini-api <command> --help` or
  `--usage`. Not installed: `curl -fsSL https://raw.githubusercontent.com/google-gemini/gemini-api-cli/main/scripts/install.sh | bash`.
