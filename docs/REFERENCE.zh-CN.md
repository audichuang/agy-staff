# agy-staff — 完整参考

返回 [README](../README.zh-CN.md)。英文版见 [English reference](REFERENCE.md)。

## 模式与默认值

| 模式 | 用途 | 默认模型 | 权限配置 | 执行方式 |
|---|---|---|---|---|
| `ask` | 不使用工具的低成本一次性问答（约 3 秒）；也可作为安装后的冒烟测试 | `gemini-4-hydrogen-low` | restricted（仅靠提示词） | 同步——答案在同一次调用中返回 |
| `staffer` | 通用委派，没有专门的角色或固定的输出格式；仍遵守共享的操作约定 | `gemini-4-hydrogen-medium` | unrestricted | 后台任务——返回任务 ID |
| `research` | 深度调研，引用来源，并明确标出尚未验证的结论 | `gemini-4-neon-high` | unrestricted | 后台任务——返回任务 ID |
| `review` | 第二意见的验证者，按对象分成两种方式：代码审查（按严重程度列出问题，附 `file:line` 位置）和通用审查（对方案、设计或决策做多角度的挑战） | `gemini-4-neon-high` | unrestricted | 后台任务——返回任务 ID |
| `implement` | 范围明确的编码任务；agy 修改工作区，也可以完成明确要求的 Git 交付 | `gemini-4-neon-high` | unrestricted | 后台任务——返回任务 ID |

`skills/agy/references/lead.md` 为当前 agent 提供任务编排指导，复用现有的 companion 模式，没有自己的模式。

执行方式由模式决定，不能通过参数覆盖。`continue` 沿用所解析模式的执行方式（继续一个 `ask` 仍是同步；继续其他模式会返回任务 ID）。

Claude Code 和 Codex 加载同一个技能（`skills/agy/`，安装到 `~/.aghub/agy`）：`SKILL.md` 把请求路由到某个模式，`references/` 存放按需读取的细节（审查的组装、任务、主导、setup、疑难排查），`templates/` 存放发给 agy 的提示词，一个 companion 脚本（`companion/agy-companion.mjs`，只依赖 Node 标准库）负责运行一切。在 Claude Code 中用 `/agy <mode> …` 调用，在 Codex 中用 `$agy <mode> …`，也可以直接用自然语言提问（“agy 的任务完成了吗？”）。

<a id="双权限档模型"></a>

## 双权限配置模型

每个模式都恰好运行在两种配置之一下。**所有使用工具的模式默认都是 `unrestricted`**，所以技能装好就能用，不需要允许列表，也不需要 setup；`--restricted` 是需要主动选择的加固参数。`--restricted`／`--unrestricted` 可按次覆盖（`ask` 不使用工具，强制为 restricted——它忽略这两个参数，传入 `--unrestricted` 时会打印一条说明，然后仍按 restricted 继续）。

