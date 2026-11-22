# Installing agy-staff — a guide for coding agents

You are a coding agent asked to install and verify the **agy** skill from `audichuang/agy-staff`. When you finish (or get blocked), report the outcome to the user in the language they have been using with you.

> [!IMPORTANT]
> If you reached this file through a URL, read its **raw** text — `curl -fsSL https://raw.githubusercontent.com/audichuang/agy-staff/master/docs/INSTALL_FOR_AGENTS.md` — or clone the repo. A web tool that summarises the page can paraphrase the install command, and a paraphrased command is a broken one.

## 0. Prerequisites

1. **`agy` binary** — run `agy --version`; it must support `stream-json`. If it is missing, do **not** install it yourself: give the user <https://antigravity.google/docs/cli/install> and wait for them to install it and log in (running `agy` interactively once handles login).
2. **Node.js** — run `node --version`. The companion uses only the Node standard library.
3. **`aghub-cli`** — it installs the skill for Claude Code and Codex. If it is missing, report that and stop.

## 1. Install or upgrade

This repo ships one self-contained skill, `skills/agy/`. aghub-cli copies it to `~/.aghub/agy` and links it into each agent's skills folder:

```bash
aghub-cli -g -a claude,codex source sync audichuang/agy-staff --skill agy --install-missing --yes   # install / repair links
aghub-cli -g -a claude,codex source sync audichuang/agy-staff --skill agy --update --yes            # upgrade an existing install
```

Without `--yes` either command only previews. A session that started before the install may not list the skill yet; the shell smoke test below works regardless.

> [!IMPORTANT]
> agy cannot run inside a harness command sandbox (e.g. Codex workspace-write): it binds a localhost port and reads its OAuth token file, which the sandbox blocks or hides. Every companion command — dispatch and `wait` / `status` / `result` alike — must run **unsandboxed** (in Codex, request escalated approval or have the user grant full access). The failure signature is `operation not permitted` on `~/.gemini/antigravity-cli/...` or on binding `127.0.0.1`, followed by empty output or a bogus "authentication failed".

## 2. Smoke test

```bash
node ~/.aghub/agy/companion/agy-companion.mjs ask --prompt "reply with OK"
```

Expect a short answer on stdout and an `[agy-staff]` line on stderr (mode, model, duration, conversation id — for you, not the user). If it errors, relay the error verbatim; the usual causes are expired agy auth or an invalid model id (`node ~/.aghub/agy/companion/agy-companion.mjs models` lists valid ids). Do not improvise flags to work around errors.

A passing `ask` means the install is done. `staffer`, `research`, `review` and `implement` default to the unrestricted profile and need no setup.

## 3. Optional hardening

Only when the user wants `--restricted` runs: follow `skills/agy/references/setup.md` (dry run first; the rules go into agy's global settings; apply only after explicit confirmation). Do not run setup unprompted.

## 4. Report back

Tell the user, in their language: whether the install succeeded, the smoke-test result, and whether setup was applied, declined, or not offered.
