<a id="006-deployment"></a>

# 006 部署

> 原文版权：Copyright (c) 2026 DeepSeek Harness Team；MIT License。
> 来源：[006-deployment.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/006-deployment.md) · 固定版本：`6337fc823ce20251c8de8085948165c81aaa701e`。中文翻译，非官方；代码与配置保持原样。

本指南介绍如何安装、配置和运维 dsh-superagent，涵盖账号、启动检查、升级和故障排查。阅读前应先了解 [001 领域模型](001-domain.md)至 [005 面板](005-panel.md)中的内容。

<a id="what-runs"></a>

## 需要运行什么

- **一个专用的 DeepSeek Harness 配置档案（profile）**，本指南将其命名为 `superagent`。从 Web 配置模板创建它。它负责加载插件包，并在 `127.0.0.1` 上提供面板。请将它与你日常使用的配置档案分开：插件会修改进程级的 `PATH` 和 git 设置，而且其进程需要独立重启和升级。
- **一个智能体专用的 GitHub 账号。** 智能体通过只登录这个账号的 GitHub CLI，以该账号的身份发表评论、推送分支和创建拉取请求。
- **可选：** 一个用于接收聊天请求的飞书机器人。

插件配置保存状态目录和机器限制。其他内容均在面板中配置：在 Sources 中配置仓库和账号，在 Policy 中配置允许名单，在 Execution 中配置工作目录和命令。

<a id="permissions-and-isolation"></a>

### 权限与隔离

插件包为执行器定义了 `dsh-superagent-worker` 预设，为主 Session 定义了 `dsh-superagent-router` 预设。它们会出现在 DeepSeek Harness 的预设列表中，但仅供 dsh-superagent 使用。不要为个人 Session 选择它们，也不要禁用其配置条目。应用需要用它们来创建和恢复自己的 Session。

在 Web UI 中打开空白的受管理 Session 时，可能会选中该配置档案的默认预设。dsh-superagent 会在第一轮交互前恢复该角色专用的预设。

这两个预设都以运维者的操作系统用户身份，在 `danger-full-access` 模式下运行，并读取已订阅线程中的消息。账号包装脚本会默认选择智能体的凭据；它们并不隔离进程。执行器可以通过路径执行程序，也可以读取运维者的主目录。如果需要隔离，请使用一个只保存智能体凭据的专用操作系统账号。

<a id="prerequisites"></a>

## 前置条件

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 和 tmux。
- GitHub CLI（`gh`）2.48 或更高版本。来源的 CLI 字段必须指向 `gh` 或 `lark-cli` 的包装脚本；宿主依赖它们的子命令和输出格式。
- 一个位于 github.com 的仓库；不支持 GitHub Enterprise Server。若要为每张工单创建独立检出目录，需要保留一份本地克隆，准备命令会以它为基础添加 Git 工作树（worktree）。如果仓库使用 Git LFS，还要安装 `git-lfs`。
- 已为 DeepSeek Harness 配置模型提供方。除非 Eval 或 Execution 设置指定其他模型，否则 dsh-superagent 使用配置档案的默认模型。
- 若使用飞书：需要 `lark-cli`（已测试版本为 1.0.90），以及一个具备机器人能力的飞书或 Lark 应用。

<a id="accounts"></a>

## 账号

<a id="agent-github-account"></a>

### 智能体的 GitHub 账号

1. 为智能体创建一个 GitHub 用户账号，并赋予它仓库写入权限。不能使用 GitHub App 充当智能体账号，因为 dsh-superagent 需要读取该账号的通知。
2. 为它提供一个仅使用其自身凭据的 GitHub CLI：

   ```sh
   mkdir -p ~/.local/bin
   cat > ~/.local/bin/gh-agent <<'EOF'
   #!/bin/sh
   GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
   EOF
   chmod +x ~/.local/bin/gh-agent
   gh-agent auth login
   gh-agent api user --jq .login
   ```

   以智能体账号登录；最后一条命令会输出它的登录名。dsh-superagent 也通过同样的方式读取该登录名。

<a id="feishu-bot-optional"></a>

### 飞书机器人（可选）

