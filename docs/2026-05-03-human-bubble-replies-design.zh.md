# 真人多气泡回复 — 阶段一设计

日期：2026-05-03  
状态：待 review 草案  
范围：只做设计；本仓库暂不实现代码。

## 目标

让 Telegram 私聊里的日常对话更像真人聊天：允许助手把一次自然回复拆成两到三条独立气泡，并在气泡之间加入短暂停顿。

目标效果：

```text
早~
[等 1 秒]
睡到快中午，真有你的杂鱼
[等 300 毫秒]
话说胸口还痛吗？
```

这和“长回复分块”不是一回事。目标不是按长度切文本，而是在真人本来就会连发几条短消息的场景里，保留聊天节奏。

## 非目标

- 不把“降低 block streaming 阈值”当成主方案。
- 不让每条回复都变成多气泡。
- 不拆任务回复、代码回复、结构化解释、错误信息、工具摘要或隐私敏感内容。
- 除非做一次性 POC，否则不绕过 OpenClaw 正常发送链路。
- 不依赖模型日常主动多次调用 `message(send)`。

## OpenClaw 现有机制

OpenClaw 现在已有三块相关能力：

1. **块回复（block replies）**
   - 助手输出可以在最终回复前，以普通频道消息的形式提前发送。
   - 这是最接近“多气泡”的现有机制。

2. **类人延迟（humanDelay）**
   - `agents.defaults.humanDelay` 可以在 block replies 之间加入随机暂停。
   - 目前只作用于 block replies，不作用于最终回复或工具摘要。

3. **最终回复去重**
   - 如果 block reply 已经发过内容，最终回复阶段会尽量避免再发一遍。
   - bubble marker 的设计必须保住这个行为。

这些机制和目标形态匹配，但现在的分块依据是长度、段落、句子，不认识“模型主动声明的气泡边界”。

## 候选方案

### 方案 A：降低 block streaming 阈值

把 block streaming 配得更激进，让它更频繁地发小块。

优点：
- 不用改代码。
- 复用已有 block replies + humanDelay。

缺点：
- 按长度切，不按聊天意图切。
- 会把技术回复/长说明切得很碎。
- 不支持每个气泡指定延迟。
- 很容易变烦。

结论：不采用。

### 方案 B：让 skill 指导模型多次调用 `message(send)`

助手每个气泡调用一次 message tool，最后返回 `NO_REPLY`。

优点：
- 现在就能用。
- 每个气泡都是正常 Telegram 消息。
- 不需要 core patch。

缺点：
- 不适合作为默认行为，模型可能忘记 `NO_REPLY` 导致重复回复。
- 普通闲聊也要工具调用，开销和复杂度变高。
- 容易和贴纸、TTS、媒体链路互相打架。
- 很难稳定限制在日常聊天场景。

结论：可以作为手动逃生路线，不适合作为默认架构。

### 方案 C：`message_sending` 插件拦截 marker 并自己发 Telegram 消息

插件检测类似 `⟦bubble:1000⟧` 的 marker，自己调用 Telegram Bot API 发多条消息，然后取消原始单条消息。

优点：
- 不改 OpenClaw core 也能做。
- 适合快速验证手感。

缺点：
- 绕过 OpenClaw 正常 delivery 链路。
- `message_sent` hook、重试、reply threading、link preview、媒体处理、TTS、delivery diagnostics 都可能不一致。
- 如果插件中途失败，可能漏发、重复发，甚至把原始 marker 暴露给用户。

结论：只适合临时 POC。如果追求低 bug 风险，不推荐作为长期方案。

### 方案 D：在 block-reply 层加入显式气泡边界

通过 skill 教模型在日常聊天中输出显式 bubble marker；core 增加一个很窄的处理点，把这些 marker 转换成多个 block reply。

模型内部输出示例：

```text
早~⟦bubble:1000⟧睡到快中午，真有你的杂鱼⟦bubble:300⟧话说胸口还痛吗？
```

用户实际看到：

```text
早~
```

```text
睡到快中午，真有你的杂鱼
```

```text
话说胸口还痛吗？
```

优点：
- 复用 OpenClaw 已有 block reply delivery path。
- replyTo、thread routing、delivery hooks、dedupe、媒体策略、diagnostics 都尽量保持现有行为。
- marker 是显式的，分割来自模型意图，不是长度算法。
- skill 可以控制语感，不需要模型日常调用工具。

缺点：
- 需要一个小 OpenClaw core patch。
- 必须小心处理最终回复去重。
- 需要严格限制适用范围，避免 marker 出现在严肃输出里。

结论：推荐。

## 推荐架构

采用 **方案 D：在 block-reply 层加入显式气泡边界，并配合一个聊天风格 skill**。

由两部分配合：

1. **气泡风格 skill**
   - 教模型什么时候可以使用 bubble marker。
   - 只在 Telegram 私聊日常闲聊里使用。
   - 使用频率要克制、自然。
   - 避免在菜单式回复、结构化回复、代码、debug、媒体、严肃健康/安全话题里使用。

2. **core bubble-boundary parser**
   - 在 block reply 发出前、final dedupe 比较前运行。
   - 解析 marker。
   - 发出多个 block reply payload。
   - 从最终可见文本里移除 marker。
   - 确保最终去重比较用的是“去掉 marker 后的可见文本”。

## Marker 格式

推荐格式：

```text
⟦bubble:1000⟧
```

含义：在这里拆分，并在下一条气泡前等待约 1000ms。

规则：

- delay 单位是毫秒。
- delay 允许范围：50–2500ms。
- delay 非法时回落到默认值，例如 800ms。
- 最多允许 3 个气泡，多余的 cap 或忽略。
- marker 只供内部使用，绝不能让用户看到。

