<p align="center"><img src="assets/logo/gemini-agy.svg" width="440" alt="AGY-STAFF"></p>

<p align="center"><a href="README.md">English</a> | <a href="README.zh-CN.md">简体中文</a></p>

<p align="center"><a href="https://antigravity.google/product/antigravity-cli"><img src="assets/badges/powered-by-antigravity.svg" height="20" alt="powered by: Antigravity"></a> <img src="assets/badges/model-gemini-4-neon-hydrogen.svg" height="20" alt="model: Gemini 4 Neon / Hydrogen"></p>

<p align="center"><a href="LICENSE"><img src="assets/badges/license-mit.svg" height="20" alt="license: MIT"></a></p>

把 Google 的 Antigravity CLI（`agy`）雇来当 **Claude Code** 和 **OpenAI Codex** 的「员工」。这个 fork 自 [keli-wen/agy-staff](https://github.com/keli-wen/agy-staff)，只提供一个自包含的技能 `skills/agy/`。

从 0.8.0 起，技能不再以无头方式运行 agy。它教主 agent 选任务类型与 Gemini 模型、写 brief，然后在 [Orca](https://github.com/stablyai/orca) 终端里以交互方式启动 agy（`orca agent launch --agent antigravity`），只用 agy 当前登录的那一个账号。0.7 的 companion（`agy -p`、后台任务、账号轮替）已移除：Antigravity 个人方案条款把「把 agy 包进无头自动化」和「自动轮替账号」列为可停权的理由。媒体工作（TTS、逐字稿、音乐、视频、PDF／视频分析）交给 Google 的 [gemini-api CLI](https://github.com/google-gemini/gemini-api-cli)，使用付费 API key。

## 安装

需求：Orca 能启动 Antigravity（`orca agent installed` 列出 `antigravity`）、agy 已登录过一次；要做媒体工作再装 `gemini-api` 并准备 API key。

```bash
aghub-cli -g -a claude,codex source sync audichuang/agy-staff --skill agy --install-missing --yes
```

升级把 `--install-missing` 换成 `--update`；开新 session 才会读到新版内容。

## 示例

| 用途 | 请求示例 |
|---|---|
| 调研主题 | 「请 agy 调研这个 repo 的 auth 怎么运作」 |
| PR 第二意见 | 「请 agy 审查 PR #730」 |
| 挑战方案 | 「agy review：挑战 docs/plan.md 的迁移方案」 |
| 小范围修复 | 「请 agy 修好那个不稳定的 retry 测试」 |
| 录音转逐字稿 | 「用 agy 把这段录音转成逐字稿」 |
| 旁白配音 | 「帮我产生一段旁白配音」（gemini-api，不经 agy） |

## 结构

`skills/agy/` 是安装内容：`SKILL.md`、`references/`（按需读取）与 `templates/`（主 agent 填写的 brief 骨架）。`README.md` 与 `README.zh-CN.md` 保持同步。

## 许可

MIT — 见 [LICENSE](LICENSE)。