1. 创建一个具备机器人能力的自建应用。授予它接收消息事件、读取和发送聊天消息，以及读取聊天成员的权限，然后发布应用版本。
2. 为机器人提供独立的 `lark-cli` 配置，将身份限制为机器人，并创建一个包装脚本：

   ```sh
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config init --app-id <cli_…> --app-secret-stdin
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config strict-mode bot --global
   cat > ~/.local/bin/lark-bot <<'EOF'
   #!/bin/sh
   export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
   exec lark-cli "$@"
   EOF
   chmod +x ~/.local/bin/lark-bot
   ```

   第一条命令从标准输入读取应用密钥。请使用独立的配置目录：`lark-cli` 允许每个应用 ID 对应一份配置，而你个人的配置中可能已经包含该应用的用户登录信息。

   宿主会拒绝能够以用户身份执行操作的 CLI。它会从机器人配置中读取应用 ID、飞书／Lark 品牌和机器人的 `open_id`。

   两份配置会共用保存在钥匙串中的应用密钥条目。若只想移除机器人配置，请删除它的目录；不要运行 `lark-cli config remove`，因为这会影响共用的密钥。

<a id="install"></a>

## 安装

在尚未存在 `superagent` 配置档案的机器上，执行以下命令块一次：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent && cd dsh-superagent
pnpm install && pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

上面固定的版本已经公开发布；其他版本必须满足 `package.json` 中的 `engines.dsh` 要求。

第四条命令从 DeepSeek Harness Web 模板创建 `~/.dsh/profiles/superagent`。第五条命令安装本插件包。设置 `DSH_HOME` 可以更改默认的 `~/.dsh` 目录位置。

创建配置档案的命令仅用于全新安装。如果配置档案已经存在，这条命令会替换它，并删除其中的 `cordis.patch.yml`，包括机器配置。升级时不要使用这条命令。

<a id="machine-configuration"></a>

## 机器配置

将 [examples/profile.cordis.patch.yml](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/examples/profile.cordis.patch.yml) 复制到 `~/.dsh/profiles/superagent/cordis.patch.yml`，替换该文件，并填写所有 `<placeholder>` 占位内容。如果使用飞书，还要在其中启用 `dsh-superagent-feishu` 条目。`webserver` 条目用于设置面板端口。

| 键 | 是否必填 | 含义 |
| --- | --- | --- |
| `stateDir` | 是 | 绝对路径：保存数据库、锁、`worker-bin` 中的命令代理脚本、智能体的 git 配置，以及默认位于 `work` 下的工单目录。升级前请备份 |
| `promptMaxChars`, `admissionContextMaxPages`, `admissionContextTimeoutMs` | 否 | 分别规定可保存的提示词配置大小上限，以及一次研判最多读取多少页线程内容、允许花费多少时间 |

首次启动后，可以在 **Settings → Deployment** 或 DeepSeek Harness Plugin Manager 中编辑机器限制。修改无须重启即可生效。`stateDir` 不能在那里修改。无效值或未知配置键会阻止启动；日志会指出对应的键。

<a id="start-and-stop"></a>

## 启动与停止

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 会在名为 `dsh-superagent` 的 tmux 会话中运行 `dsh --profile superagent`；使用其他配置档案时，会话名为 `dsh-<profile>`。它最多等待一分钟，直到运行时就绪或报告配置要求。首次运行时，它会输出缺少的配置信息。它拒绝启动重复实例，也能识别在脚本之外启动的实例。

`stop` 和 `restart` 会请求优雅关闭，并最多等待执行器五分钟。它们绝不会强制杀死进程，也不会操作其他 tmux 会话。

`status` 会输出进程 ID、健康状态和配置问题。若使用不同的配置档案或端口，请传入 `--profile` 和 `--url`，或设置 `DSH_SUPERAGENT_PROFILE` 和 `DSH_SUPERAGENT_URL`。

`status`、`start` 和 `restart` 会以以下四种退出码之一退出：

| 退出码 | 含义 |
| --- | --- |
| 0 | 服务正常运行 |
| 1 | 服务未运行、出现两个实例、没有响应，或运行时报告错误；重启可能解决问题 |
| 2 | 服务正在运行，但运行时之外的某个部分报告错误，例如来源或策略同步（`sync:<source id>`）；重启无法解决问题 |
| 3 | 服务正在运行，等待完成配置；重启无法解决问题 |

尚在启动中的来源算作正常运行。配置检查无法检查的飞书机器人也算正常。

`status` 在 `errors` 中返回失败的部分及其详细信息，在 `setup` 中返回配置要求。`start` 和 `restart` 将这些问题逐行输出；如果服务始终未启动，则以退出码 1 退出。只有主动执行 `stop` 才会暂停自动恢复。

