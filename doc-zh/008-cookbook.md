<a id="008-cookbook"></a>

# 008 扩展实践

> 原文版权：Copyright (c) 2026 DeepSeek Harness Team；MIT License。
> 来源：[008-cookbook.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/008-cookbook.md) · 固定版本：`6337fc823ce20251c8de8085948165c81aaa701e`。中文翻译，非官方；代码与配置保持原样。

本实践指南介绍如何扩展 dsh-superagent。文中使用 [001 领域模型](001-domain.md)和 [002 授权](002-authorization.md)中的术语，以及 [007 架构](007-architecture.md)中的组件。

<a id="add-a-source"></a>

## 添加一种来源

本节实现一种名为 `webhook` 的来源类型。它通过虚构的 CLI `hookq` 读取一个假想的消息队列，并将消息转换为触发。其他接口均属于 dsh-superagent。

实现承担四项职责：转换事件，定义接收和授权规则，提供回复与上下文，以及维护轮询循环。下面的编号步骤依次构建这些部分。

示例假定 CLI 遵循以下约定：

- `hookq pull --after <n> --json` 按从旧到新的顺序输出位置 `n` 之后的消息：`{"me": "<the agent's id on the queue>", "items": […], "next": <position>}`。其中 `me` 是智能体在队列中的 ID，`next` 是位置。每个条目包含 `id`、`conversation`、`thread`（发起该线程的消息 ID；发起消息自身没有这个字段）、含 `id` 和 `bot` 的 `sender`，以及 `mentions_me`、`text`、`at`、`url`。
- `hookq reply --conversation <id> --to <message id> --key <key> --text <text>` 在指定消息下回复，每个 key 只执行一次。如果对话已被删除，命令会失败，并向标准错误输出 `{"error": {"code": 410, "message": "…"}}`。
- `hookq thread --conversation <id> --root <message id> --json` 按从旧到新的顺序输出线程的一页内容：`{"items": […], "next": "<cursor>"}`。读取后续页时添加 `--after <cursor>`；最后一页不包含 `next`。

参照 `src/sources/github.ts`，将来源类型及其插件入口放在 `src/sources/webhook.ts` 中。如果来源实现变得复杂，可以像飞书来源一样，将转换、渠道和轮询代码拆分为独立模块。

<a id="1-map-the-events-onto-the-domain"></a>

### 1. 将事件映射到领域模型

编写代码之前，先选择稳定的映射。触发 ID、线程 ID 和渠道地址都会持久化，因此日后修改它们可能让已有记录失去关联。

| 概念 | 在 webhook 队列中的对应物 |
| --- | --- |
| 触发（Trigger） | 一条消息 |
| 线程（Thread） | 同一对话中的一条消息及其回复：`<conversation>:<id of the first message>` |
| 行为者（Actor） | 发送者在队列中的 ID，与允许名单使用的 ID 相同 |
| 渠道地址（Channel address） | 对话，以及回复应发布在哪条消息之下 |
| 接收范围（Receiving scope） | 一个对话，在准入表中有自己的一行 |

<a id="2-configuration-schema-fields-facts-and-health"></a>

### 2. 配置：schema、fields、facts 和 health

模块导入后续步骤使用的内容：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { setTimeout as delay } from 'node:timers/promises'
import { z } from 'zod'
import { key, threadKey, type OutboundMessage, type Trigger } from '../core/index.ts'
import { CliError } from '../infra/commands.ts'
import { KV } from '../kv.ts'
import { health, sourceContext, type SourceContext } from './host.ts'
import type { SourceKind } from './kind.ts'
```

```ts
const ConfigSchema = z.object({
  conversations: z.array(z.object({
    id: z.string().regex(/^[a-z0-9-]+$/),
    listen: z.enum(['mentions', 'all']),
  }).strict()).default([]),
  pollSeconds: z.number().int().min(10).max(3600).default(30),
}).strict()
export type WebhookConfig = z.infer<typeof ConfigSchema>

