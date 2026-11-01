<a id="000-philosophy"></a>

# 000 设计理念

> 原文版权：Copyright (c) 2026 DeepSeek Harness Team；MIT License。
> [英文原文](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/000-philosophy.md) · 固定版本：`6337fc823ce20251c8de8085948165c81aaa701e`  
> 中文翻译，非官方；代码与配置保持原样。

dsh-superagent 既是一个框架，也是一种与 Agent 协作的方式。它的核心原则是：

**尽可能少地占用人的注意力，尽可能多地创造价值。**

项目的每个部分，都要用它占用了多少注意力来衡量。一条消息、一次通知、一个问题、一项设置、面板上的一个元素，以及文档中的一个段落，都会消耗某个人的注意力。每一项带来的价值都必须超过这份消耗，否则就应当去掉。

<a id="what-follows-for-the-agent"></a>

## 这对 Agent 意味着什么

- **只有在收到请求，或者书面规则允许时，才采取行动。** 请求只有在直接面向 Agent，且来自团队允许的人时才有效：这个人要么在允许名单中，要么身处团队已开放请求权限的对话中。没有人要求的工作，必须有团队制定的规则作为依据，而且最初只能以记录模式运行：仅记录 Agent 本来会做什么。
- **只有在需要人采取行动，或者结果对人有用时，才发言。** 它对请求确认一次，并说明将做什么。执行过程中保持安静；当结果已准备好、需要作出决定，或者无法完成任务时，再次发言。消息要简短，并发送到提出请求的原对话中。
- **每次只问一个具体问题，** 并使用结构化形式，让问题真正送达需要回答的人，而不是埋在一篇长报告中。
- **根据影响决定停在哪里，而不是根据已经花了多少力气。** 局部修改应推进到可供审查的结果；需要设计决策的修改，则应带着一个明确的问题停下来。
- **如实报告。** 草稿要标明尚未完成，失败就报告失败，绝不能为了避免提问而绕过限制。

<a id="what-follows-for-the-design"></a>

## 这对设计意味着什么

- **概念要少。** 设计只包含自身的领域概念，例如来源（Source），以及这些概念的具体实现，例如 GitHub 和飞书。业务需求应成为某个现有概念的属性，或成为 Agent 遵循的提示词；设计不为它额外增加概念。
- **设置要少。** 如果一个值适用于几乎所有团队，就用默认值取代设置项。团队用自然语言描述的行为，应该写进提示词，而不是变成设置。
- **面板只展示需要人处理的内容。** 优先列出等待人处理的事项。健康状态不占用展示空间。
- **文档按顺序阅读。** 每篇编号文档只依赖前面的文档，读者不会需要用到尚未介绍的概念。

<a id="responsibility"></a>

## 责任

Agent 会通过团队每天都能看到的账号发表评论、创建拉取请求，以及向人发送消息。它发送的每一条消息都会占用同事的注意力，而你要对此负责。Agent 能做的事情越多，运营它的人就越有责任谨慎对待团队：

- **从小范围开始。** 先使用一份简短的允许名单，主动发起的工作也先仅作记录。确认 Agent 本来会采取的行动合适后，再扩大这两方面的范围。
- **调优的目标是减少干扰。** 如果 Agent 太吵，就把提示词改得更简短、更严格，或者暂停它。不要再发更多消息来解释这些干扰。
- **不要用它堆数量。** 批量创建拉取请求、群发消息，以及制造无意义的忙碌，给团队带来的负担会超过价值。
- **对它说的话负责。** 阅读 Agent 以你的名义发布的内容；如果有误，就通过提示词纠正它。

目标是在恰当的时候提供恰当的帮助，其余时候不打扰。

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