每个配置档案的日志和控制文件都保存在 `.artifacts/` 下：

- 启动日志：`dsh-superagent-<profile>-<date>-<time>-<pid>.log`，其中 pid 是脚本的进程 ID。保留最新的 20 份日志。
- 控制文件：`control/profiles/<profile>/`，其中包含停止标记、锁和进程 ID。

打开日志中输出的 URL，然后在侧栏中选择 **Superagent**。

<a id="hourly-check"></a>

### 每小时检查

在你平时使用的 DeepSeek Harness 配置档案中创建每小时检查，不要创建在 `superagent` 中。智能体服务停止运行时，这项检查仍须继续运行。请让该配置档案中的智能体使用下面的提示词创建每小时执行的计划，并填写检出目录路径：

```text
Check dsh-superagent: run `scripts/dsh-superagent --profile superagent status` in <checkout>. Exit 0: do nothing and say nothing. Exit 2: do not restart; tell me each part under `errors` with its detail, unless you already told me about the same ones. Exit 3: do not restart; tell me what `setup` says setup needs, unless you already told me. If it reports manually_stopped: true, leave it stopped. Otherwise (exit 1) run `scripts/dsh-superagent --profile superagent --respect-stop restart`, which exits the same way: 0 means it recovered, so say nothing; 2 and 3 are reported as above; if it exits 1 or fails otherwise, read the newest .artifacts/dsh-superagent-superagent-*.log and the Troubleshooting section of docs/006-deployment.md, fix what you can, and tell me what was wrong and what you did.
```

> 上述巡检提示词的中文翻译：检查 dsh-superagent：在 `<checkout>` 中运行 `scripts/dsh-superagent --profile superagent status`。退出码为 0：不执行操作，也不发消息。退出码为 2：不要重启；告诉我 `errors` 中每个部分及其详细信息，除非你已经报告过相同问题。退出码为 3：不要重启；告诉我 `setup` 表明还需要哪些配置，除非你已经报告过。如果结果包含 `manually_stopped: true`，就保持停止状态。其他情况（退出码为 1）则运行 `scripts/dsh-superagent --profile superagent --respect-stop restart`。它使用相同的退出码：0 表示已恢复，因此不发消息；2 和 3 按前述方式报告；如果以 1 退出或发生其他失败，请阅读最新的 `.artifacts/dsh-superagent-superagent-*.log` 以及 `docs/006-deployment.md` 中的故障排查部分，修复你能够修复的问题，并告诉我出了什么问题、你做了什么。

`--respect-stop` 会保留主动停止的状态。只有 `stop` 会写入停止标记；`start` 会清除它。`restart` 不写标记，因此即使重启时关闭过程超时，检查任务仍可将其恢复。

<a id="settings"></a>

## 设置

配置检查必须成功，轮询和工作才会开始。Overview 会列出缺失的条件，并链接到相应设置。修改设置后会重新运行配置检查。如果你在应用之外修复了问题，例如 CLI 身份认证，请选择 **Check again**。

1. **Sources。** 设置 GitHub 仓库（`owner/name`）和智能体 CLI，例如 `/home/me/.local/bin/gh-agent`。请求通常通过智能体账号的通知到达。在新建 Issue 或新评论中提及该账号，会在下一次轮询时收到；通过编辑已有 Issue 正文补加的提及不会被收到。此时请另发一条新评论。若使用飞书，请设置机器人 CLI、聊天列表、每个聊天接收全部消息还是仅接收提及，以及是否接收私聊消息。
2. **Policy。** 两条示例规则均可编辑或删除。在 **On each Source** 下添加可以请求工作的 GitHub 账号和飞书用户；名单中至少要有一个人，请求才可能成为工单。使用飞书时，可以选择一个聊天，从其成员同步允许名单；同步在启动时和点击 **Sync now** 时进行。
3. **Execution。** 设置目录根路径和准备命令。要让每张工单拥有自己的仓库检出目录，可以使用：

   ```sh
   git -C /home/me/src/widgets fetch --no-tags "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
   git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
   ```

   将个人 CLI，例如已登录你本人账号的 CLI，列入被阻止的命令。
4. **Prompts。** 设置团队内容区：GitHub 身份披露语句、评审机器人、提交作者和自动合并条件。

<a id="management-api"></a>