export const webhookKind: SourceKind<WebhookConfig> = {
  kind: 'webhook',
  title: { zh: 'Webhook 队列', en: 'Webhook queue' },
  description: { zh: '接收 webhook 队列里下列对话的消息。', en: 'Receives the messages of these conversations from the webhook queue.' },
  schema: ConfigSchema,
  fields: [
    { key: 'conversations', type: 'rows', label: { zh: '接收的对话', en: 'Conversations' }, columns: [
      { key: 'id', type: 'text', label: { zh: '对话', en: 'Conversation' }, pattern: '[a-z0-9-]+' },
      { key: 'listen', type: 'select', label: { zh: '接收', en: 'Receive' }, options: [
        { value: 'mentions', label: { zh: '仅 @', en: '@ only' } }, { value: 'all', label: { zh: '全部消息', en: 'All' } }] },
    ] },
    { key: 'pollSeconds', type: 'number', advanced: true, min: 10, max: 3600,
      label: { zh: '轮询间隔 / 秒', en: 'Polling interval / seconds' } },
  ],
  health: ['webhook'],
  // find, scope, channel, threadContext, policy and environment follow in the next steps.
}
```

- `kind` 是来源的稳定 ID。它会出现在触发的 `source`、线程键的前缀、`source:webhook` 设置块和来源策略中。
- `schema` 验证已保存的配置，并提供默认值。在运维者保存实例前，注册表会根据这些默认值创建一个已启用的实例。默认配置应当不接收任何内容；本示例默认不配置任何对话。
- `fields` 生成来源设置表单，无须自定义面板代码。`rows` 字段编辑以第一列为键的条目；`advanced` 隐藏不常用设置。如果来源能将 ID 解析为显示名称，请实现 `refreshLabels`。将名称保存在 `KV.sources.labels(kind)` 下，带有 `choose` 和 `refresh` 的列即可提供这些名称供选择。
- 可选的 `facts(deployment)` 展示配置检查推导出的值，例如 CLI 对应的账号。目前配置检查只检查 `SETUP_FIELDS` 中列出的 GitHub 和飞书字段，因此本示例不提供 facts。
- `health` 声明来源的健康状态条目，初始值为 `starting`。注册过程会占用每个名称，并拒绝与 `runtime`、`reply`、`worker` 或 effect 类型等已有条目重名。应使用来源类型的名称，并将 `sync:<id>` 保留给策略同步。`error` 会使状态脚本以 2 退出，并使每小时检查通知运维者，因此只有在可能需要人工介入时才报告它。

<a id="3-translate-a-message-into-a-trigger"></a>

### 3. 将消息转换为触发

转换之前先验证 CLI 响应：

```ts
/** One message as `hookq pull` and `hookq thread` print it. */
const MessageSchema = z.looseObject({
  id: z.string().min(1), conversation: z.string().min(1), thread: z.string().optional(),
  sender: z.looseObject({ id: z.string(), bot: z.boolean().default(false) }), mentions_me: z.boolean().default(false),
  text: z.string(), at: z.string(), url: z.string().optional(),
})
type QueueMessage = z.infer<typeof MessageSchema>
/** The answer of `hookq pull`, whose items are read one by one, and of `hookq thread`. */
const PageSchema = z.looseObject({ me: z.string(), items: z.array(z.unknown()), next: z.number().int() })
const ThreadPageSchema = z.looseObject({ items: z.array(MessageSchema), next: z.string().optional() })
```

```ts
/** The Trigger of one queue message; the agent's own messages produce none. */
export function webhookTrigger(message: QueueMessage, me: string): Trigger | undefined {
  if (message.sender.id === me) return undefined
  const thread = `${message.conversation}:${message.thread ?? message.id}`
  return {
    id: key('webhook', message.id),
    source: 'webhook',
    threads: [{ source: 'webhook', id: thread }],
    actor: { source: 'webhook', id: message.sender.id },
    human: !message.sender.bot,
    ...(message.mentions_me ? { addressed: 'mention' as const } : {}),
    at: message.at,
    text: message.text,
    channel: { source: 'webhook', thread: `webhook:${thread}`,
      address: { conversation: message.conversation, message: message.id, ...(message.url ? { url: message.url } : {}) } },
    ...(message.url ? { url: message.url } : {}),
    meta: { conversation: message.conversation, messageId: message.id, root: message.thread ?? message.id },
  }
}
```

- **稳定 ID。** 通过 `key`，从队列 ID 派生触发 ID。对于已经存储的 ID，`Engine.receive` 返回 `duplicate`，重复接单也只会创建一张工单。因此，重复读取消息是安全的。如果要把编辑视为新事件，就在 ID 中加入编辑版本；GitHub 对评论使用 `updated_at`，就是这种做法。
- **线程。** 第一个线程是主线程，拥有对应的评估。其他线程通过关联提供上下文。来源自行选择 ID；其完整键为 `webhook:<id>`。
- **发送者。** 在 `actor` 中使用来源原生的 ID。对于机器人和 CI，设置 `human: false`。在名单中的机器人也只能成为推断候选。像 CI 那样的空 actor ID，会使用准入表中“不在名单、未指向智能体”的发送者单元格。只有消息确实指向智能体时，才将 `addressed` 设为 `mention` 或 `direct`；这样，允许名单中的人的消息才会获得明确强度。绝不要为智能体自己的消息生成触发。
- **回复目标与元数据。** `channel` 标识回复位置；`url` 链接到原始消息。`channel.thread` 将消息分组，以便按顺序投递：同一工单的后续消息会等待前面的消息，直到较早的消息已失败三次。其他工单不必等待。将来源特有字段保存在 `meta` 中，例如接收消息的对话和线程根消息。`meta` 和 `text` 都是不可信数据，不能作为授权依据。

智能体可以通过 `orchestrator_read trigger/<id>` 和 `thread/<key>` 读取新类型。它们会暴露消息文本、去向和相邻触发。新增一种来源类型，不会自动获得内置技能、原始服务记录资源或拉取请求绑定能力。

文本搜索无须额外代码即可工作。当人们使用消息 ID、URL 或 `#12` 等来源特有引用时，请实现 `find(query)`。返回这些引用对应的线程键和搜索词：