阶段一先只保留一个 canonical marker，避免别名太多导致解析复杂。

## 触发范围

只有同时满足这些条件才启用气泡拆分：

- 频道是 Telegram。
- 聊天类型是私聊。
- payload 只有文本，没有媒体。
- payload 是普通助手回复，不是错误、推理块、压缩提示、工具摘要、审批提示或 TTS-only 消息。
- 文本里包含合法 bubble marker。
- 文本不包含代码块。
- 文本长度低于保守上限，例如 500 字符。
- 拆分后能得到 2–3 个非空气泡。

任一条件不满足，就走安全降级。

## 不应拆分的内容

以下内容出现时不要拆：

- fenced code blocks
- markdown 表格
- 编号列表或项目符号列表
- 多个 URL
- `MEDIA:` 指令
- `[[reply_to_current]]` 等 reply tag
- 明显结构化输出，例如 JSON/XML/SQL
- 错误信息或 stack trace
- 工具结果摘要

默认策略：不确定就一条消息。

## 延迟行为

阶段一建议：

- 尊重 marker 中的 delay。
- delay clamp 到 50–2500ms。
- 可以加少量 jitter，例如 ±15%；但低于 150ms 的 delay 不加 jitter。
- 如果未来支持无 delay marker，就使用 `humanDelay` natural 模式作为默认。

延迟应该在 block reply 发送附近实现，不应该通过拖慢模型生成实现。

## 最终回复去重要求

这是最重要的正确性要求。

如果 block reply 已经发了：

```text
早~
睡到快中午，真有你的杂鱼
话说胸口还痛吗？
```

最终回复阶段不能再发送一条合并后的原始文本。

因此实现必须在 duplicate comparison 前归一化 bubble marker。

必须满足：

```text
normalizeForDedupe("早~⟦bubble:1000⟧睡到快中午")
== normalizeForDedupe("早~睡到快中午")
```

比较对象应该是用户可见文本，而不是带 marker 的原始文本。

## 失败模式与降级

### marker 解析失败

降级：移除疑似 marker token，发送一条清理后的消息。

### 前几条 bubble 已发送，后续发送失败

降级：不要重发已经成功发出的 bubble。记录失败。只有在没有任何可见 bubble 成功发送时，才考虑发送单条 cleaned fallback。

### marker 出现在不支持的频道

降级：移除 marker，发送一条普通消息。

### 模型过度使用 marker

降级：最多 3 条；后续靠 skill 约束。必要时加 telemetry 和节流。

### marker 泄漏给用户

这是 release-blocking bug。测试必须覆盖 malformed marker 和 unsupported scope。

## Skill 指导草案

skill 应该教模型：

- 只在 Telegram 私聊日常闲聊里使用 bubble marker。
- 只有当真人也会自然分开发几条时才使用。
- 优先 2 条；偶尔 3 条；绝不超过 3 条。
- 不在技术工作、代码、长说明、医疗/安全紧急内容、结构化数据中使用。
- 不要把一个句子硬拆开。
- 每个气泡单独看也要自然。

好例子：

```text
早~⟦bubble:800⟧睡到快中午，真有你的杂鱼⟦bubble:300⟧胸口还痛吗？
```

坏例子：

```text
这里是实现方案：⟦bubble:1000⟧1. 修改 hook⟦bubble:1000⟧2. 写测试
```

## 测试计划

设计级测试用例：

1. Telegram 私聊日常文本，两个 marker 生成三条消息。
2. 显式 delay 被 clamp，并按顺序应用。
3. final dedupe 不会再发原始合并文本。
4. 群聊不拆分。
5. 带媒体不拆分。
6. 代码块里不拆分。
7. malformed marker 被移除或安全忽略。
8. 超过 3 个气泡会被 cap。
9. 没有 marker 的文本不变。
10. 工具摘要和错误回复不变。

## 待确认问题

1. session transcript 里应该保存原始 marker 文本，还是保存 cleaned 可见文本？
   - 建议：调试层保留模型原始输出；用户可见 replay/delivery 使用 cleaned 文本。

2. delay 应该精确执行，还是作为 hint？
   - 建议：作为 hint，经过 clamp 和 jitter。

3. 第一版应该只做 core，还是 core + skill 同时上？
   - 建议：先实现 parser 和安全测试，确认 marker 不会泄漏，再加入 skill 指导。

## 阶段一结论

推荐路线：

1. 本仓库作为设计/spec 工作区。
2. review 通过后，再写 implementation plan。
3. 实现目标应该是小 OpenClaw core patch，而不是 Telegram 发送绕路插件。
4. core parser 安全后，再加配套 skill。

## 2026-05-03 marker 格式修订

原设计使用 `⟦bubble:800⟧`。review 后考虑模型更容易学习 ASCII 风格 marker，曾尝试 `<|bubble:800|>`；但 OpenClaw 现有 sanitizer 会把 `<|...|>` 当模型 special token 清理，导致 marker 在进入 human-bubble parser 前被移除。

因此当前实现改为：

```text
<bubble:800>
```

这个格式比 `⟦⟧` 易写，也不会撞 OpenClaw 的 `<|...|>` special-token 清理逻辑。

## 2026-05-03 skill 打包修订

Skill 文案已改为泛用例子，不再使用私人对话中的句子。操作要求明确为：模型只输出一条带 `<bubble:delay>` marker 的普通最终回复，不调用 `message(send)` 多次，也不直接调用 Telegram API；OpenClaw core 负责把 marker 转成 block replies。

同时新增 `extensions/human-bubble-replies/`，作为插件形式携带该 skill。插件本身不劫持发送链路，只通过 `openclaw.plugin.json` 的 `skills` 字段发布 skill；实际解析和发送仍由 core block-reply 代码完成。
