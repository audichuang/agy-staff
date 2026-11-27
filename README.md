<p align="center"><img src="assets/logo/gemini-agy.svg" width="440" alt="AGY-STAFF"></p>

<p align="center"><a href="README.md">English</a> | <a href="README.zh-CN.md">Simplified Chinese</a></p>

<p align="center"><a href="https://antigravity.google/product/antigravity-cli"><img src="assets/badges/powered-by-antigravity.svg" height="20" alt="powered by: Antigravity"></a> <img src="assets/badges/model-gemini-4-neon-hydrogen.svg" height="20" alt="model: Gemini 4 Neon / Hydrogen"></p>

<p align="center"><a href="LICENSE"><img src="assets/badges/license-mit.svg" height="20" alt="license: MIT"></a></p>

Hire Google's Antigravity CLI (`agy`) as a staffer for **Claude Code** and **OpenAI Codex**. Forked from [keli-wen/agy-staff](https://github.com/keli-wen/agy-staff); this fork ships one self-contained skill, `skills/agy/`.

Since 0.8.0 the skill no longer runs agy headless. It teaches the host agent which task type and Gemini model to pick, how to write the brief, and then starts agy as an interactive session in an [Orca](https://github.com/stablyai/orca) terminal (`orca agent launch --agent antigravity`) on the single account agy is logged into. The 0.7 companion (`agy -p`, background jobs, account rotation) is gone: Antigravity's consumer terms name wrapping agy in headless automation and automated account rotation as grounds for suspension. Media work (TTS, transcripts, music, video, PDF/video analysis) is routed to Google's [gemini-api CLI](https://github.com/google-gemini/gemini-api-cli), which uses a paid API key.

## Install

Requirements: Orca with the Antigravity agent available (`orca agent installed` lists `antigravity`), agy logged in once, and optionally `gemini-api` with an API key for media.

```bash
aghub-cli -g -a claude,codex source sync audichuang/agy-staff --skill agy --install-missing --yes
```

Upgrade with `--update` instead of `--install-missing`; start a new session to pick up the new text.

## Examples

| Use case | Example request |
|---|---|
| Survey a topic | "have agy research how auth works in this repo" |
| Second opinion on a PR | "ask agy to review PR #730" |
| Challenge a plan | "agy review: challenge the migration plan in docs/plan.md" |
| Scoped fix | "have agy implement a fix for the flaky retry test" |
| Recording to transcript | "用 agy 把這段錄音轉成逐字稿" |
| Voiceover | "幫我產生一段旁白配音" (gemini-api, no agy) |

## Layout

`skills/agy/` is the install payload: `SKILL.md`, `references/` (read on demand) and `templates/` (brief skeletons the host fills in). `README.md` / `README.zh-CN.md` are kept in sync.

## License

MIT — see [LICENSE](LICENSE).