```ts
  find: query => ({ threads: [], terms: [...new Set(query.match(/\bm[0-9]+\b/g) ?? [])] }),
```

这里，查询中的队列消息 ID，例如 `m42`，会被用于搜索触发。

<a id="4-what-the-source-receives-scope"></a>

### 4. 来源接收什么：scope

```ts
  scope({ meta }, config) {
    const conversation = config.conversations.find(item => item.id === meta.conversation)
    if (!conversation) return { listened: false, reason: 'conversation_not_listed' }
    return { listened: true, observed: conversation.listen === 'all', row: conversation.id }
  },
```

引擎通过 `SourceRegistry` 为每个传入的触发调用 `scope`。不在范围内的消息会被保存为已过滤，并记录原因，但不会创建线程。应将消息传给 `receive`，不要在轮询循环中静默丢弃它们，这样面板才能解释过滤原因。

`observed: true` 表示来源会接收该范围内未指向智能体的消息，从而允许准入表对这些消息作决定。`row` 标识相应的准入行。

重新验证授权时，会使用授权中保存的元数据调用 `scope`。应让它只依赖 `meta` 和配置。恢复执行要求 `listened`，而不要求 `observed`。如果没有 `scope`，该类型会接收自己产生的每个触发，但不会将任何触发视为主动观察到的内容。

<a id="5-replies-the-channel"></a>

### 5. 回复：渠道

```ts
  channel: {
    allowed: (address, config) => config.conversations.some(item => item.id === address.conversation),
    send: sendWebhook,
    link: address => ({ url: address.url ?? '',
      label: { zh: `Webhook 对话 ${address.conversation}`, en: `Webhook conversation ${address.conversation}` } }),
  },
```

