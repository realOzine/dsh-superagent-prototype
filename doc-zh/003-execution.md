<a id="003-execution"></a>

# 003 执行

> 原文版权：Copyright (c) 2026 DeepSeek Harness Team；MIT License。
> [英文原文](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/003-execution.md) · 固定版本：`6337fc823ce20251c8de8085948165c81aaa701e`  
> 中文翻译，非官方；代码与配置保持原样。

本文追踪一个已接单工单的执行、交付和恢复过程，建立在 [001 领域模型](001-domain.md) 和 [002 授权](002-authorization.md) 之上。

<a id="ticket-stages"></a>

## 工单阶段

每个尚未结束的工单都有一个阶段，由它正在等待什么决定。自上而下读取下表，以第一个匹配的行为准。

| 阶段 | 条件 | 面板标签 |
| --- | --- | --- |
| person / delivery | 向执行器 Session 投递输入失败；2 秒后重试，之后延迟逐次翻倍，最多为一分钟；投递成功、新建派工或工单结束时清除此状态 | 输入未送达 |
| person / answer | 存在待人工决定的问题 | 等待回答 |
| person / review | 最新报告为 `ready` | 等待审查 |
| person / blocked, failed | 最新报告为 `blocked` 或 `failed`，且此后没有人的消息到达该工单 | 受阻 · 失败 |
| working / queued | 没有派工，或派工为 pending | 排队中 |
| idle | 最新报告为 `waiting`，或派工已结束 | 等待触发 |
| working | 其他所有情况 | 工作中 |

`person` 阶段表示工单必须获得人的输入才能继续。新的人类消息会将 `blocked` 或 `failed` 工单交回执行器继续处理，因此工单会进入下一个匹配阶段，通常是 `working`。

`idle` 工单已经结束当前回合，且没有尚待回答的问题。已订阅线程上的触发可以使它继续。

<a id="the-executor"></a>

## 执行器

每个工单都会获得一个名为 `worker-<ticket id>` 的执行器 Session，以及一个以工单短名称命名的目录。目录创建在配置的执行根目录下，该根目录默认为 `<stateDir>/work`。

执行器 Session 以操作人员的操作系统用户身份运行命令，无需审批。其 `dsh-superagent-worker` 预设包含工作所需的工具，并在 5 秒后将前台 shell 命令转为后台作业。这样，worker 就能在下一步接收新消息，而不必等长时间运行的命令结束。

主 Session 使用 `dsh-superagent-router`，shell 和作业等待也受相同的 5 秒限制，并使用一组更小的路由工具。Session 会保留创建时选定的预设。worker 的子 Agent 使用 DeepSeek Harness 的 `standard` 预设。

<a id="background-work-and-worker-slots"></a>

### 后台工作与 worker 槽位

如果回合结束时仍有 shell 作业或子 Agent 在运行，工单会保留其 worker 槽位，并继续处于工作中状态。它不会被报告为等待中，主 Session 也不会收到等待报告。作业或子 Agent 完成后，会带着结果在同一个槽位中唤醒执行器。只有当回合结束时已没有后台工作，执行器才会正常报告等待状态。

每个回合结束后，后台工作最多还有 30 分钟可以完成。到达时限后，shell 作业会被终止，工单被报告为等待中。报告会包含执行器最后说的话，以及哪些作业被终止、哪些子 Agent 仍在运行。子 Agent 被允许继续。执行器会在下一个回合读取被终止作业的结果。

交给人处理时，会立即释放槽位，报告状态保持有效。`ready` 报告或提问后，只要此后没有新输入到达，就适用这一规则；任何 `blocked` 或 `failed` 报告后也适用。

交接时，子 Agent 会停止。shell 作业，例如供人审查的服务器，可以继续运行，直到派工结束。这些作业不占槽位，也不受 30 分钟时限约束，即使后来又发生了其他回合也是如此。它们完成时不会唤醒执行器；执行器在下一个回合读取结果。

人也可以直接在执行器 Session 中发消息来启动一个回合。由此产生的后台工作遵循相同规则。

派工结束时，其 shell 作业会被终止，且不会唤醒 Session。其子 Agent 会被释放，由这些子 Agent 完成事件唤醒的任何回合都会被丢弃。替代派工及其等待中的输入，要在上述关闭过程完成后才启动。

在 DeepSeek Harness 进程中停止或重启插件，也使用同样的关闭路径。尚未被纳入 Session 历史的输入会保留。关闭时，对每个 worker 的后台工作最多等待 10 秒；进程重启则会直接结束这些工作。忙碌时被中断的派工，会带着一条恢复说明重新启动。

