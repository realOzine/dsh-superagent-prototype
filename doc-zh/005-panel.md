<a id="005-panel"></a>

# 005 面板

> 原文版权：Copyright (c) 2026 DeepSeek Harness Team；MIT License。
> [英文原文](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/005-panel.md) · 固定版本：`6337fc823ce20251c8de8085948165c81aaa701e`  
> 中文翻译，非官方；代码与配置保持原样。

在 DeepSeek Harness Web 侧边栏中打开 **Superagent**，即可监督 Agent。面板优先展示需要人处理的工作，其次是系统健康状况和历史记录。本文建立在 [000 设计理念](000-philosophy.md) 到 [004 提示词](004-prompts.md) 的内容之上。

<a id="three-questions"></a>

## 三个问题

打开面板时，应按以下顺序回答三个问题：

1. 现在有没有事情需要我处理？
2. 系统是否健康，是否正在做正确的事情？
3. 某项工作从触发、评估到工单，完整历史是什么？

<a id="tabs"></a>

## 标签页

标签页沿着工作流程排列：**Overview（概览）**、**Source（来源）**、**Trigger（触发）**、**Eval（评估）**、**Execution（执行）** 和 **Settings（设置）**。在仓库和账号设置成功之前，概览会显示缺少的内容，并链接到相应设置。

| 标签页 | 展示内容 |
| --- | --- |
| Overview（概览） | 需要采取行动的工单、系统问题、最近 24 小时的活动计数，以及执行器负载。每个区块都链接到其详细页面 |
| Source（来源） | 来源状态、设置问题，以及非零计数。被过滤的触发和被丢弃的消息，使用与概览相同的 24 小时时间窗口 |
| Trigger（触发） | 可按已订阅、评估中、24 小时内有新触发，或全部线程筛选。每个线程按最新优先展示触发，并显示决定和订阅 |
| Eval（评估） | 决定及其原因和匹配规则 |
| Execution（执行） | 执行器状态、暂停处理，以及可按未结束、待启动、工作中、等待触发或已结束筛选的工单。工单详情展示可执行操作、授权、等待中的输入、订阅和派工 |
| Settings（设置） | 来源、评估、执行、提示词和部署。每组都有自己的保存历史 |

<a id="reading-individual-records"></a>

### 阅读单条记录

被丢弃的消息会链接到所属工单；如果没有工单，则链接到所属线程。来源页面会说明，是对话已经消失，还是来源不再监听该对话。

仍由执行器拥有处理权的人类后续消息，会被标为 **Delivered to Ticket（已投递到工单）**，没有评估记录。线程也会展示退回理由和仅供上下文使用的订阅。**Evaluate again（再次评估）** 可以把这类后续消息送入评估，也可以重试没有有效判断的触发：被来源过滤的触发、结果为 unavailable 的评估，或者既没有非 stop 决定、也没有执行器确认的已投递消息。

工单详情展示起始触发、决定及理由、范围和策略版本。派工和次要细节折叠在当前可执行操作下方。输入投递失败，或者工单在 blocked 或 failed 报告后等待人处理时，会提供 **Retry（重试）**。已结束的工单会显示结果，例如 **Pull request merged（拉取请求已合并）** 或 **Stopped by a person（已被人停止）**。

<a id="settings-groups"></a>

## 设置分组

- **Sources（来源）：** 每个来源一张卡片，包含开关、字段和已验证的账号事实。GitHub 还为新 Issue、拉取请求、评论和默认分支 CI 提供接收全部消息开关。轮询和读取限制放在 Advanced（高级）中。
- **Eval（评估）：** 先展示策略版本和规则，再展示 **On each Source（各来源）** 以及评估设置。每个来源标签页包含允许名单、成员同步、准入表和机器人名单。同步在启动时以及点击 **Sync now（立即同步）** 时执行。准入表区分接收范围、发送者是否在允许名单上，以及消息是否面向 Agent。评估设置用于选择接单模式、超时和研判模型。
- **Execution（执行）：** worker 模型、并发数、目录根路径、准备命令和禁用命令。点击可用变量，会将其插入准备命令。
- **Prompts（提示词）：** **Edit（编辑）** 用于修改团队区段、内置部分和技能。**Full prompt（完整提示词）** 展示组装后的文本，并可与已保存文本或默认值比较。**History（历史）** 列出各个版本。编辑过的内置部分会标为已修改，并可重置；模板会列出必需的占位符。

  如果升级改变了一个已被覆盖的默认部分，面板会将它标为 **Default updated（默认内容已更新）**。面板在团队文本上方比较新旧默认内容。如果没有记录旧默认内容，则改为比较团队文本和新默认内容。可以选择 **Keep my text（保留我的文本）** 或 **Use the new default（使用新默认内容）**。已移除部分的覆盖项列在标签栏下方，下次保存时会被丢弃。
- **Deployment（部署）：** 启动问题及 **Check again（再次检查）**、状态目录和机器限制。保存后的限制无需重启即可生效。DeepSeek Harness Plugin Manager 也提供同一个页面。

<a id="search"></a>

## 搜索

页面顶部搜索可以按文本、id、飞书消息 id，或已存储链接（例如 GitHub 评论 URL）查找触发。如果原始消息不包含某条飞书链接，则只有宿主在当前进程存活期间读取过它之后，该链接才可被搜索到。

搜索也能通过 `#12` 或 GitHub URL 之类的引用查找线程，并通过标题或匹配的线程查找工单。

<a id="principles"></a>

## 原则

1. **关注真正重要的内容。** 每个元素都要回答上述三个问题之一，否则就移除。只有当某个数字会改变监督者的行动时，才展示它。
2. **定义准确。** 分类必须有可验证的定义，彼此互斥，并覆盖所有情况，就像工单阶段那样。标签应说明这一行在等谁、需要什么，而不是展示内部状态码。
3. **每个概念只有一个归属。** 来源状态放在 Source；线程和触发放在 Trigger；决定放在 Eval；工单放在 Execution。设置按其配置的概念分组。Overview 汇总这些页面并链接过去，而不是成为另一个详情视图。
4. **遵循领域结构。** 面板使用领域术语。内部标识符只出现在折叠详情中；能够解析名称时，就用名称代替标识符。
5. **如实展示状态。** 计数等于其所链接列表的长度，任何状态都不能与其他事实矛盾。
6. **默认安静。** 健康状态不占额外空间。问题必须能促使人采取行动；偶发的一次网络错误还不够。需要人处理的行，应直接提供操作入口。页面说明保持一句话，更详细的内容放进 (?) 提示中。破坏性操作需要确认。
7. **控件保持一致。** 使用 DeepSeek Harness 的设计令牌和控件行为。同样的信息在面板各处应有相同外观。每个设置区块只有一个保存栏，仅在存在修改、正在保存或保存失败时可见。

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