```ts
export async function sendWebhook(host: Pick<SourceContext, 'commands'>, message: OutboundMessage): Promise<'sent' | 'gone'> {
  const { conversation, message: to } = message.channel.address
  if (!conversation || !to) throw new Error('Missing webhook reply target')
  try {
    await host.commands.run('hookq', ['reply', '--conversation', conversation, '--to', to, '--key', message.id,
      '--text', message.text])
    return 'sent'
  } catch (error) {
    if (error instanceof CliError && error.code === 410) return 'gone'
    throw error
  }
}
```

- **幂等发送。** 发件箱（Outbox）调用 `SourceRegistry.send`，后者调用 `channel.send`。来源插件未加载时，消息会等待。远端服务可能已经收到消息，本次发送却仍然报错，因此重试不能产生重复消息。请用 `message.id` 作为幂等键：`hookq` 支持 `--key`；GitHub 保存一个隐藏的评论标记；飞书则接受幂等键。
- **永久失败与可重试失败。** 成功时返回 `sent`；只有永久无法投递时才返回 `gone`，例如本例中的 410。gone 消息会保留为已丢弃，永远不再排队；它会出现在 `GET sources` 和来源页面中。其他失败应抛出异常，让发件箱在 5 秒后重试，重试间隔逐次翻倍，最高为 300 秒。`CliError.code` 是 CLI 输出中报告的错误码。
- **允许的目标。** `allowed(address, config)` 只检查保存的地址和当前配置。将检查所需的全部事实放入地址中；例如，飞书会记录聊天是否为私聊。不再监听的目标会使消息被丢弃；来源被禁用时，消息则继续等待。若没有 `allowed`，来源处于启用状态时，所有地址都被允许。
- **链接。** `link(address, links)` 返回面板通过 `SourceRegistry.link` 展示的 URL 和标签。

<a id="6-thread-context-for-assessments"></a>

### 6. 为研判提供线程上下文

```ts
  threadContext: {
    messageId: trigger => String(trigger.meta.messageId ?? ''),
    // A queue thread is flat: its root, and the message a reply answers, are on its first page, as on GitHub.
    read: async (commands, trigger, _part, cursor) => {
      const page = ThreadPageSchema.parse(JSON.parse(await commands.run('hookq', ['thread', '--conversation',
        String(trigger.meta.conversation), '--root', String(trigger.meta.root), '--json', ...(cursor ? ['--after', cursor] : [])])))
      const messages = page.items.map(item => ({ id: item.id, from: item.sender.id, at: item.at, text: item.text,
        ...(item.sender.bot ? { bot: true as const } : {}), ...(item.url ? { url: item.url } : {}) }))
      return { messages, ...(page.next ? { next: page.next } : {}), body: { messages } }
    },
  },
```

主动研判需要触发周围的对话。`orchestrator_eval` 读取初始窗口；`orchestrator_eval_read` 扩展读取后续页、根消息或父消息。实现可选的 `threadContext` 接口即可提供这些读取能力：

- `read(commands, trigger, part, cursor, config)` 返回 `ContextPart`。其中的 `messages` 使用 `ContextMessage`：`{id, from, at, text}`，以及适用时的 `bot`、`deleted`、`reply_to`、`url`、`path` 和 `line` 字段。最后一页省略 `next`。`body` 是展示给研判 Session 的内容，这里与 messages 相同。可选的 `status` 可以描述特殊结果，例如飞书的 `chat_neighbours`。`part` 为 `thread`、`root` 或 `parent`。
- `messageId(trigger)` 在这些消息中定位触发本身所对应的消息。
- `images(text)` 和 `image(…)` 适用于消息携带图片的来源类型，例如飞书；本示例省略它们。

没有 `threadContext` 的类型不会为研判提供线程上下文，但触发本身的文本仍然可读。

如果消息会回复较早的消息，请添加 `repliesTo(trigger)`。它返回对应的触发 ID，使后续通知和执行器输入能够在被回复消息已经保存时展示它。例如，假设 `hookq` 提供 `reply_to`，并且转换过程将其保存为 `meta.replyTo`：

```ts
  repliesTo: trigger => typeof trigger.meta.replyTo === 'string' ? key('webhook', trigger.meta.replyTo) : undefined,
```