worker 的子 Agent，包括嵌套子 Agent，使用 DeepSeek Harness 的 `standard` 预设和 worker 的权限。它们只保留 worker 获准使用的工作工具，并在工单结束时失去这些工具。使用 `standard` 时，shell 命令在 60 秒后转入后台。子 Agent 结束时，其作业会被终止。

<a id="preparing-a-ticket-directory"></a>

### 准备工单目录

在首次派工前，如果工单目录不存在，准备命令会在执行根目录中通过 `sh` 运行一次，最多有十分钟完成。如果没有准备命令，worker 会在一个空目录中启动。

命令失败会使工单失败，并记录命令输出。重试，或者人在工单某个线程中发消息，都会再次尝试准备。典型的准备命令使用 `git worktree` 创建检出目录。当某个拉取请求的分支路径最后一段与工单短名称相同时，该拉取请求就属于这个工单。

<a id="concurrency"></a>

### 并发

从启动开始（包括准备阶段），或者从投递输入以唤醒空闲执行器时起，工单就占用一个 worker 槽位。活跃回合的整个过程中也一直占用该槽位。

配置的并发限制作用于这些槽位，但会为人的消息或状态请求额外保留一个槽位。普通槽位已满时，这类唤醒可以使用额外槽位，而且会先于新启动的工作取得下一个空闲槽位。因此，新工作无法阻止人继续处理已有工单。

其他输入，例如 CI 结果，要等待普通槽位。启动任务在槽位空闲前处于 `working / queued`。多个启动任务并发运行，因此缓慢的准备命令只会拖延它自己的工单。

<a id="execution-variables"></a>

### 执行变量

准备命令和执行器指令会收到相同的执行变量。命令通过环境变量读取；执行器则从提示词中读取变量值和说明。这些变量都不包含凭据。

| 变量 | 注册者 | 值 |
| --- | --- | --- |
| `DSH_SUPERAGENT_DIR`、`DSH_SUPERAGENT_NAME`、`DSH_SUPERAGENT_TICKET` | 执行器 | 工单目录、短名称和 id |
| `DSH_SUPERAGENT_BLOCKED_COMMANDS` | 执行器 | 执行器不得运行的命令 |
| `DSH_SUPERAGENT_GITHUB_REPOSITORY` | GitHub | 来源监听的仓库，格式为 `owner/name` |
| `DSH_SUPERAGENT_GITHUB_ACCOUNT`、`DSH_SUPERAGENT_GITHUB_COMMIT_EMAIL` | GitHub | Agent 账号及其提交邮箱 |
| `DSH_SUPERAGENT_GITHUB_NUMBER`、`DSH_SUPERAGENT_GITHUB_PR_HEAD` | GitHub | 发起该工单的 Issue 或拉取请求；若为拉取请求，还包括其 head ref |
| `DSH_SUPERAGENT_FEISHU_BOT`、`DSH_SUPERAGENT_FEISHU_CHAT` | 飞书 | 机器人，以及发起该工单的群聊 |

工单专属值来自其目录和起始触发。其他值来自当前部署设置。只有面板可以修改执行环境设置；发给主 Session 的文本不能改变已配置的 shell 命令。

worker 通过 Agent 账号的 `gh` 访问 GitHub，通过机器人的 `lark-cli` 访问飞书。被禁用的命令按名称调用时会失败。

这些包装器和目录建立的是默认行为，不是安全边界。worker 仍然可以通过绝对路径执行程序。

<a id="merging"></a>

## 合并

自动合并由 **Delivery requirements（交付要求）** 提示词约束。宿主没有专门针对合并的决策逻辑。

默认情况下，执行器绝不以 Agent 账号启用自动合并：GitHub 会把合并归于启用自动合并的账号。交付要求必须指定另一个账号用于合并的 CLI。如果没有这样的 CLI，就由人合并拉取请求。

默认指令只允许对为该工单创建的拉取请求启用自动合并，而且必须满足以下条件：拉取请求已准备就绪，所需批准覆盖 head commit，新增行数与删除行数之和少于 200 行。推送下一次修改前，执行器会撤回自动合并。

没有正文的审查，例如批准，也会作为触发到达，并能唤醒等待审查的执行器。这些指令约束的是 Agent，不是 GitHub 的合并规则。要强制要求批准，应在分支保护中启用批准要求，并撤销过时批准。省略合并 CLI，或者删除自动合并段落，就能要求手动合并。

