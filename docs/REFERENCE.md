# agy-staff — Full reference

Back to the [README](../README.md). See the [Chinese reference](REFERENCE.zh-CN.md).

## Modes and defaults

| Mode | What it is | Default model | Profile | Execution |
|---|---|---|---|---|
| `ask` | Cheap zero-tool one-shot Q&A (~3s); doubles as the post-install smoke test | `gemini-4-hydrogen-low` | restricted (prompt-only) | synchronous — the answer comes back in the same call |
| `staffer` | General-purpose delegation without a specialist role or fixed output format; shared operational guardrails still apply | `gemini-4-hydrogen-medium` | unrestricted | background job — returns a job id |
| `research` | Deep survey with cited sources and explicit unverified-claims marking | `gemini-4-neon-high` | unrestricted | background job — returns a job id |
| `review` | Second-opinion verifier, two flavors routed by subject: code review (severity-ranked findings with `file:line` refs) and general review (multi-angle challenge of a plan, design, or decision) | `gemini-4-neon-high` | unrestricted | background job — returns a job id |
| `implement` | Well-scoped coding task; agy edits the working tree and can perform explicitly requested Git delivery | `gemini-4-neon-high` | unrestricted | background job — returns a job id |

`skills/agy/references/lead.md` provides task orchestration guidance for the current agent, reusing the existing companion modes without adding a mode of its own.

Execution style is fixed per mode and cannot be overridden by a flag. `continue` inherits the resolved mode's style (continuing an `ask` stays synchronous; continuing the others returns a job id).

Claude Code and Codex load the same skill (`skills/agy/`, installed to `~/.aghub/agy`): `SKILL.md` routes the request to a mode, `references/` carries on-demand detail (review composition, jobs, lead, setup, troubleshooting), `templates/` holds the prompts sent to agy, and one companion script (`companion/agy-companion.mjs`, Node stdlib only) runs everything. Invoke it as `/agy <mode> …` in Claude Code or `$agy <mode> …` in Codex, or just ask in natural language ("is the agy job done?").

## The two-profile permission model

Every mode runs under exactly one of two profiles. **Every tool-using mode defaults to `unrestricted`**, so the skill works out of the box with no allowlist and no setup; `--restricted` is the opt-in hardening flag. `--restricted`/`--unrestricted` override per call (`ask` is tool-free and forced restricted — it ignores both, and passing `--unrestricted` to it prints a note and proceeds restricted).