### 管理 API

面板使用本地 HTTP API，智能体也可以调用它。`SETUP.md` 使用它保存设置，服务脚本则读取 `GET state`。

向 `http://127.0.0.1:<port>/orchestrator/api/` 发送请求，并携带 `x-orchestrator-request: 1` 请求头。Host 必须是 `127.0.0.1:<port>` 或 `localhost:<port>`。缺少请求头，或 `Origin` 来自其他站点时，会收到 403。

通过 API 保存配置具有面板的权限。它可以修改执行环境和用于配置账号的字段；通过主 Session 工具保存配置则没有这些权限。

智能体可以依赖下列路由和字段。`src/app/api.ts` 中的其他路由为面板服务，会随面板变化。

| 请求 | 响应 |
| --- | --- |
| `GET state` | 各页面读取的状态：`setup`，包含 `problem` 和 `feishuProblem`，各自为 `null` 或包含 `place`、`code`、`values`、`detail` 的配置问题，账号解析完成后还包含 `accounts`；`health`，按下表为每个部分提供一条记录，包含 `state`（`starting`、`ready`、`paused` 或 `error`）、`detail` 和 `at`；`load`，包含正在运行的执行器 Session 数 `running`、排队工单数 `queued` 和 `maxWorkers`；以及 `paused`、`masterId` |
| `GET settings` | `blocks`：`policy`、`eval`、`executor`，各含 `value` 和 `revision`；`sources`：来源实例，各含 `id`、`kind`、`installed`、配置块的 `value`（形如 `{"enabled": …, "config": {…}}`）和 `revision`、该类型的表单，以及存在时的配置 `problem`；`policyVersion`；`environment`：执行环境的 `root` 和 `variables` |
| `GET sources` | 每个来源的状态：`state`（`missing`、`off`、`error`、`setup`、`starting`、`paused` 或 `ready`）、各部分的 `health`、配置 `problem`、最近一天的 `stats`、当天被 `filtered` 的触发，以及当天被 `dropped` 的消息。丢弃记录按最新在前排列，各含 `channel`、`preview`；若消息属于某张工单，还包含 `ticket`（`id` 和 `title`）；若存储中有对应线程，则包含 `thread`；另含 `dropped.at` 和 `dropped.reason`，原因值为 `channelGone` 或 `channelIgnored` |
| `PUT settings/<block>`，请求体为 `{"value": …, "revision": n}` | 保存一个配置块：返回 `saved`、该块的新 `revision`、`policy_changed` 和 `restarting`。`value` 是 `GET settings` 返回的整个配置块；缺少的键使用默认值 |
| `POST executor/pause`，请求体为 `{"paused": true}` 或将值设为 `false` | 暂停或恢复处理：返回 `paused` |
| `POST setup/check` | 通过重启 dsh-superagent 重新运行配置检查：返回 `restarting: true`。没有待修复的配置问题时，请求会被拒绝 |
| `POST policy-sync/<source id>` | 立即同步该来源的允许名单：返回 `policy`，即已保存的该来源策略部分 |
| `GET prompts`、`PUT prompts`，后者的请求体为 `{"config", "baseRevision"}` | 读取提示词配置：`current`，包含 `revision` 和 `config`；`defaults`；`sections`，即按 Session 读取顺序排列的团队内容区；`skillName`，团队技能名称的格式；`builtin`，团队技能不能占用的名称；`skillPrefix`，Session 加载技能名称时使用的前缀。保存时返回新的 `revision` |
| `POST tickets/<id>/retry`、`POST tickets/<id>/stop` | 与面板一样，重试或停止工单 |

健康状态条目按职责分组：

| 条目 | 报告内容 |
| --- | --- |
| `runtime` | 应用本身 |
| `models` | 正在等待模型预算重试的 Session |
| `evals` | 最近一小时内未完成，且之后没有新研判或未被忽略处理的评估 |
| `assessment`, `launch`, `delivery`, `relay`, `notice`, `reply` | [003 执行](003-execution.md)描述的已持久化工作；delivery 统计等待中的工作，reply 则涵盖渠道消息 |
| `worker` | 执行器 Session 的事件处理；首次失败后才出现 |
| `github`, `feishu`, `history` | 来源轮询、事件和历史恢复 |
| `sync:<source id>` | 允许名单同步 |

请求被拒绝时，响应包含状态码和 `error`，后者是英文错误文本：