<a id="how-a-ticket-ends"></a>

## 工单如何结束

- **已就绪但没有拉取请求：** `ready` 报告会将工单以已完成状态结束。
- **没有事情需要做：** 如果执行器发现没有需要人采取行动的工作，例如请求无法复现，或者已被其他工作取代，就报告 `dropped`。工单会结束且不发消息，除非匹配规则要求发送消息；后来人在其线程中发消息时，工单可以恢复。

<a id="pull-requests"></a>

### 拉取请求

执行器报告起始拉取请求后，工单就会关注该拉取请求。对于从工单分支创建的每个拉取请求，只要执行器报告了它，或者 GitHub 来源发现了它，工单也会关注。重新打开的拉取请求会再次被关注。

当工单创建的拉取请求中已没有任何一个仍然打开时，工单完成。如果工单未创建过拉取请求，那么它关注的所有拉取请求都必须已关闭。之后，工单会带着最后一次相关关闭的结果结束，并保留订阅，以允许后来的人类评论使它恢复。

如果恢复、重试或故障恢复后的工作尚未完成，关闭拉取请求不会结束工单。还有一种情况也要等待：执行器已经确认或回答了自己拥有处理权的后续消息，但在读取该消息后，尚未提交非 progress 报告，也尚未结束一个回合。之后提交的 `ready` 报告，会在满足同样的拉取请求完成规则时结束工单。

聊天渠道会收到一条关于拉取请求结果的最终通知。GitHub 本身已经在拉取请求页面上展示结果。移除工单对某个拉取请求的订阅，也会使它停止关注该拉取请求。

因此，“等待审查”指的是带有拉取请求的 `ready` 报告。

<a id="follow-ups-and-context-threads"></a>

### 后续消息与上下文线程

执行器负责判断工单自身线程中的人类后续消息。它处理范围内的请求，并退回超出范围或属于其他工单的请求。当没有任何执行器拥有该消息的处理权时，它才成为主 Session 的候选。除此之外，主 Session 只能应人的请求，针对后续消息采取停止工作的行动。

当 Session 将输入纳入某个步骤，并在之后提交报告或结束回合后，该输入才算已读。工单结束时，未读且未被确认、回答或退回的消息，会按 [002 授权](002-authorization.md) 所述返回评估。已读但未采取行动的消息，例如感谢，不会返回评估。

仅为上下文而关注的线程仍会投递消息，但不会使工单取得人类后续消息的处理权，也不会恢复已结束的工单。这些消息可以独立接受研判。

只要有尚未结束的工单订阅某个线程，来源就会轮询该线程；工单结束后，还会继续轮询两周，起算时间取最后一个触发与订阅工单结束时间中的较晚者。此后，消息只通过常规接收路径到达：GitHub 通知或飞书实时事件。

<a id="failure-and-recovery"></a>

## 失败与恢复

<a id="executor-blocked-or-failed"></a>

### 执行器受阻或失败

当执行器 Session 报告 `blocked` 或 `failed`，或者启动失败或发生错误并被记录为 `failed` 时，工单不再接收自动输入。

对于模型供应商或网络的临时故障，例如 HTTP 408、429、5xx，或连接被拒绝、重置、超时，会先按逐步增长的退避间隔重试三次：启动任务会重新运行，回合失败的 worker 会收到继续执行的指示；失败的准备命令不会自动重试。

一旦进入失败状态，CI 和机器人消息仍会记录在工单上，但不会投递，也不会唤醒 Session。

工具仍然可用，因此人可以直接在 Session 中继续工作。

失败的启动任务不会自行重试。

人的消息、之后提交的既非 `blocked` 也非 `failed` 的报告，或者重试操作，都可以解除暂停。

该状态根据已存储的报告和输入计算，因此重启后仍会保留。

<a id="retry"></a>

### 重试

尚未结束的工单处于 `person / delivery`、`person / blocked` 或 `person / failed` 时，可以重试。对这些情况，恢复方法都是新建派工：重新启动 worker，并接收等待中的输入。

一旦人的消息到达受阻或失败的工单，执行器就已经拥有该消息，因此工单显示为工作中；如果是在失败后重新启动，则显示为排队中。此时不再提供重试入口，除非该消息投递失败。

重试会结束当前派工，并在相同的执行器 Session、目录、分支和拉取请求中启动新派工。

执行器模型设置只在创建 Session 时生效；已有 Session 保留原模型。

<a id="restart"></a>

### 重启