<a id="7-who-may-ask-the-sources-part-of-the-policy"></a>

### 7. 谁可以请求工作：来源的策略部分

```ts
  policy: {
    schema: z.object({ allowed: z.array(z.object({ id: z.string().min(1), name: z.string().trim().max(100).optional() }).strict())
      .default([]) }).strict(),
    fields: [{ key: 'allowed', type: 'rows', label: { zh: '能发起工作的发送者', en: 'Senders who may originate work' }, columns: [
      { key: 'id', type: 'text', label: { zh: '发送者', en: 'Sender' } },
      { key: 'name', type: 'text', label: { zh: '名字', en: 'Name' }, optional: true }] }],
    explicit: { zh: '名单里的人在消息里 @ 智能体。', en: 'Someone on the list mentions the agent in a message.' },
    rows: stored => ConfigSchema.parse(stored).conversations.map(conversation => {
      const all = conversation.listen === 'all'
      return { id: conversation.id, label: conversation.id, note: all ? { zh: '全部消息', en: 'All' } : { zh: '仅 @', en: '@ only' },
        cells: { listedUnaddressed: all, unlistedAddressed: true, unlistedUnaddressed: all } }
    }),
  },
```

- `schema` 验证来源策略，包括 `allowed` 中的发送者 ID。ID 比较不区分大小写。准入表单独验证，并在来源类型之间共用。该策略部分保存在 `policy.sources.webhook`，并计入策略版本。
- `fields` 生成 **Eval → On each Source** 下的表单，以 `allowed` 开头。`explicit` 说明如何向智能体发出请求。可选的 `unnamed: {person, bot}` 为飞书 `ou_…` 这类无法直接看出含义的 ID 提供显示标签；本例的队列 ID 是可读名称，因此省略它。
- `rows(config)` 列出准入范围及其可用单元格。每一行的 ID 必须与 `scope` 返回的 `row` 匹配。未指向智能体的消息单元格，只能在来源会观察这些消息的范围内使用。没有 `rows` 的类型使用默认单元格。
- 可选的 `bots` 列出有资格成为推断候选的机器人，条目形状与 `allowed` 相同。没有该字段时，不会准入任何机器人。
- 可选的 `sync: {label, ready(part), run(host, part, signal)}` 在启动时和点击 **Sync now** 时刷新成员名单。飞书使用聊天成员。失败会显示在 `sync:<id>` 下，并在一分钟后重试；间隔逐次翻倍，最高为一小时，直到成功。

<a id="8-variables-for-the-executor"></a>

### 8. 执行器使用的变量

```ts
  environment: {
    variables: [{ key: 'DSH_SUPERAGENT_WEBHOOK_CONVERSATION', description: {
      zh: '接下工单的 webhook 对话；工单不是从这个来源来的时为空',
      en: 'The webhook conversation that started the ticket; empty when the ticket did not start on this Source' } }],
    values: ({ trigger }) => ({
      DSH_SUPERAGENT_WEBHOOK_CONVERSATION: trigger?.source === 'webhook' ? String(trigger.meta.conversation ?? '') : '' }),
  },
```

即使来源被禁用，Execution 设置也会在准备命令旁显示已声明的 `variables`。来源启用时，`values` 根据部署、配置和发起工单的触发，提供该工单特有的值。

准备命令读取环境变量；执行器从自己的指令中读取这些变量的值。请使用 `DSH_SUPERAGENT_<KIND>_…` 前缀，为来自其他来源的工单返回空值，且绝不能包含凭据。

<a id="9-the-plugin-entry-and-its-loop"></a>

### 9. 插件入口与轮询循环

```ts
export const name = 'dsh-superagent-webhook'
export const inject = ['orchestrator']
export function apply(ctx: Context): void {
  ctx.effect(() => ctx.orchestrator.sources.register(webhookKind), 'webhook: source kind')
  ctx.effect(() => {
    const source = sourceContext(ctx.orchestrator, webhookKind)
    const controller = new AbortController()
    // DSH ends its process on an unhandled rejection, so a failure that escapes the loop becomes the Source's health.
    const running = watch(source, controller.signal)
      .catch((error: unknown) => { source.report('webhook', { state: 'error', detail: String(error) }) })
    return async () => { controller.abort(); await running }
  })
}
```