- 400：无效值（`issues` 中每项含 `path`）、账号检查失败（`setup`）、固定类型的拒绝（`code` 和 `values`），或者请求体不是 JSON 或体积过大。
- 404：目标不存在。
- 409：修订版本已经过期。
- 500：其他情况。

各配置块如下：

- `source:github`、`source:feishu`：`{"enabled": true, "config": {…}}`。GitHub 配置包含 `repository` 和 `cli`，以及默认关闭的 `observeIssues`、`observePulls`、`observeComments`、`observeCi` 开关；启用后，分别接收仓库中所有新建 Issue、新建拉取请求、评论，以及默认分支的 CI。飞书配置包含 `cli`、形如 `[{"chatId": "oc_…", "name": "…", "listen": "mentions"}]`（也可以用 `"all"`）的 `chats`、`directMessages`，以及值为 `"none"` 或 `"mentions"` 的 `otherChats`。
- `policy`：`{"rules": […], "sources": {"github": {"allowed": [{"id": "<login>"}], "bots": [{"id": "dependabot[bot]"}]}, "feishu": {"allowed": [{"id": "ou_…", "name": "…"}], "syncChat": "oc_…", "admission": {"oc_…": {"unlistedAddressed": "explicit"}}}}}`。`bots` 列出其消息可以成为推断候选的机器人。在飞书中，`admission` 行以聊天 ID、`other` 或 `direct` 为键；在 GitHub 中，以 `issues`、`pulls`、`comments` 或 `ci` 为键。未设置的单元格使用默认值，参见 [002 授权](002-authorization.md)。
- `eval`：`mode`、`timeoutSeconds` 和 `model`。
- `executor`：`model`、`maxWorkers`、`directory`、`prepare` 和 `blockedCommands`。处理暂停状态是独立设置 `paused`，通过 `POST executor/pause` 设置。

修改仓库或 CLI 时，会先执行相应的配置检查。检查失败返回 400，并在 `setup` 中提供详情。成功修改账号，或在配置尚未完成时保存任何设置，都会返回 `restarting: true`。应用重启期间，API 会暂时不可用。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' $api/settings | jq '.sources[] | select(.id == "github") | .revision')
curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' $api/settings/source:github \
  -d "{\"value\": {\"enabled\": true, \"config\": {\"repository\": \"acme/widgets\", \"cli\": \"$HOME/.local/bin/gh-agent\"}}, \"revision\": $rev}"