运行中的派工会以“host restarted”（宿主已重启）结束，新派工会带着恢复说明启动，按最旧工单优先的顺序执行。恢复说明要求执行器在重复任何外部行动前，先检查目录、Git 历史和 GitHub。

正在等待、已就绪、受阻或失败的派工维持原状，除非在相应报告之后有输入到达；这种情况下，它们也会重新启动。

Session、目录、订阅、等待中的消息以及下文所述的已存储工作，都会在重启后保留。每个执行器 Session 已将哪些消息纳入历史的记录也会保留：步骤在认领消息之后、调用模型之前，就会将消息纳入历史。如果某条消息已被一个后来因重启中断的回合纳入历史，那么之后一旦有报告或回合结束，它就算已读；如果步骤已经认领消息，但在写入历史前被取消或出错中断，则该消息仍算未读。

重试或重启时，新派工会再次收到这些消息：它们曾投递给被替代的派工，但尚未被其执行器 Session 纳入历史，可能因为取消或步骤中断而丢失。如果 Session 中还等待着一份副本，无论这份副本是重启时保留的，还是崩溃后从持久化恢复的，都会在新派工的第一条消息前被移除，以确保新投递是唯一副本。

<a id="stored-work"></a>

### 已存储的工作

dsh-superagent 在作出决定之后要执行的事情，例如向主 Session 转发候选、后续消息或报告、启动派工，或投递输入，都会先存储，再执行。

转发或投递失败时，例如无法连接其 Session，会在 2 秒后重试，之后延迟逐次翻倍，最多一分钟；启动任务失败则会使其工单失败，除非属于上文所述的临时故障。

每类工作只有在其中一项连续失败三次后，才会作为错误出现在概览的系统问题中；输入投递失败则显示在相应工单上。

发往主 Session 的转发保持顺序，通知也保持顺序；某一项失败会阻塞同类中后续的项目。

<a id="source-failure"></a>

### 来源故障

GitHub 来源会分别读取每个 Issue、拉取请求和线程。读取失败会在对应线程上保留错误，下次轮询时再次读取，并计入来源错误；默认连续三次轮询失败后，来源才会报告错误。任何一次成功轮询都会重置计数。

Agent 失去私有仓库访问权限时，GitHub 也会返回 404，因此 404 会被重试。

对于通知指向但持续读取失败的 Issue 或拉取请求，系统会在第一次读取失败一天后放弃该项；如果 GitHub 报告其已被删除，则立即放弃。此后它不再导致轮询失败，其线程仍保留错误，来源状态中也会指出该项。

其读取位置保持不变，因此后来的通知或线程轮询，仍能读取此前失败时遗漏的内容。

飞书事件消费者会报告每次失败；对已订阅话题的轮询，会在各自线程上保留每次失败。

GitHub 列表或飞书分页中的格式错误条目会被跳过，来源状态会记录其数量。

<a id="channel-failure"></a>

### 渠道故障

发出的消息会先存储，再作为已存储工作发送；重试采用退避策略，从 5 秒开始，逐次翻倍，最多 300 秒。

同一工单发往同一对话的消息，按入队顺序发送，以保证确认消息早于之后的回答或结果：只要该工单发往该对话的更早消息尚未发出，后面的消息就等待，直到前面的消息连续失败三次并被列为问题。

其他工单的消息，以及不属于任何工单的消息，不需要等待它。

有过失败尝试的消息会一直被视为系统问题，直到成功发出或被撤回。

只有在来源不再监听该对话，或者来源明确表示该对话永远无法接收该消息时，才会丢弃消息；来源关闭期间，消息会等待。

GitHub 上 Issue 已删除或对话已锁定，以及飞书上消息已撤回或已无法找到时，来源会返回这种无法投递的结果。

回复的审查评论如果已不存在，消息会改发到拉取请求的对话中。

被丢弃的消息仍保留记录，包括何时丢弃、为何丢弃，以及属于哪个工单，从而确保它不会再次入队；被丢弃的确认消息会显示为已丢弃，而不是已发送。

<a id="pause-processing"></a>

### 暂停处理

暂停派发、新评估、评估超时、输入投递和回复；已经在途的启动任务会等待处理恢复。

来源继续记录触发，正在运行的执行器回合不会被中断。

发往已结束工单的人类消息会等待：暂停后的第一轮处理会在其授权仍符合当时策略时，恢复该工单。

进行中的评估不会在暂停期间过期；其截止时间会按暂停持续的时长向后延长。

<a id="invalid-configuration"></a>

### 无效配置

如果已存储的来源配置被其来源类型拒绝，该来源实例会被禁用。

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