插件注入 `orchestrator`。第一个 effect 注册来源类型，并在销毁时取消注册。第二个 effect 使用 `SourceContext` 运行轮询循环。销毁时会中止循环，并等待其完成；应用停止或重启时也一样。

```ts
async function watch(ctx: SourceContext, signal: AbortSignal): Promise<void> {
  await ctx.ready
  // Setup restarts dsh-superagent once it resolves the accounts; until then the Source reads nothing.
  if (!ctx.configured()) { ctx.report('webhook', health('paused', 'setupPending')); return }
  let failures = 0
  while (!signal.aborted) {
    const config = ctx.config<WebhookConfig>('webhook')
    if (!config) ctx.report('webhook', health('paused', 'sourceOff'))
    else {
      const at = new Date().toISOString()
      try {
        const { read, unreadable } = await pollQueue(ctx, signal)
        failures = 0
        ctx.report('webhook', { state: 'ready', detail: `${read} messages, ${unreadable} unreadable`, at })
      } catch (error) {
        failures += 1
        // One network failure is retried on the next round; three in a row need a person.
        if (!signal.aborted && failures >= 3) {
          ctx.report('webhook', health('error', 'pollFailures', { count: String(failures), error: String(error) }))
        }
      }
    }
    await delay((config?.pollSeconds ?? 30) * 1000, undefined, { signal }).catch(() => {})
  }
}

/** Read the queue after the cursor, take in each message, then move the cursor past them. */
export async function pollQueue(ctx: SourceContext, signal: AbortSignal): Promise<{ read: number; unreadable: number }> {
  const after = ctx.state.get(KV.webhook.cursor) ?? 0
  const output = await ctx.commands.withSignal(signal).run('hookq', ['pull', '--after', String(after), '--json'])
  const page = PageSchema.parse(JSON.parse(output))
  let unreadable = 0
  for (const item of page.items) {
    if (signal.aborted) throw new Error('The webhook Source stopped during the read')
    const message = MessageSchema.safeParse(item)
    if (!message.success) { unreadable++; continue }
    const trigger = webhookTrigger(message.data, page.me)
    if (!trigger) continue
    await ctx.receive(trigger)
    const conversation = { label: { zh: '对话', en: 'Conversation' }, value: message.data.conversation }
    for (const ref of trigger.threads) ctx.threads.describe(threadKey(ref), { facts: { conversation } })
  }
  ctx.state.set(KV.webhook.cursor, page.next)
  return { read: page.items.length, unreadable }
}
```

- **配置检查与开关。** 等待 `ctx.ready`；`ctx.configured()` 为 false 时，不读取任何内容。每轮都读取 `ctx.config(kind)`：来源关闭时，它为 undefined。
- **游标。** 使用 `KV`（`src/kv.ts`）中的类型化条目，在 `ctx.state` 中保存读取位置，例如 `webhook: {cursor: record<number>('webhook-cursor')}`。`tests/store.spec.ts` 要求键的字面量只能出现在这个注册表中。外部包可以使用导出的 `record` 和 `family` 定义键。将值设为 `undefined` 会删除记录；`ctx.atomic` 将多次写入归入同一个事务。

  只有覆盖到的全部触发都已存储后，才能推进游标。如果一轮读取被中断，游标应保持不变；稳定 ID 使重放安全。对于时间戳游标，`src/sources/poll.ts` 中的 `cursorAt(started)` 会减去五秒，为远端延迟写入留出余量。