The profile for a run is resolved in this order: CLI flag > recorded conversation profile (when continuing) > per-repo policy ([`setup --restrict`](#per-repo-policy-setup---restrict)) > built-in default.

| | **unrestricted** (default: staffer, research, review, implement) | **restricted** (opt-in hardening; forced for ask) |
|---|---|---|
| agy invocation | `--dangerously-skip-permissions` | no permission skipping — fail-closed, every unlisted tool call is auto-denied |
| What agy can do | anything, including editing files and running commands | workspace file reads plus `git gh cat head ls grep find rg wc`, with five targeted git/gh deny prefixes (AGY permission rules) |
| Safety net | prompt-level guardrails (default-deny on irreversible/costly actions) + the tiered git guards below | agy stays inside agy's own permission enforcement |
| Typical use | the normal path: Q&A, surveys, reviews, coding tasks | hardened runs: untrusted input, or machines where skipping agy's permission prompts is unacceptable |

`--restricted` and `--unrestricted` are mutually exclusive; passing both is an error. Two things to know before hardening: `--restricted` is only useful once setup's allowlist is installed (otherwise agy denies its own evidence gathering and the run comes back empty), and some native tools ignore allow-rules in headless mode entirely — a restricted run can be thinner than an unrestricted one.

### Tiered git guards

The guards apply to **unrestricted runs only** and differ by mode:

| Mode | Inside a git repo | Not a git repo |
|---|---|---|
| `implement` | dirty workspaces are allowed. If the repo already has changes, the companion adds a short pre-run status summary to agy's prompt so it knows which paths were already touched and must treat them as user-owned work. The summary is capped; agy should run `git status --porcelain` and inspect diffs when ownership is unclear. Afterward the companion reports whether the workspace is clean, changed, or still dirty | warns that agy's edits cannot be reviewed or rolled back via git, then proceeds |
| `research`, `review` | never blocked, no clean-tree check. The worker snapshots `git status --porcelain` before the run and compares afterwards; if agy introduced changes, the result carries a warning listing the delta plus a rollback hint | nothing to compare — silent |
| `staffer` | same snapshot/report as research/review, but neutrally worded: a general task may legitimately edit files, so the delta is information for the caller ("verify the task asked for it"), not an accusation | nothing to compare — silent |

Silence is the normal case for `research`/`review`: the delta warning shows up only when agy touched the working tree, which the templates tell it not to do.

### Prompt-level guardrails: default-deny, prompt-opens

The `staffer`, `research`, `review` and `implement` templates deny irreversible or costly side effects **by default**:

- no commits, pushes, PR writes or history rewrites unless the task explicitly asks for that exact Git delivery;
- no deleting files outside the workspace;
- no side-effectful network calls;
- no commands that burn paid API quota or tokens (e.g. an e2e suite that bills a live API).

Scratch scripts go in a temp dir, and everything the run does inside the workspace stays git-revertible.

**The default is closed, not locked.** If your request explicitly authorizes one of those operations ("commit this", "open a draft PR", "run the e2e tests", "call the staging API"), agy does exactly what was authorized and reports what it ran — so pass such authorizations through verbatim when you delegate. `review` in particular may run read-only commands, scratch scripts and tests to verify a finding; what it must not do is modify tracked files, commit or push.

### Reviewing untrusted content

Under `unrestricted`, prompt injection is code execution. If you point `review` or `research` at content written by someone you do not trust — a PR from a stranger, a vendored dependency, an issue body full of instructions — text inside that content can tell agy to run arbitrary commands, and an unrestricted agy will run them.

Two mitigations, neither of them the default:

- **`--restricted`** — agy's own permission enforcement applies, so unlisted tools are auto-denied. Caveats as above: it needs setup's allowlist to be useful, the allowlist is prefix-matched rather than read-only, and native tools that ignore allow-rules may fail-close and thin out the review. It shrinks the blast radius; it is not a sandbox.
- **An isolated checkout** — review it in a throwaway clone, container or VM with no credentials worth stealing.

The default optimizes for the common case: your own code on your own machine. Untrusted input is the case where you should reach for one of the two.

### Optional hardening (setup)

`setup` is an optional companion management command; its guided flow is `skills/agy/references/setup.md`. Ask your host agent to configure agy's restricted mode when you need it. The command checks the `agy` binary and previews **evidence-gathering allow/deny rules** for `~/.gemini/antigravity-cli/settings.json`. Only after explicit confirmation does it back up the file and append the configuration. Default unrestricted tasks and tool-free ask do not depend on setup.

Two properties of these rules you should know before applying them:

- **It allows broad commands with a small deny list.** Setup keeps `command(git)` / `command(gh)` and adds deny prefixes for `git push`, `git reset --hard`, `git clean`, `gh pr merge` and `gh release delete`. [AGY evaluates deny before ask before allow](https://www.antigravity.google/docs/cli/permissions/); the companion only installs configuration, with no command parser or per-task allowlist. Existing allow/deny/ask rules are preserved. Applying setup to an earlier narrow configuration adds the broad grants shown in its dry run. These prefixes prevent common mistakes, not all irreversible actions: other argument arrangements, aliases, scripts, APIs and other allowed commands are not comprehensively covered. The setup is not a read-only boundary. A denied operation stays denied even if requested in the task; change the settings explicitly when needed.
- **It is global.** The file is `~/.gemini/antigravity-cli/settings.json`, so the rules apply to every `agy` run on the machine, not only to agy-staff jobs. That is the intended product path ("set up once, use everywhere").

Web search is not in the allowlist and does not need to be: on the tested agy (v1.1.13) `search_web` runs headless without an allow rule.

### Per-repo policy (`setup --restrict`)

If you want some modes to run restricted every time *in a particular repository* — say, a repo where you routinely review strangers' PRs — you can declare that once instead of remembering the flag:

```bash
setup --restrict review,research   # these modes default to restricted in this repo
setup --restrict none              # back to the built-in defaults
```

The policy is written to `<repo>/.agy-staff/config.json` and applied automatically (the run prints a note that the profile came from the project policy). Three properties:

- **Precedence.** Explicit `--restricted`/`--unrestricted` flags override the policy. Continuations without an explicit override inherit their recorded profile; new tasks use the repository policy or built-in defaults. `ask` is tool-free and always restricted.
- **Scope.** `.agy-staff/` is normally git-ignored, so the policy is a personal, per-machine preference — it is not shared with your team through the repo.
- **What it is not.** This is a run policy for consistency and accident prevention, not a security boundary: it feeds the same `--restricted` machinery, with the same caveats (needs the global allowlist, prefix-matched, some tools ignore allow-rules headless). For genuinely untrusted input, use an isolated checkout.

Note the two files are different things: the **allow/deny rules** (what a restricted agy may execute) is global by agy's design; the **policy** (which modes default to restricted) is per-repo by ours.

### Advanced: project-scoped permissions

If a machine-wide allowlist is too broad for you, agy also supports project-scoped permission rules (it treats them as highest priority) tied to its `--project` system. That would let you grant the evidence rules only inside the repos where you delegate.

Caveat, stated plainly: **the exact project-settings file path is undocumented and unverified against the current agy release**, so agy-staff does not write it and this document does not guess it. If you want project scoping, check `agy` interactively for where it reads project-level rules from, and configure it yourself. Until then, either accept the global scope or skip setup entirely — the default unrestricted profile bypasses agy's permission system rather than depending on it, and `ask` needs no allowlist at all; skipping setup only costs you the ability to harden a run with `--restricted`.

## Flags (uniform across modes)

| Flag | Meaning |
|---|---|
| `--conversation <id>` | resume a specific agy conversation |
| `--continue` | reuse this mode's last conversation id from state |
| `--model <id>` | agy model, checked against the live `agy models` list (the `models` subcommand prints it, plus each account's quota left when `AGY_BIN` is agy-pick). Ids are effort-suffixed (`gemini-4-hydrogen-low`); a bare family or part of a family name (`neon`, `flash`; several versions of one family resolve to the newest) takes `--effort` (default medium, else the highest the family has). Claude and GPT ids are refused: they are agy-pick's fallback reserve. Argon ids are refused: this skill uses Neon in its place since 0.7.9. Unknown or ambiguous names fail pre-flight with the list; when the list cannot be fetched, a full id passes through |
| `--effort low\|medium\|high\|xhigh` | effort for the mode's default model: `gemini-4-neon-<effort>` for research/review/implement, `gemini-4-hydrogen-<effort>` for staffer/ask |
| `--restricted` / `--unrestricted` | permission profile override (ignored by `ask`). `unrestricted` is the default for `staffer`/`research`/`review`/`implement`, so `--restricted` is the flag you actually reach for |
| `--restrict <modes\|none>` | (setup) per-repo policy: the listed modes default to restricted in this repository; `none` clears it. See [Per-repo policy](#per-repo-policy-setup---restrict) |
| `--json` | (review) schema-enforced JSON findings; default is free-form markdown. Meant for the code-review flavor |
| `--timeout <dur>` | Background worker hard limit (default 60m, maximum 120m). AGY receives the selected response timeout. For synchronous ask: AGY response timeout, default 2m |
| `--prompt <text>` | the task text as one argument. Quote it; whatever is inside is opaque |
| `--prompt-file <path>` | read the task text from a file — for long prompts, instead of shell quoting |
| `--stdin` | read the task text from stdin. Exactly one task source per call: `--prompt`, `--prompt-file`, or `--stdin` |

That table is the whole public surface. There is no flag for execution style — see the modes table above.

## Task text

This section is about the **companion CLI**, not about how you invoke the skill. You type your task as plain text (`/agy review Review PR #730`); the skill reads it and composes the `--prompt` call below. You never type `--prompt` yourself.

Every run command (`staffer`, `research`, `review`, `implement`, `ask`, `continue`) takes its task from **exactly one** of three sources:

```
ask       --prompt "what does git diff --check verify?"
research  --prompt-file /tmp/task.md
review    --stdin < /tmp/task.md
```

Giving two at once is an error (`task text given more than one way (…) — use exactly one`), and giving none is an error too.

**The opacity guarantee.** The companion parses the shell's argv once, exactly as the shell delivered it: it re-splits no argument, interprets no quotes, and inspects no byte of a task value. Your task reaches agy byte for byte — whitespace, quotes and newlines included — and flag-like text inside it (`--check`, `--json`, `--timeout`, an unknown `--whatever`) is prompt content, never a companion option. The same holds for `--prompt-file` and `--stdin` contents.

`--prompt` accepts a value that starts with `--` when it is a real sentence, meaning one that contains whitespace: `ask --prompt "--check means what?"` works. A flag-shaped value with no whitespace reads as a forgotten value and is rejected. `--` itself carries no special meaning; it parses as an unknown flag.

Value flags (`--conversation`, `--model`, `--effort`, `--timeout`, `--restrict`, `--prompt`, `--prompt-file`) require a value. A missing one, an empty one, and — apart from the `--prompt` sentences above — a flag-shaped one are all errors, so `--model ""` is an error.

Each flag is its own argument. Flags packed into a single quoted string (`review "--restricted Review PR #730"`) are an error that names the fix rather than a guess at where the flags end and the task begins.

## Review is prompt-based

`review` takes a subject description and gathers the evidence itself with the tools it has (`gh pr view`/`gh pr diff` for PRs, `git diff`/`git log` for refs and the working tree, reading files for patches). There is no flag that hands it a diff; describe the subject in the prompt instead:

```
review --prompt "Review PR #730"
review --prompt "Review the current working tree"
review --prompt "Review changes against master"
review --prompt "Review the patch at /tmp/change.patch"
```

An empty task string is an error — `review` needs a subject. If the subject is ambiguous, agy is instructed to report the ambiguity rather than guess at what you meant.

The review template itself is a neutral skeleton (reviewer stance, evidence discipline, guardrails). Everything flavor-specific — the evidence-gathering menu, review axes, severity ranking, and output format for code reviews; the multi-angle challenge framing for plan/decision reviews — travels in the task string, composed by the skill from `skills/agy/references/{code-review,general-review}.md`.

## State and background jobs

**Output split.** stdout carries the result and any guard warning about the working tree; the `[agy-staff]` telemetry line (mode, profile, model, duration, tokens, conversation id) goes to stderr, and for background jobs into `jobs/<id>.log`. Telemetry is metadata for the calling agent — it is not part of the deliverable and is not stored in `jobs/<id>.result.md`.

`staffer`, `research`, `review` and `implement` return stable job handles promptly. Default flow: prepare the prompt, dispatch, wait for the final result, then validate as needed. While running, do not proactively observe, read logs or inspect intermediate artifacts, including for routine updates. Observe progress only when the user explicitly asks; diagnose after a failure or a result requiring intervention. Each detached worker continuously drains AGY `stream-json`, even with no observers. There is no additional daemon or scheduler.

- `wait [id] [--timeout <dur>]` waits for completion or its wait timeout. Completion delivers the existing result; soft expiry returns a JSON observation snapshot directly and leaves execution running. Ordinary tool activity does not end the wait early. Bare wait defaults to 100s; the launch output prints an explicit 9m wait. On exit 2, wait again for the same job without extra progress checks; the returned snapshot does not require intervention.
- `observe [id]` always returns bounded JSON (at most 8 KiB): current progress while running; terminal status, result path/availability and collection instructions when finished. Error/canceled/crashed states include bounded diagnostic and recovery metadata, never the full report. It reads job state independently of any wait, without resetting deadlines or consuming another observer’s history.
- `status [id]` lists jobs or shows state and a bounded diagnostic log tail. `result [id]` reprints stored output.
- `models` prints the ids agy offers now (`AGY_BIN models`) and, when `AGY_BIN` is agy-pick, each account's quota left (5h / weekly) from `agy-pick --status`.
- `cancel <id>` requests cancellation and returns success after the worker stores its report and publishes `canceled`. It preserves crash diagnostics and never signals an unverified stored PID. Legacy jobs without a cancellation channel fail explicitly. A cancellation error requires checking the job and logs; it does not establish that execution stopped. Interrupting a wait does not cancel the worker.
- `continue --job <id> --prompt "..."` resumes the known conversation with its original mode/model/profile and a linked new job. `continue --conversation <id>` also resolves configuration from that known conversation, never an unrelated last mode.
- `restart <id>` explicitly relaunches the original task/configuration without a conversation, linked to the original job. Inspect partial workspace changes with `git status` and `git diff` before either recovery action. New runs regenerate workspace context. Legacy specifications label historical snapshots and append current context.

Continuation/restart can be invoked from any directory within the same worktree and execute in the original cwd. Generic `continue` rejects unrecorded conversation IDs without launching AGY or searching other worktrees. Continuing a conversation whose job is still running (generic `continue`, or a mode's `--continue`/`--conversation`) is refused with exit 1 and the job ID and status; the follow-up is not queued, and the caller waits or cancels first. For continuation, explicit model/profile flags override the recorded values; otherwise those values are inherited. A new recovery budget defaults to 60m unless overridden.

Exit codes for wait/observe/status-with-id: **0** done, **2** running, **3** error/crashed, **4** canceled, **5** attention (resumable timeout), **1** command error. `result` exits 5 for attention and 0 for every other finished state, error and canceled included. Observe exit 0 means the job is finished, not that its report was delivered. Collect an already pending wait session; otherwise call result. Wait/result keep their existing full-result contract. A `done` status means invocation and response delivery ended, not that the task was accepted. Successful calls with warnings include an 8 KiB log tail on stderr and the full-log path; native status and exit code are recorded in that log. The orchestrator assesses completion and inspects further only when needed. Intermediate tool errors are retained internally and are not automatically promoted to warnings in successful delivery. A running snapshot contains timestamps, elapsed time, the latest five tool activities with input/output excerpts and the latest response text merged by step. Unknown states, incomplete text and truncation are labeled; a tool finishing is not proof of useful progress. UTF-8 JSON budgets are 1 KiB per activity, 2 KiB for text and 8 KiB total. For a user-requested progress answer or diagnosis after failure/required intervention, read bounded `details` excerpts only if the returned information is insufficient.

The worker passes the selected timeout to AGY and independently enforces an overall limit including initialization: default 60m, configurable up to 120m with launch `--timeout`. If response text arrived before hard-expiry cleanup, it is delivered with a warning; an empty or absent response needs attention when a conversation is known, otherwise it remains a failure. Wait/observe cannot renew it. Without a response, hard expiry produces `reason=hard_timeout`; an explicit TIMEOUT status or AGY's exact `ERROR` plus `timeout waiting for response` produces `reason=response_timeout`. A known conversation yields `status=attention` and exit 5; without one the status is `error`. Reports retain the last snapshot, logs, original configuration and pre-run/current workspace status (porcelain cannot detect content changes to an already dirty file; inspect diffs). Recovery metadata includes `requires_user_confirmation`, `suggested_timeout` (double the previous timeout, capped at 120m for background jobs) and an exact continuation command. At the ceiling, narrow the task. Calling agents must ask whether to continue or stop and inspect, and recover only after explicit user confirmation. Explicit recovery creates a linked new job with a fresh budget, preserving the old terminal record; the companion never retries automatically.

Wait is silent until completion or soft expiry. Only an explicit user progress question warrants observation while running; keep the pending wait open and do not turn the question into recurring checks. Host session collection (such as Codex `write_stdin`) retrieves the pending command’s output, which may include a soft-expiry snapshot; it does not independently read AGY progress. Prefer background completion delivery or a long supported blocking wait. Avoid short empty polls, sleep/observe loops and progress queries for routine updates.

Use one independent background wait per job where the harness supports it, never serialize multiple jobs in one shell. Otherwise use the longest practical wait within the host's tool-call limit. Completion ends an outstanding wait, but the outer harness controls when the model receives it. Bash + skills cannot universally wake an idle model; a timer instruction alone does not schedule another invocation.

Per-repository state lives in `<repo>/.agy-staff/`. `state.json` stores conversations, configuration history indexed by conversation ID (including foreground ask), and lifecycle records, protected by short write transactions; observation reads remain read-only. `config.json` holds optional permission policy. Each job has a spec, diagnostic log, result, final status sidecar, raw stdout (`.events.jsonl`) and atomically published bounded snapshot (`.progress.json`). Raw records may include unknown/malformed events. Missing activity files on legacy jobs yield a status-only snapshot.

After warning-free success, results and metadata become durable before the raw stream/snapshot are deleted. Failures, cancellation, hard timeout and warning results retain intermediates. Results, diagnostic logs, conversation metadata and AGY's own conversation storage are retained. Readers racing with cleanup recheck terminal state: observe returns terminal metadata; wait returns the result. Observe never reads result-file contents, even after successful intermediates have been deleted. Crash-without-result reports include dispatch/worker-start evidence, process IDs, log existence/size and next-inspection/recovery commands without copying prompts or environment values.

### Keeping `.agy-staff/` out of git

Automatic: when the companion creates `.agy-staff/` for the first time in a repo, it appends `.agy-staff/` to `.git/info/exclude` (repo-local, untracked) unless the path is already ignored. It never touches the tracked `.gitignore` — the state directory is local scratch, and adding it to a shared, committed file would change the repo for everyone else.

## Troubleshooting

- **"agy reported an error (status ERROR)"** — the companion relays agy's own error verbatim, and appends a cause hint only when the error text actually matches one (invalid model id → run the companion's `models` subcommand; expired auth → run `agy` interactively once to re-login; exhausted quota). If agy reported an error but still returned response text, the companion delivers the response anyway — exit 0, response on stdout, warning on stderr (`done_with_warnings`); a deadline with no response but a known conversation instead signals attention (exit 5).
- **`operation not permitted` on `~/.gemini/...` / `bind: operation not permitted` / sudden "authentication failed" while `agy` works in your terminal** — check whether a harness command sandbox is blocking access. AGY needs its OAuth credentials and a localhost port for its internal language server; allowing workspace writes alone may not provide that access. Where those restrictions apply, use the host's authorization mechanism to run the companion in a context that supports AGY. Use the same permission context to start and manage a job.
- **False crash report on `wait`/`status` ("finished with status crashed and no stored result")** — the background job was started in one permission or sandbox context (e.g. unsandboxed) and collected from another (e.g. inside a command sandbox). The collector cannot see the worker PID across the sandbox boundary and misclassifies the running job as crashed. Run management commands (`wait`, `status`, `result`) in the same unsandboxed permission context as the job start; rerunning from the unsandboxed context resumes waiting or reporting normal status.
- **Empty response, "status SUCCESS"** — a restricted run may report success even when its tool calls were denied. Check whether the profile came from a flag, repository policy or an earlier conversation. Use `setup` to configure allowed commands, or pass `--unrestricted` explicitly if authorized; dropping `--restricted` alone does not override an inherited profile. Some native tools remain unavailable in restricted headless runs even with allow-rules. An empty response from unrestricted mode or tool-free ask needs a separate diagnosis; retain the diagnostics and report it.
- **"unknown flag --X: the whole string … arrived as a single argument"** — several flags, and usually the task, are quoted into one argument. Each flag is its own argument; the task belongs in `--prompt`. See [Task text](#task-text).
- **"task text exceeds the 200KB inline limit"** — the companion ultimately passes the complete prompt to AGY as one command-line argument. `--prompt-file` and `--stdin` simplify input but do not remove that limit. Shorten the task text by referring to a PR, branch or file, and let AGY read the material itself.
- **Workspace attachment** — foreground and background calls pass `--add-dir <repoRoot>` (the launch directory outside Git), including continuation and restart. Without explicit attachment, tested AGY print-mode sessions can start in `~/.gemini/antigravity-cli/scratch` even without `--sandbox`; inherited shell cwd or trusted-workspace settings alone do not attach the repo. The companion does not pass `--sandbox`. Workspace attachment grants scoped file reads; restricted commands still need their allow-rules.
- **Dirty workspace on implement** — `implement` can start even when the repo already has changes. The companion adds a capped status summary to agy's prompt so it knows those paths are pre-existing user work. If the task does not clearly include them, agy should ask before overwriting, cleaning, stashing, resetting, deleting, committing, pushing, or opening a PR with those changes.
- **"agy modified the working tree during this review"** — an unrestricted `research`/`review` run changed files it was asked to preserve. Inspect the listed paths to identify changes from this run before deciding what to revert, preserving pre-existing user work.
- **Project-scoped agy permissions** — agy has project-level rules ("highest priority") tied to its `--project` system; the settings-file path for those is undocumented and unverified, so setup only edits the global file. If a rule seems ignored, check agy interactively. See [Advanced: project-scoped permissions](#advanced-project-scoped-permissions).
- **Rules context** — agy auto-loads `AGENTS.md`/`GEMINI.md`/`.agents/rules/*.md` from the workspace; keep those files sane in repos where you delegate.

## Default model history

0.7.10 lists Gemini 4 Carbon, open to Google AI Pro since 2026-11-23, as an opt-in `--model carbon` for the hardest long-horizon tasks; defaults are unchanged.

0.7.9 follows Google's Gemini 4 Neon launch: research, review and implement default to `gemini-4-neon-high`, staffer and ask stay on Hydrogen, and a bare `--effort` keeps the mode's default model. Argon and Orchestrator (Argon) ids are refused.

0.7.8 moves every mode default and the bare `--effort` shorthand from Gemini 3.8 Flash to Gemini 4 Hydrogen (same effort tiers) and adds the `xhigh` effort. Flash ids still work with an explicit `--model`.

0.5.0 moved every mode default and the `flash` alias/`--effort` shorthand from Gemini 3.7 Flash to Gemini 3.8 Flash.

If the installed `agy` CLI does not offer a mode's default model, the companion fails clearly without silent fallback: it queries `agy models` and reports the available models with a compatible recommendation, advising that updating `agy` is preferred.

## Windows support

Windows is supported on a best-effort basis and exercised by the `Tests (Windows)` CI job; it has not yet been validated against a real Windows `agy` installation. Subprocesses are spawned with `windowsHide: true` so no console windows appear during background execution. Job cancellation and process cleanup discover descendant processes via PowerShell (`Get-CimInstance Win32_Process`, with `CreationDate` in round-trip precision) and terminate each identified member individually; the leader falls to `taskkill /PID <pid> /F`, never `/T`. A parent link is only followed when the child was created after its parent: Windows keeps a dead parent's PID in `ParentProcessId`, so once that PID is reused an unrelated orphan (typically another job's detached worker) would otherwise look like a descendant and be killed. State locking retries transient Windows errors (`EPERM`/`EBUSY`/`EACCES`) when renaming or unlinking lock directories and marker files.

## Upgrading

```bash
aghub-cli -g -a claude,codex source sync audichuang/agy-staff --skill agy --update --yes
```

aghub-cli replaces `~/.aghub/agy` with the new commit. A session that started earlier may still hold the old skill text; start a new session to pick it up.

## Repository layout

```
skills/agy/SKILL.md                       entry point: mode routing, flags, result rules
skills/agy/references/                    on-demand detail (code/general review, jobs, lead, setup, troubleshooting)
skills/agy/templates/                     prompts sent to agy (staffer/ask/research/review/implement, evidence)
skills/agy/companion/agy-companion.mjs    command entrypoint, modes, job management and setup
skills/agy/companion/stream-worker.mjs    streaming execution, process cleanup and deadlines
skills/agy/companion/observation.mjs      event parsing, progress snapshots and output budgets
skills/agy/companion/state-lock.mjs       state-write locking and stale-lock recovery
scripts/                                  diagram, badge and logo generators
assets/                                   design diagrams, logo, badges
tests/                                    offline regression tests and opt-in integration suites
docs/                                     reference, agent install guide and release notes
```
