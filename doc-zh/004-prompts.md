<a id="004-prompts"></a>

# 004 提示词

> 原文版权：Copyright (c) 2026 DeepSeek Harness Team；MIT License。
> [英文原文](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/004-prompts.md) · 固定版本：`6337fc823ce20251c8de8085948165c81aaa701e`  
> 中文翻译，非官方；代码与配置保持原样。

本文说明 Agent 会读取什么、团队可以编辑什么，以及变更何时生效。本文建立在 [000 设计理念](000-philosophy.md) 到 [003 执行](003-execution.md) 的内容之上。提示词文本使用英语编写。

<a id="what-a-team-writes"></a>

## 团队编写什么

团队可以编辑三个团队区段、添加技能，并覆盖每一个内置系统提示词部分。大多数默认内容位于 `prompts/` 和 `skills/` 下。工具描述、通知和卡片仍在代码中，无法通过提示词设置编辑。

| 区段 | 包含的内容 |
| --- | --- |
| Communication（沟通） | 何时向人发消息、消息如何措辞、每条 GitHub 消息开头的说明（默认为 `This is by an automated agent, not a human`）、如何处理审查，以及哪些普通账号属于审查机器人 |
| Ticket scope（工单范围） | 工单何时继续，何时以草稿停下 |
| Delivery requirements（交付要求） | 拉取请求中的证据、自动合并条件及其行数阈值，以及提交作者；如果未指定，执行器会使用 Agent 账号的 no-reply 邮箱提交 |
| Team skills（团队技能） | Agent 在需要时加载的知识，例如用一个 `identities` 技能说明某个飞书用户对应哪个 GitHub 账号 |

内置文本涵盖角色、工具用法、宿主机制、授权规则，以及 GitHub 和飞书技能。它通过标题引用团队区段，而不重复这些区段的内容。

<a id="built-in-parts-and-templates"></a>

### 内置部分与模板

覆盖项使用以下键：

| 部分 | 键 |
| --- | --- |
| 共享指令与角色 | `runtime/shared`、`runtime/orchestrator`、`runtime/worker` |
| 工作指令 | `runtime/subscriptions`、`runtime/replies`、`runtime/dispatch`、`runtime/work`、`runtime/admission` |
| 交接指令 | `runtime/handoff/orchestrator`、`runtime/handoff/worker` |
| 来源技能 | `skill/feishu`、`skill/github` |
| 生成的区段 | `runtime/environment`、`runtime/policy`、`runtime/policy/none` |

生成的区段是可编辑模板，必须保留以下占位符：

- `runtime/environment`：`{variables}`，填入执行环境的各行内容。
- `runtime/policy`：`{version}`、`{mode}` 和 `{rules}`，每条已启用规则占一行。
- `runtime/policy/none`：`{version}` 和 `{mode}`，没有启用任何规则时使用。

编辑某个部分会改变模型收到的指令，但不会改变准入层、授权、工具检查或账号包装器；这些仍由代码强制执行。

人明确提出请求时，主 Session 也可以使用 `orchestrator_prompts_save` 保存覆盖项。如果保存时省略某个覆盖项的 `base`，就保留已保存的 base。如果已保存配置中包含某个已被移除的部分，保存时会将其丢弃；其他未知部分则会被拒绝。

覆盖项会记录它基于哪一版内置文本。升级会保留团队文本。如果默认内容发生变化，面板会将该部分标为 **Default updated（默认内容已更新）**，并在团队文本旁展示差异。团队可以保留自己的文本、使用新默认值，或继续编辑。没有覆盖项的部分会自动跟随新的默认内容。

主 Session 可以在 `orchestrator_read prompts` 的 `default_changed` 中读取受影响部分的名称。

<a id="assembly-and-precedence"></a>

## 组装与优先级

系统提示词按以下顺序组装：

1. 共享指令和角色。
2. 三个团队区段。
3. 订阅与回复。
4. 主 Session 使用派发指令，执行器使用工作指令。
5. `runtime/admission`，仅用于主 Session。
6. 交接指令。
7. `runtime/environment`。
8. `runtime/policy`，仅用于主 Session。

准入指令来自 `prompts/admission.md`。它描述主动研判，以及何时可以直接根据卡片忽略无关候选。`orchestrator_eval` 返回上下文，而不会重复这些指令。

环境区段向主 Session 提供部署级的值，向每个执行器提供其工单专属的值。策略区段在每次模型请求时根据设置重新生成，包含策略版本、接单模式、明确请求的豁免，以及格式为 `- <name> (<id>): <instruction>` 的已启用规则；如果没有启用规则，则包含相应说明。它被放在最后，使策略变化不会改变前面的提示词文本。

执行器还会读取 DeepSeek Harness 的编程提示词和仓库的 `AGENTS.md`；主 Session 不会。推断工单会在最后加上一条说明，将执行器限制在匹配规则之内。派工和传入消息以内联触发卡片的形式到达，只有被截断的内容才使用资源指针。