- **线程元数据。** 接收触发后，调用 `ctx.threads.describe(key, {label, url, facts})` 填充其显示信息。`ctx.threads.patch(key, meta)` 委托给 `threads.patchMeta`，设置由来源管理的键，或者在值为 `undefined` 时删除这些键。两者都会忽略不存在的线程，包括已过滤触发对应的线程。
- **逐条处理错误。** 统计格式错误的条目并继续，不要让整个批次失败。`src/infra/commands.ts` 中的 `validItems(item, list, tally)` 支持这种模式。对于独立轮询的线程，使用 `src/sources/poll.ts` 中的 `pollThreads`；它把每次失败记录为 `pollError`。其 `polled` 谓词会保留有活动订阅的线程，并在“最近一次触发”和“工单结束”两者中较晚的时间点之后，继续保留两周的轮询窗口。
- **健康状态。** 通过 `ctx.report` 仅写入已经声明的条目。`health(state, code, values)` 提供 `setupPending`、`sourceOff`、`pollFailures` 等固定消息。报告 `error` 之前，应先统计连续失败次数。
- **中止。** 将 signal 传给 `ctx.commands.withSignal(signal)` 和 `delay`，并在各条目之间检查它，使停止或重启能立即结束循环。

本示例不发送由来源发起的通知。有此需要的来源可以调用 `ctx.notify`；发件箱会像其他回复一样投递这些通知。

本示例也不把线程与工单完成状态绑定。只有当来源的线程生命周期决定工作是否完成时，才使用 `ctx.tickets.bindThread` 和 `threadClosed`，例如 GitHub 拉取请求的情况。工单何时结束，仍由引擎决定。

<a id="10-register-the-plugin"></a>

### 10. 注册插件

在 `tsdown.config.ts` 中将模块作为独立入口构建：

```ts
entry: { index: 'src/index.ts', 'github-watch': 'src/sources/github.ts', 'feishu-watch': 'src/sources/feishu.ts',
  'webhook-watch': 'src/sources/webhook.ts' },
```

然后在 `package.json` 的 `exports` 中加入 `"./webhook-watch": "./lib/webhook-watch.js"`，导出构建后的文件，并将插件加入插件包 `cordis.patch.yml` 的 `insert` 列表中；像飞书一样，默认禁用它：

```yaml
    - id: dsh-superagent-webhook
      name: dsh-superagent/webhook-watch
      disabled: true
```

在配置档案的 `cordis.patch.yml` 中，通过 `id: dsh-superagent-webhook` 和 `disabled: false` 启用插件。构建并重启。新来源会出现在 Sources 设置和 **Eval → On each Source** 下。

将它的模块和测试规格文件加入 [007 架构](007-architecture.md)中的模块表和测试表。`tests/docs.spec.ts` 会检查两个索引是否完整。

<a id="11-tests"></a>

### 11. 测试

参照 `tests/sources.spec.ts` 的模式创建 `tests/webhook.spec.ts`：在临时目录中使用真实的 Store，创建带 `HealthBoard` 的 `SourceRegistry`、使用替代端口实现的 Engine，以及由 `sourceContextOf` 构造的 `SourceContext`。

通过监视并替换 `Commands.run` 来模拟 CLI 调用。还要监视并替换 `withSignal`，因为它通常会返回一个新的 `Commands` 实例。