```

<a id="setup-checks"></a>

## 配置检查

配置检查与执行器的 shell 一样，使用 DeepSeek Harness 的子进程服务。它会移除名称中包含 KEY、PASSWORD、SECRET 或 TOKEN 的环境变量。

配置检查会验证 GitHub 仓库和智能体的推送权限，写入执行器环境，并检查 git 是否仅对 github.com 使用智能体凭据。飞书配置检查失败只会禁用飞书来源。

配置检查写入的进程环境包括：

- **命令代理脚本（shim）**：位于 `<stateDir>/worker-bin`，排在 `PATH` 最前面。`gh` 运行智能体 CLI，`lark-cli` 运行机器人 CLI，被阻止的命令会失败。代理脚本在运行目标程序前，会将自身目录从 `PATH` 中移除。
- **git 配置**：位于 `<stateDir>/worker-gitconfig`，替代运维者的全局和系统 git 配置。它将智能体 CLI 设为 github.com 的凭据助手，将 GitHub SSH 地址改写为 HTTPS，并定义 Git LFS 过滤器。个人 git 设置，例如 HTTP 代理，不会生效。这里不设置提交身份；git 绝不会提示输入凭据，并且会移除 `GH_TOKEN` 和 `GITHUB_TOKEN`。

如果 `worker-bin` 或 `worker-gitconfig` 中存在并非由 dsh-superagent 写入的文件，配置检查会停止。如果工单目录位于某个 git 仓库中，而该仓库自身的配置又添加了凭据，该工单会失败。

<a id="try-it"></a>

## 试用

使用允许名单中的 GitHub 账号，创建 Issue 或发布包含 `@<agent login> <request>` 的评论。在一个轮询周期内（默认一分钟），Overview 应当出现一张工单。智能体会确认请求，在自己的 Session 和目录中工作，并在原对话中交付结果。使用飞书时，在已经配置的聊天中提及机器人即可。

<a id="upgrade"></a>

## 升级

1. 等待所有执行器都不再处于某一轮交互的执行中途。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 运行 `git pull && pnpm install && pnpm build`。
4. 运行 `scripts/dsh-superagent restart`，并检查其状态。

重启会结束正在运行的派工，并携带恢复说明启动替代派工。团队文本和已覆盖的提示词部分会保留；没有覆盖的部分则跟随新的默认值。升级后，请检查标记为 **Default updated** 的部分。

经较新版本迁移的数据库无法由旧版本打开。因此，回滚需要使用升级前的备份。升级 DeepSeek Harness 本身后，应重启正在运行的实例，让面板加载与之匹配的构建产物。

<a id="agent-presets"></a>

### 智能体预设

worker 和 router 预设会在 shell 命令运行 5 秒后将其移到后台。Session 缺少自己的预设时无法加载。在这些 Session 仍被使用期间，请保留 `cordis.patch.yml` 中的 `dsh-superagent-worker` 和 `dsh-superagent-router` 条目；否则恢复时会报 `agent-preset/not-found`。

<a id="troubleshooting"></a>

## 故障排查

配置问题会以面板所用语言显示在概览中，并附上修复入口；日志和 API 则显示下列英文文本。

| 现象 | 原因与修复方法 |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub 卡片中设置智能体 CLI |
| `The agent GitHub CLI was not found.` | 将其设为包装脚本的绝对路径，或 `PATH` 中的命令名 |
| `The agent GitHub CLI cannot read its account` | 用 `auth login` 登录该包装脚本所用账号 |
| `@… cannot read …` 或 `@… cannot push to …` | 赋予智能体账号仓库写入权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片中按 `owner/name` 格式设置仓库 |
| 工单报 `The prepare command failed: …` 并失败 | 输出会指出原因，例如克隆路径错误；修正命令后重试工单 |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更高版本 |
| 工单报 `The git configuration of the repository in … sets …` 并失败 | 从克隆仓库的 `.git/config` 中移除消息指出的凭据设置或 `http.*.extraheader` 设置 |
| `… was not written by dsh-superagent` | 将指定文件移出 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig`，或者选择其他 `stateDir` |
| `The bot lark-cli allows user identities` | 按前面“飞书机器人”部分的方式，让飞书来源指向仅允许机器人身份的配置 |
| `The bot lark-cli does not work.` 或 `The bot identity of app … is …` | 检查配置中的应用 ID 和密钥 |
| `The Feishu bot is not a member of the chat.` | 在聊天设置中将机器人加入聊天；下一次尝试便会成功 |
| `status` 以 2 退出 | 某个部分报告错误，重启无法修复：`errors` 会按“管理 API”部分所列名称指出每个部分及详情；该部分恢复后会自行清除对应条目 |
| `status` 以 3 退出 | 等待完成配置：`setup` 说明缺少什么，概览会链接到可修复它的设置 |
| `status` 以 1 退出 | 服务未运行、运行了两个实例、没有响应，或运行时报告错误；输出和最新日志会说明具体是哪种情况 |
| `… timed out after 90 s`、`… was stopped by …`、`… exited with code …` 或 `… could not run: …` | CLI 失败，却没有输出原因；请以运维者的用户身份手动运行指定程序，查看它自己的错误信息 |
| 面板提示 dsh-superagent 没有响应 | 它正在重启，或因配置问题未能启动；DeepSeek Harness 日志会指出原因 |
| 日志出现 `dsh-superagent configuration has keys it does not use` | 移除指定的键；消息会说明每个值现在应配置在哪里 |
| 日志提示插件与正在运行的 dsh 不兼容 | 安装版本处于 `engines.dsh` 范围内的 DeepSeek Harness |
| 升级 DeepSeek Harness 后，面板显示“Failed to load plugins” | 重启正在运行的实例 |
| 提及智能体后没有任何反应 | 检查发起者是否位于该来源的允许名单中、来源是否已开启、处理是否未被暂停。在 GitHub 上，通过编辑 Issue 正文补加的提及不会被接收；请在评论中提及智能体 |

<!-- original-license:start -->
<details>
<summary>原项目 MIT 许可与翻译说明</summary>

本篇为非官方中文翻译；中文翻译新增部分同样按 MIT 许可提供。原文来源和固定版本见篇首。

```text
MIT License

Copyright (c) 2026 DeepSeek Harness Team

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

</details>
<!-- original-license:end -->