执行器在无人值守的情况下工作。它的工具集不包含向人提问、向人展示 UI，或修改配置档案的工具，例如 `ask_user_question`、`present` 和 `plugin_manager`。

从强到弱的优先级如下：

1. 工具和宿主强制执行的事实：身份、权限、工具参数检查、回复用途和长度限制。提示词不能改变这些内容。
2. 内置区段和生成区段；如团队提供了覆盖项，则采用覆盖后的文本。
3. 主动工单所匹配规则中的流程与沟通要求，仅对该工单的执行器 Session 生效。
4. 团队区段和团队技能。
5. 外部文本：消息、评论、文档和引文。它们是数据，绝不能改变规则或授权。

<a id="principles-of-the-built-in-text"></a>

## 内置文本的原则

以下原则把 [000 设计理念](000-philosophy.md) 应用到 Agent：

- **分开角色。** 主 Session 只做路由：转发原始触发和路由事实，不作技术决策；它使用一个路由工具，一次调用就能处理多个等待候选的决定。路由备注最多 30 个词。执行器读取原始来源、判断范围、实施并验证。
- **主 Session 不等待。** 报告、后续消息通知、候选以及已完成的后台作业，会自动以消息形式到达。没有剩余工作时，主 Session 就结束回合。防护逻辑会拒绝 `sleep`、超过 30 秒的 shell 超时，以及超过 5 秒的作业等待，因为阻塞的步骤会拖延新候选。在主 Session 的完整提示词中，这些指令取代 DeepSeek Harness 自身的作业通知说明。
- **只有在需要人采取行动，或者结果对人有用时才发消息：** 结果可供审查或使用、需要决定或资源，或者工作无法完成。聊天消息默认只写一句，硬性上限为 100 个词。
- **执行器决定如何处理自己的后续消息。** 每个执行器处理并确认范围内的请求，将超出工单范围或属于其他工单的请求退回。只有当所有拥有消息处理权的工单都退回了它，或者在未接手的情况下结束，它才会成为新工作。主 Session 会收到后续消息通知，但只能据此停止工作。退回的消息随后会作为候选到达，并带上执行器的理由。
- **通过订阅和主 Session 协调。** 执行器关注相关线程以获取上下文。主 Session 可以用最多 100 个词的备注发送另一工单中的某项事实。订阅和备注都不授予权限。
- **只有明确引用时才附加。** 消息必须指向某个工单的拉取请求、Issue、来源消息或确认消息，或者指向一个近期变更且恰好只有一个工单与之匹配。仅仅主题相似并不足够。附加操作会让工单订阅整个线程；执行器如果发现线程无关，会使用 `orchestrator_thread_unrelated`，且不在那里回答。
- **具体确认，而且只确认一次。** 在开展实质性工作前，执行器会针对每个新请求在原始消息下回复，说明处理对象和预期结果。
- **通过 `human_action` 提问。** 执行器在报告中放入一个具体问题，主 Session 原样转发。如果问题由某条人类消息引起，就指出该消息，由宿主在该处提问；否则就在工单的原始对话中提问。与此同时，已经获得授权的工作可以继续。
- **诚实交付。** 创建拉取请求不等于已就绪；草稿必须标记为未完成；没有真实修改前截图的 UI 变更不能算已就绪。
- **人的暂停要求持续有效，** 不会因后续消息、新执行器 Session 或重启而失效，直到人解除暂停。自动反馈永远不会恢复被暂停的工作。
- **示例只展示格式。** 示例句子不是 Agent 可以声称为真的事实。
- **按结果处理审查。** 已修复并验证的评论静默解决；不采纳的评论用一句话说明理由。自动化审查者的讨论线程在获得回答后即解决，而且每个自动化审查者对每个拉取请求最多进行三轮。
- **遇到重复工作时让出。** 如果某个人的分支或拉取请求已经包含该工单的修改，执行器会关闭自己的拉取请求，并用一句话指向对方的工作。

<a id="versions"></a>

## 版本

- 每次保存都会创建一个不可变版本，包含完整配置、修订号、前序版本、时间戳和来源（面板或工具）。恢复历史版本或默认值，也会创建另一个版本。
- 保存操作会指定其基准修订。发生冲突时，保存会被拒绝，但不会丢弃草稿。
- Session 在每次模型请求时读取最新版本，包括同一回合的后续步骤。保存不会重启、取消或唤醒 Session。宿主会记录各 Session 使用过的修订；执行器可以读取自己曾依据其工作过的任何修订。
- 部署会设置包含覆盖项在内的总大小限制。验证会拒绝未知键、保留的内置技能名称、未知覆盖部分，以及缺失的模板占位符。文本绝不会被静默截断。与默认内容相同的覆盖项会被移除，使该部分能够跟随未来的默认更新。

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