一次运行的权限配置按以下顺序确定：命令行参数 > 已记录的会话配置（继续会话时）> 仓库策略（[`setup --restrict`](#仓库级-policysetup---restrict)）> 内置默认值。

| | **unrestricted**（默认：staffer、research、review、implement） | **restricted**（可选加固；ask 强制使用） |
|---|---|---|
| agy 调用方式 | `--dangerously-skip-permissions` | 不跳过权限——失败即关闭，未列出的工具调用一律自动拒绝 |
| agy 能做什么 | 任何事，包括修改文件和运行命令 | 读取工作区文件，加上 `git gh cat head ls grep find rg wc`，并带有五个针对性的 git/gh deny 前缀（AGY 权限规则） |
| 安全网 | 提示词层面的护栏（对不可逆或高成本的操作默认拒绝）+ 下面的分级 git 保护 | agy 仍处于它自己的权限检查之内 |
| 典型用途 | 日常路径：问答、调研、审查、编码任务 | 加固运行：不可信输入，或不能接受跳过 agy 权限提示的机器 |

`--restricted` 和 `--unrestricted` 互斥；同时传入会报错。加固之前要知道两点：`--restricted` 只有在安装了 setup 的允许列表之后才有用（否则 agy 会拒绝它自己的取证操作，运行结果为空）；而且部分原生工具在非交互模式下完全忽略允许规则——受限运行返回的内容可能比不受限的更少。

<a id="分级-git-保护"></a>

### 分级 git 保护

这些保护**仅适用于 unrestricted 运行**，各模式的处理不同：

| 模式 | 在 git 仓库内 | 不是 git 仓库 |
|---|---|---|
| `implement` | 允许工作区有未提交改动。如果仓库已有改动，companion 会在 agy 的提示词中加入一份简短的运行前状态摘要，让它知道哪些路径已经被改过，并要求把它们视为用户已有的工作。摘要有长度上限；如果无法判断归属，agy 应自行运行 `git status --porcelain` 并查看差异。任务结束后，companion 会报告工作区是干净、有改动，还是仍有未提交改动 | 提醒你无法通过 git 审查或回滚 agy 的修改，然后继续运行 |
| `research`、`review` | 从不拦截，也不检查工作区是否干净。worker 在运行前记录 `git status --porcelain`，运行后再比较；如果 agy 引入了改动，结果中会附上列出差异的警告和回滚提示 | 没有可比较的状态——静默 |
| `staffer` | 前后比较与报告方式同 research/review，但措辞中性：通用任务本来就可能修改文件，所以差异只是提供给调用方的信息（“确认任务确实要求了这些修改”），而不是指责 | 没有可比较的状态——静默 |

对 `research`／`review` 来说，静默是正常情况：只有 agy 动了工作区时才会出现差异警告，而模板明确要求它不要这么做。

<a id="prompt-层护栏默认拒绝按需开放"></a>

### 提示词层面的护栏：默认拒绝，由提示词开放

`staffer`、`research`、`review` 和 `implement` 的模板**默认**拒绝不可逆或高成本的副作用：

- 不提交、不推送、不写入 PR、不改写历史，除非任务明确要求这种具体的 Git 交付；
- 不删除工作区之外的文件；
- 不发起有副作用的网络请求；
- 不运行会消耗付费 API 额度或 token 的命令（例如会向真实 API 计费的 e2e 测试）。

临时脚本放在临时目录中，运行在工作区内做的所有修改都应保留通过 git 撤销的可能。

**默认是关上的，不是锁死的。** 如果你的请求明确授权了其中某项操作（“提交这个改动”、“创建一个草稿 PR”、“运行 e2e 测试”、“调用 staging API”），agy 只会执行被授权的操作，并报告实际运行了什么——所以委派时要把这类授权原样传过去。尤其是 `review`，它可以运行只读命令、临时脚本和测试来验证某个问题；它不能做的是修改被跟踪的文件、提交或推送。

### 审查不可信内容

在 `unrestricted` 下，提示词注入就是代码执行。如果你让 `review` 或 `research` 去处理不可信的人写的内容——陌生人的 PR、引入的第三方依赖、写满指令的 issue 正文——内容里的文字可以让 agy 运行任意命令，而不受限的 agy 会照做。

有两种缓解办法，都不是默认值：

- **`--restricted`**——agy 自己的权限检查生效，未列出的工具会被自动拒绝。限制同上：它需要 setup 的允许列表才有用，允许列表按前缀匹配而不是只读，而忽略允许规则的原生工具可能因失败即关闭而让审查内容变少。它缩小了影响范围，但不是沙箱。
- **隔离的检出目录**——在一次性的克隆、容器或虚拟机中审查，里面不放任何值得被偷的凭据。

默认值针对的是最常见的情况：你自己的代码、你自己的机器。处理不可信输入时，就应该选用这两种办法之一。

<a id="可选加固-setup"></a>

### 可选加固（setup）

`setup` 是可选的 companion 管理命令；它的引导流程在 `skills/agy/references/setup.md`。需要时，让你的宿主 agent 配置 agy 的受限模式即可。该命令会检查 `agy` 可执行文件，并预览准备写入 `~/.gemini/antigravity-cli/settings.json` 的**取证类 allow/deny 规则**。只有在明确确认之后，它才会备份文件并追加配置。默认的 unrestricted 任务和不使用工具的 ask 都不依赖 setup。

应用之前，要了解这些规则的两个特点：

- **它用一个很小的 deny 列表放行宽泛的命令。** setup 保留 `command(git)` / `command(gh)`，并为 `git push`、`git reset --hard`、`git clean`、`gh pr merge` 和 `gh release delete` 添加 deny 前缀。[AGY 按 deny > ask > allow 的顺序处理规则](https://www.antigravity.google/docs/cli/permissions/)；companion 只安装配置，没有命令解析器，也没有按任务生成的允许列表。已有的 allow/deny/ask 规则会保留。对先前收窄过的配置执行 setup，会添加其预演中展示的宽泛允许项。这些前缀用于防止常见误操作，不能覆盖所有不可逆的操作：其他参数排列、别名、脚本、API 以及其他被允许的命令都没有被全面覆盖。setup 不是只读边界。被 deny 的操作即使任务中要求也仍然被拒绝；确需执行时，请明确调整设置。
- **它是全局的。** 文件是 `~/.gemini/antigravity-cli/settings.json`，所以这些规则适用于这台机器上的每一次 `agy` 运行，而不仅仅是 agy-staff 的任务。这正是预期的产品路径（“配置一次，到处使用”）。

联网搜索不在允许列表中，也不需要：在测试过的 agy（v1.1.13）上，`search_web` 无需允许规则就能在非交互模式下运行。

<a id="仓库级-policysetup---restrict"></a>

### 按仓库设置策略（`setup --restrict`）

如果你希望*在某个特定仓库中*让某些模式每次都以 restricted 运行——比如一个你经常审查陌生人 PR 的仓库——可以一次声明下来，不必每次记得传参数：

```bash
setup --restrict review,research   # these modes default to restricted in this repo
setup --restrict none              # back to the built-in defaults
```

策略写入 `<repo>/.agy-staff/config.json`，并自动生效（运行时会打印一条说明，指出权限配置来自项目策略）。有三个特点：

- **优先级。** 显式的 `--restricted`/`--unrestricted` 参数会覆盖策略。没有显式覆盖的续跑会继承其已记录的权限配置；新任务使用仓库策略或内置默认值。`ask` 不使用工具，始终是 restricted。
- **范围。** `.agy-staff/` 通常被 git 忽略，所以这项策略是个人的、按机器的偏好——不会通过仓库分享给你的团队。
- **它不是什么。** 这是为了保持一致、防止失误的运行策略，不是安全边界：它走的是同一套 `--restricted` 机制，限制也相同（需要全局允许列表、按前缀匹配、部分工具在非交互模式下忽略允许规则）。对于真正不可信的输入，请使用隔离的检出目录。

请注意这两个文件是不同的东西：**allow/deny 规则**（受限的 agy 可以执行什么）按 agy 的设计是全局的；**策略**（哪些模式默认受限）按我们的设计是按仓库的。

<a id="进阶项目级权限"></a>

### 进阶：项目级权限

如果全局的允许列表对你来说太宽，agy 还支持与其 `--project` 体系关联的项目级权限规则（它将这类规则视为最高优先级）。这样你就可以只在你委派任务的仓库内授予取证规则。

坦率地说明限制：**项目级设置文件的确切路径没有文档说明，也没有对照当前的 agy 版本验证过**，所以 agy-staff 不会写入它，本文档也不去猜测。如果你想要项目级范围，请在 agy 交互模式中确认它从哪里读取项目级规则，再自行配置。在此之前，要么接受全局范围，要么完全跳过 setup——默认的 unrestricted 配置是绕过 agy 的权限系统，而不是依赖它，`ask` 则完全不需要允许列表；跳过 setup 的代价只是无法用 `--restricted` 加固一次运行。

<a id="统一-flags各模式一致"></a>

## 参数（各模式一致）

| 参数 | 含义 |
|---|---|
| `--conversation <id>` | 继续指定的 agy 会话 |
| `--continue` | 继续状态记录中该模式最近一次的会话 ID |
| `--model <id>` | agy 模型，会对照 `agy models` 的实时列表检查（`models` 子命令会列出该列表；当 `AGY_BIN` 是 agy-pick 时，还会列出各账号剩余的额度）。模型 ID 带有推理强度后缀（`gemini-4-hydrogen-low`）；也可传模型系列名或其中一段（`neon`、`flash`；同一系列有多个版本时取最新），并配合 `--effort`（默认 medium，否则取该系列最高的一档）。Claude 和 GPT 的 id 会被拒绝：它们是 agy-pick 的备用额度。Argon 的 id 也会被拒绝：自 0.7.9 起本技能用 Neon 取代它。未知或有歧义的名称会在启动前报错并列出可用 ID；无法获取列表时，完整的 id 会直接放行 |
| `--effort low\|medium\|high\|xhigh` | 指定该模式默认模型的推理强度：research/review/implement 为 `gemini-4-neon-<effort>`，staffer/ask 为 `gemini-4-hydrogen-<effort>` |
| `--restricted` / `--unrestricted` | 覆盖权限配置（`ask` 会忽略）。`staffer`/`research`/`review`/`implement` 默认就是 `unrestricted`，所以实际会用到的是 `--restricted` |
| `--restrict <modes\|none>` | （setup）按仓库设置策略：列出的模式在本仓库默认使用 restricted；`none` 清除策略。见[仓库策略](#仓库级-policysetup---restrict) |
| `--json` | （review）由模式强制的 JSON 格式问题列表；默认是自由格式的 markdown。面向代码审查方式 |
| `--timeout <dur>` | 后台 worker 的硬性时限（默认 60m，最长 120m）。AGY 会收到所选的响应超时参数。对同步的 ask：AGY 的响应超时，默认 2m |
| `--prompt <text>` | 将任务正文作为一个参数传入。要用引号包住；里面的内容都是不透明的 |
| `--prompt-file <path>` | 从文件读取任务正文——适合较长的提示词，免去 shell 引号的麻烦 |
| `--stdin` | 从标准输入读取任务正文。每次调用恰好选一个任务来源：`--prompt`、`--prompt-file` 或 `--stdin` |

这张表就是全部的公开接口。没有用于选择执行方式的参数——见上面的模式表。

## 任务正文

本节讲的是 **companion CLI**，而不是你如何调用技能。你把任务写成普通文本（`/agy review Review PR #730`）；技能读取它，并组装出下面的 `--prompt` 调用。你自己不需要输入 `--prompt`。

每个运行类命令（`staffer`、`research`、`review`、`implement`、`ask`、`continue`）都从以下三个来源中**恰好一个**获取任务：

```
ask       --prompt "what does git diff --check verify?"
research  --prompt-file /tmp/task.md
review    --stdin < /tmp/task.md
```

同时提供两个来源会报错（`task text given more than one way (…) — use exactly one`），一个都不提供也会报错。

**不透明性保证。** companion 按 shell 交给它的参数边界解析一次 argv：不会再次拆分任何参数，不解释引号，也不检查任务值中的任何字节。你的任务逐字节到达 agy——包括空白、引号和换行——其中形似选项的文本（`--check`、`--json`、`--timeout`、未知的 `--whatever`）是提示词内容，绝不会成为 companion 的选项。`--prompt-file` 和 `--stdin` 的内容同理。

当值是一句真正的话（即包含空白）时，`--prompt` 接受以 `--` 开头的值：`ask --prompt "--check means what?"` 可以正常执行。形似选项且不含空白的值会被视为忘了传值而被拒绝。`--` 本身没有特殊含义；它会被解析为未知选项。

需要取值的参数（`--conversation`、`--model`、`--effort`、`--timeout`、`--restrict`、`--prompt`、`--prompt-file`）必须带值。缺少值、空值，以及——除上述 `--prompt` 的句子外——形似选项的值，都会报错，所以 `--model ""` 是错误的。

每个选项都是独立的参数。把多个选项塞进一个带引号的字符串（`review "--restricted Review PR #730"`）会报错，并提示正确的写法，而不是去猜选项在哪里结束、任务从哪里开始。

<a id="review-完全基于-prompt"></a>

## 审查基于提示词

`review` 接收一段对象描述，并用它拥有的工具自行收集证据（PR 用 `gh pr view`/`gh pr diff`，引用和工作区用 `git diff`/`git log`，补丁则读取文件）。没有用来传入 diff 的参数；请在提示词中描述审查对象：

```
review --prompt "Review PR #730"
review --prompt "Review the current working tree"
review --prompt "Review changes against master"
review --prompt "Review the patch at /tmp/change.patch"
```

空的任务字符串会报错——`review` 需要一个审查对象。如果对象有歧义，agy 会被要求说明歧义，而不是猜测你的意思。

审查模板本身是中性的骨架（审查者的立场、证据纪律、护栏）。所有与具体方式相关的内容——代码审查的取证清单、审查维度、严重程度排序和输出格式，以及方案／决策审查的多角度挑战框架——都放在任务字符串里，由技能根据 `skills/agy/references/{code-review,general-review}.md` 组装。

## 状态与后台任务

**输出的分工。** stdout 承载结果以及任何关于工作区的保护警告；`[agy-staff]` 开头的运行信息行（模式、权限配置、模型、耗时、token、会话 ID）写入 stderr，后台任务则写入 `jobs/<id>.log`。这些信息是给调用方 agent 的元数据——不属于交付内容，也不会存入 `jobs/<id>.result.md`。

`staffer`、`research`、`review` 和 `implement` 会很快返回稳定的任务句柄。默认流程：准备提示词、派发、等待最终结果，再按需验收。运行期间，不要主动 observe、读日志或检查中间产物，例行汇报也不例外。只有用户明确询问时才观测进度；失败后或收到需要介入的结果后再诊断。每个独立的 worker 都会持续读取 AGY 的 `stream-json`，即使没有任何观察者。没有额外的守护进程或调度器。

- `wait [id] [--timeout <dur>]` 等待任务完成或等待到期。完成时返回已有的结果；软到期时直接返回 JSON 观测快照，并让执行继续运行。普通的工具活动不会提前结束等待。直接运行 `wait` 默认等待 100 秒；启动输出会给出明确的 9m 等待。退出码为 2 时，对同一任务再次等待，不要额外检查进度；返回的快照不要求介入。
- `observe [id]` 始终返回有长度限制的 JSON（最多 8 KiB）：运行中返回当前进展；完成后返回最终状态、结果路径／可用性以及收取说明。error/canceled/crashed 状态还会附上有长度限制的诊断与恢复元数据，绝不包含完整报告。它独立于任何 wait 读取任务状态，不会重置时限，也不会取走其他观察者的记录。
- `status [id]` 列出任务，或显示状态以及有长度限制的诊断日志尾部。`result [id]` 重新输出已保存的结果。
- `models` 输出 agy 当前提供的模型 ID（`AGY_BIN models`），当 `AGY_BIN` 是 agy-pick 时，还会根据 `agy-pick --status` 附上各账号剩余的额度（5h／每周）。
- `cancel <id>` 请求取消，在 worker 保存报告并发布 `canceled` 之后才返回成功。它会保留崩溃的诊断信息，也不会向未经核实的已保存 PID 发送信号。缺少取消通道的旧任务会明确报错。取消报错时需要检查任务和日志；这并不能证明执行已经停止。中断 wait 不会取消 worker。
- `continue --job <id> --prompt "..."` 沿用原来的模式／模型／权限配置继续已知会话，并创建一个关联的新任务。`continue --conversation <id>` 同样根据该已知会话确定配置，绝不会套用一个无关的最近模式。
- `restart <id>` 显式地用原任务／原配置重新启动，不复用会话，并关联到原任务。两种恢复操作之前，都应先用 `git status` 和 `git diff` 检查工作区中已有的部分改动。新的运行会重新生成工作区上下文。旧版本保存的任务规格会把历史快照标记出来，再追加当前上下文。

续跑／重启可以从同一 worktree 内的任意目录发起，并在原来的工作目录中执行。通用 `continue` 遇到未登记的会话 ID 会报错，不会启动 AGY，也不会搜索其他 worktree。如果目标会话对应的任务仍在运行（无论是通用 `continue`，还是某个模式的 `--continue`/`--conversation`），会以退出码 1 拒绝，并返回该任务的 ID 和状态；后续指令不会排队，调用方需要先等待或取消。续跑时，显式指定的模型／权限参数会覆盖已记录的值；否则沿用这些值。新的恢复时限默认为 60m，除非另行指定。

`wait`／`observe`／带任务 ID 的 `status` 的退出码：**0** 完成，**2** 运行中，**3** 失败／崩溃，**4** 已取消，**5** 需要关注（可续跑的超时），**1** 命令出错。`result` 对 attention 返回 5，对其他所有已结束的状态（包括 error 和 canceled）都返回 0。`observe` 退出码为 0 表示任务已结束，不表示报告已交付。请收取已经在等待的 wait 会话；否则调用 result。wait/result 保持原有的完整结果约定。`done` 状态表示调用和答复交付已结束，不代表任务已验收。有警告的成功调用会在 stderr 中附带 8 KiB 的日志尾部以及完整日志路径；原始状态和退出码记录在该日志中。编排者判断任务是否完成，仅在需要时才进一步检查。中间的工具错误会在内部保留，不会自动升级为成功交付中的警告。运行中的快照包含时间戳、已运行时长、最近五次工具活动的输入／输出节选，以及按步骤合并的最新回答文本。未知状态、不完整的文本和被截断的内容都会标明；一次工具调用完成，并不能证明取得了有用的进展。UTF-8 JSON 的长度预算是：每次活动 1 KiB、文本 2 KiB、总计 8 KiB。仅在回答用户明确提出的进度问题，或失败／需要介入后诊断，且返回的信息不够时，才读取有限的 `details` 节选。

worker 会把所选的超时参数传给 AGY，并独立地对包含初始化在内的总时长设限：默认 60m，可在启动时用 `--timeout` 调整，最长 120m。如果在硬性到期清理之前已经收到回答文本，就连同警告一起交付；回答为空或没有回答时，已知会话则为 attention，否则仍是失败。wait/observe 无法延长这个时限。没有回答时，硬性到期产生 `reason=hard_timeout`；显式的 TIMEOUT 状态，或 AGY 精确的 `ERROR` 加 `timeout waiting for response`，产生 `reason=response_timeout`。已知会话时状态为 `status=attention`、退出码 5；否则状态为 `error`。报告保留最后的快照、日志、原配置以及运行前／当前的工作区状态（porcelain 无法识别已修改文件上的进一步内容改动，需要查看 diff）。恢复元数据包括 `requires_user_confirmation`、`suggested_timeout`（原超时的两倍，后台任务最多 120m）和确切的续跑命令。已到上限时，应缩小任务。调用方 agent 必须询问用户是继续还是停下检查，得到明确确认后才恢复。显式恢复会创建一个关联的新任务并重新获得时限，保留旧任务的最终记录；companion 绝不会自动重试。

wait 在完成或软到期之前保持静默。运行期间，只有用户明确询问进度时才值得观测；保留原有的等待，不要把这次询问变成定期检查。宿主的会话收集（例如 Codex 的 `write_stdin`）取回的是待完成命令的输出，其中可能包含软到期的快照；它不会独立读取 AGY 的进度。优先使用后台完成通知或受支持的较长阻塞等待。避免没有新内容的短轮询、sleep/observe 循环，以及为例行汇报查询进度。

在运行环境支持的情况下，为每个任务使用一个独立的后台 wait，不要把多个任务串在同一个 shell 里依次等待。否则，在宿主工具调用的时限内使用尽可能长的等待。任务完成会结束正在执行的 wait，但模型何时收到结果由外层运行环境决定。Bash 加技能无法在所有环境下唤醒一个已经空闲的模型；单靠定时说明并不会安排另一次调用。

仓库内的状态保存在 `<repo>/.agy-staff/`。`state.json` 记录会话、按会话 ID 建立索引的配置历史（包括前台 ask）以及生命周期记录，更新时受短事务写锁保护；观测读取保持只读。`config.json` 保存可选的权限策略。每个任务都有执行规格、诊断日志、结果、最终状态文件、原始 stdout（`.events.jsonl`）以及原子发布的、有长度限制的快照（`.progress.json`）。原始记录可能包含未知或格式无效的事件。旧任务缺少活动文件时，只返回仅含状态的快照。

任务成功且没有警告后，会先让结果和元数据落盘，再删除原始流和快照。失败、取消、硬性超时和带警告的结果会保留中间记录。结果、诊断日志、会话元数据和 AGY 自己的会话存储都会保留。与清理竞争的读取者会重新检查最终状态：observe 返回最终元数据；wait 返回结果。observe 从不读取结果文件的内容，即使成功后的中间文件已被删除。崩溃且没有结果的报告包含派发／worker 启动的证据、进程 ID、日志是否存在及其大小，以及后续检查／恢复命令，不会复制提示词或环境变量的值。

<a id="让-agy-staff-不进-git"></a>

### 避免将 `.agy-staff/` 提交到 git

自动处理：companion 首次在仓库中创建 `.agy-staff/` 时，除非该路径已被忽略，否则会把 `.agy-staff/` 追加到 `.git/info/exclude`（仓库本地、不被跟踪）。它从不改动被跟踪的 `.gitignore`——状态目录是本地的临时空间，把它加进共享的、已提交的文件会改变其他所有人的仓库。

## 疑难排查

- **“agy reported an error (status ERROR)”**——companion 会原样转述 agy 自己的错误，仅当错误文本确实匹配某种已知情况时才附加原因提示（模型 ID 无效 → 运行 companion 的 `models` 子命令；登录过期 → 交互式运行一次 `agy` 重新登录；额度耗尽）。如果 agy 报了错但仍返回了回答文本，companion 仍会交付该回答——退出码 0，回答写入 stdout，警告写入 stderr（`done_with_warnings`）；如果到期时没有回答但会话已知，则改为 attention（退出码 5）。
- **`~/.gemini/...` 上出现 `operation not permitted` ／ `bind: operation not permitted` ／ 在终端里 `agy` 能用却突然提示 “authentication failed”**——请检查是否有宿主的命令沙箱在阻止访问。AGY 需要读取它的 OAuth 凭据，并为内部语言服务绑定本机端口；只放开工作区写权限可能不足以提供这些访问。遇到这类限制时，应使用宿主的授权机制，让 companion 在支持 AGY 的环境中运行。启动和管理任务时要使用相同的权限环境。
- **`wait`/`status` 误报崩溃（“finished with status crashed and no stored result”）**——后台任务在一种权限或沙箱环境（例如沙箱外）中启动，却在另一种环境（例如命令沙箱内）中收取。收取方无法跨沙箱边界看到 worker 的 PID，于是把仍在运行的任务误判为崩溃。请在与启动任务相同的沙箱外权限环境中运行管理命令（`wait`、`status`、`result`）；从沙箱外重新运行即可继续等待，或报告正常状态。
- **回答为空，“status SUCCESS”**——受限运行即使工具调用被拒绝，也可能报告成功。检查权限配置来自参数、仓库策略，还是更早的会话。可以用 `setup` 配置允许执行的命令，或者在已获授权时显式传入 `--unrestricted`；仅去掉 `--restricted` 不会覆盖继承来的配置。部分原生工具在受限的非交互运行中，即使有允许规则也仍然不可用。unrestricted 模式或不使用工具的 ask 返回空回答，需要另行诊断；请保留诊断信息并报告。
- **“unknown flag --X: the whole string … arrived as a single argument”**——多个选项，通常连同任务，被引号包成了一个参数。每个选项都是独立的参数；任务应放在 `--prompt` 中。见[任务正文](#任务正文)。
- **“task text exceeds the 200KB inline limit”**——companion 最终会把完整提示词作为一个命令行参数传给 AGY。`--prompt-file` 和 `--stdin` 能简化输入，却不会消除这个限制。请缩短任务正文，改为引用 PR、分支或文件，让 AGY 自行读取材料。
- **工作区附加**——前台和后台调用都会传入 `--add-dir <repoRoot>`（非 Git 目录使用启动目录），续跑和重启也一样。没有显式附加时，测试过的 AGY print 模式会话即使没有 `--sandbox`，也可能在 `~/.gemini/antigravity-cli/scratch` 中启动；仅继承 shell 的 cwd 或 trusted-workspace 设置并不会附加仓库。companion 不传入 `--sandbox`。附加工作区提供限定范围的文件读取权限；受限命令仍需要相应的允许规则。
- **implement 时工作区有未提交改动**——即使仓库已有改动，`implement` 也可以启动。companion 会在 agy 的提示词中加入有长度上限的状态摘要，让它知道这些路径是已有的用户工作。如果任务没有明确涉及它们，agy 在覆盖、清理、暂存、重置、删除、提交、推送或创建包含这些改动的 PR 之前应先询问。
- **“agy modified the working tree during this review”**——一次 unrestricted 的 `research`/`review` 运行修改了本应保留的文件。请先查看列出的路径，辨别哪些改动来自本次运行，再决定撤销什么，同时保留用户已有的工作。
- **agy 项目级权限**——agy 有与其 `--project` 体系关联的项目级规则（“最高优先级”）；这类规则的设置文件路径没有文档说明，也未经验证，所以 setup 只编辑全局文件。如果某条规则似乎被忽略，请在 agy 交互模式中确认。见[进阶：项目级权限](#进阶项目级权限)。
- **规则上下文**——agy 会自动加载工作区中的 `AGENTS.md`/`GEMINI.md`/`.agents/rules/*.md`；在你委派任务的仓库中，请让这些文件保持合理。

## 默认模型历史

0.7.10 将 Google AI Pro 自 2026-11-23 起可用的 Gemini 4 Carbon 列为可选的 `--model carbon`，用于最难的长周期任务；默认模型不变。

0.7.9 跟进 Google 发布的 Gemini 4 Neon：research、review 和 implement 默认改为 `gemini-4-neon-high`，staffer 和 ask 仍用 Hydrogen，单独给 `--effort` 时沿用该模式的默认模型。Argon 与 Orchestrator (Argon) 的 id 会被拒绝。

0.7.8 将各模式的默认模型和 `--effort` 简写从 Gemini 3.8 Flash 换成 Gemini 4 Hydrogen（推理强度档位不变），并新增 `xhigh` 档。显式 `--model` 仍可指定 Flash 的 id。

0.5.0 将各模式的默认模型，以及 `flash` 别名和 `--effort` 简写，从 Gemini 3.7 Flash 调整为 Gemini 3.8 Flash。

如果已安装的 `agy` CLI 不提供某个模式的默认模型，companion 会明确报错，不会静默回退：它会查询 `agy models`，列出可用模型并给出兼容的推荐，同时建议优先更新 `agy`。

## Windows 支持

Windows 为尽力支持，由 CI 的 `Tests (Windows)` 任务覆盖，尚未在真实的 Windows `agy` 安装上验证。子进程均以 `windowsHide: true` 启动，避免后台执行期间弹出控制台窗口。任务取消与进程清理通过 PowerShell（`Get-CimInstance Win32_Process`，`CreationDate` 使用往返精度）发现子孙进程，并逐个终止已确认身份的成员；组长进程使用 `taskkill /PID <pid> /F`，不再使用 `/T`。父子链接只有在子进程创建时间晚于父进程时才被采信：Windows 会在 `ParentProcessId` 中保留已退出父进程的 PID，该 PID 被复用后，一个无关的孤儿进程（通常是另一个任务的后台 worker）否则会被误认为子孙而被杀掉。状态锁针对 Windows 目录与标记文件的重命名和删除瞬态错误（`EPERM`/`EBUSY`/`EACCES`）进行了自动重试。

## 升级

```bash
aghub-cli -g -a claude,codex source sync audichuang/agy-staff --skill agy --update --yes
```

aghub-cli 会用新的提交替换 `~/.aghub/agy`。更早启动的会话可能仍然保留旧的技能文本；请新开一个会话来加载新版本。

## 仓库结构

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