```ts
function host(overrides: Partial<SourceContextDeps> = {}) {
  const store = new Store(join(mkdtempSync(join(tmpdir(), 'webhook-')), 'db.sqlite'))
  store.saveSettings(SettingsSchema.parse({
    sources: { webhook: { enabled: true, config: { conversations: [{ id: 'ops', listen: 'mentions' }] } } },
    policy: { sources: { webhook: { allowed: [{ id: 'alice' }] } } },
  }))
  // The registry claims the kind's health entries on the board, where the test reads them.
  const board = new HealthBoard()
  const sources = new SourceRegistry(() => store.settings(), board)
  sources.register(webhookKind)
  const engine = new Engine({ repo: store, scope: sources, policy: () => settingsPolicy(store.settings(), sources),
    assessor: { request: vi.fn() }, executor: { start: vi.fn(), deliver: vi.fn(), stop: vi.fn() },
    outbox: { send: vi.fn(), status: () => undefined, withdraw: vi.fn() } })
  const commands = new Commands(() => testDeployment(), {}, new AbortController().signal)
  const receive = vi.fn((trigger: Trigger) => { const result = engine.receive(trigger); engine.evaluateDue(); return result })
  // The Source's own notices go to the outbox, which keeps their receipts and sends nothing here.
  const deps: SourceContextDeps = { store, engine, outbox: { send: message => { store.receipts.add(message) } }, commands, sources,
    deployment: testDeployment(), ready: Promise.resolve(), configured: () => true, receive, ...overrides }
  return { ...deps, store, engine, commands, receive, health: board.all, ctx: sourceContextOf(deps, webhookKind) }
}
/** The queue as `hookq` answers it. */
function queue(h: ReturnType<typeof host>, answer: () => unknown) {
  vi.spyOn(h.commands, 'withSignal').mockReturnValue(h.commands)
  return vi.spyOn(h.commands, 'run').mockImplementation(async () => JSON.stringify(answer()))
}

it('takes in what the queue holds after the cursor, once, and moves the cursor after it', async () => {
  const h = host()
  queue(h, () => ({ me: 'agent', items: [message('m1'), { id: 'broken' }, message('m2', { conversation: 'elsewhere' })], next: 3 }))
  expect(await pollQueue(h.ctx, new AbortController().signal)).toEqual({ read: 3, unreadable: 1 })
  expect(await h.receive.mock.results[0]!.value).toBe('stored')
  expect(await h.receive.mock.results[1]!.value).toBe('filtered')
  expect(h.store.get(KV.webhook.cursor)).toBe(3)
  // An allowed sender who mentions the agent makes an explicit request.
  expect(h.engine.candidate(key('webhook', 'm1'))?.strength).toBe('explicit')
  await pollQueue(h.ctx, new AbortController().signal)
  expect(await h.receive.mock.results[2]!.value).toBe('duplicate')
  h.store.close()
})
```

`message(id, fields)` 构造一个队列条目。测试至少应覆盖：

- 转换：稳定 ID、线程、行为者、`human` 和 `addressed`，以及不为智能体自己的消息生成触发。
- 完整的一轮读取：触发只存储一次，范围外的触发被 `filtered`，游标在它们之后推进，允许名单中的发送者提及智能体时被评定为 `explicit`。
- 一轮读取中途被截断时，游标保持不变。
- `send`：成功返回 `sent`；只有 410 返回 `gone`；其他失败均抛出异常。CLI 错误用 `commandError` 构造。
- 线程上下文：一页的消息以及下一页的游标，类似 `tests/registry.spec.ts` 通过 `SourceRegistry.threadContext` 读取来源类型线程的方式。
- 插件：`apply` 注册来源类型，在等待配置完成时报告 `setupPending`；其清理函数取消注册；从循环中逸出的失败会转为来源健康状态，而不是未处理的 Promise 拒绝。
- 来源的策略部分及其准入行，类似 `tests/registry.spec.ts` 和 `tests/policy.spec.ts` 对飞书来源的检查。

<a id="checklist"></a>

### 检查清单

1. 映射触发、线程、行为者、渠道地址和接收范围。
2. 编写来源类型：`schema`、`fields`、`health`，以及配置检查会推导出值时的 `facts`。
3. 将事件转换为具有稳定 ID 的触发，并丢弃智能体自身的事件。
4. 在 `scope` 中仅依据 `meta`，决定来源接收什么。
5. 为它提供渠道：`send` 对每条消息只发布一次，只有对话永远无法再接收该消息时才返回 `gone`。
6. 如果来源能够读取线程，就为研判提供线程上下文。
7. 为它提供包含 `allowed` 和准入行的策略部分。
8. 注册其执行环境变量。
9. 编写插件入口和循环：等待配置完成，遵循开关，在 `receive` 之后推进游标，按条目处理错误，报告健康状态，并在中止时停止；在 `KV` 中声明游标名称。
10. 添加构建入口、导出项和 `cordis.patch.yml` 插入项，并在 007 的表格中列出模块及其测试规格文件。
11. 按照 `tests/sources.spec.ts` 测试其他来源的方式测试它。

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
