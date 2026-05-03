# Human Bubble Replies / OpenClaw 真人多气泡回复方案

这是一个面向 OpenClaw 的「日常聊天多气泡回复」开源资料仓。

它不是 OpenClaw 官方 PR，也不是一个能单独安装后就改写发送链路的外部插件。这个仓库发布的是：

- 已验证成功的设计方案
- 可复制的 core patch 实施流程
- 给 Agent 使用的英文执行说明
- 给模型使用的 prompt / skill
- 验收标准与测试清单

目标是让 OpenClaw 在 Telegram 私聊的日常聊天里，可以像真人一样把短回复自然拆成 2-3 个气泡，而不是把所有长回复都粗暴切块。

## 最终效果

模型输出一条普通 final 文本：

```text
第一条测试消息<bubble:700>第二条测试消息
```

OpenClaw core 解析后，Telegram 收到两条独立消息：

```text
第一条测试消息
```

约 700ms 后：

```text
第二条测试消息
```

用户不会看到 `<bubble:700>` marker。

## 为什么不是 humanDelay

OpenClaw 原有 `humanDelay` 是「节奏器」：它只在已经存在的多条 block replies 之间加停顿，不负责决定哪里拆成多条消息。

Human Bubble Replies 是「分镜器」：模型显式声明哪里应该成为新气泡：

```text
第一条<bubble:800>第二条
```

core 再把它转成多条正常 delivery payload。

两者不是替代关系：human bubble 负责拆分边界，humanDelay / pipeline delay 负责发送节奏。

## 为什么不是外部 Telegram 插件直接发送

外部插件如果在 `message_sending` hook 里拦截，再自己调用 Telegram Bot API fan-out，会绕过 OpenClaw 的正常 delivery、hooks、reply threading、dedupe、media/TTS 处理和错误恢复。

本方案选择更小但更正确的 core seam：

1. 模型只输出普通回复文本和 `<bubble:delay>` marker。
2. OpenClaw core 在 block/final reply 层解析 marker。
3. 拆出来的 payload 仍走 OpenClaw 正常发送链路。

## 仓库结构

```text
.
├── README.md                         # 中文说明，给人看
├── agent/PATCH_INSTRUCTIONS.md        # 英文说明，给执行 patch 的 Agent 看
├── docs/implementation-path.md        # 实现路径与核心文件说明
├── docs/verification.md               # 测试与验收规范
├── docs/live-debug-notes.md           # 真实环境踩坑与修复记录
├── skill/SKILL.md                     # 模型使用的 Human Bubble Replies skill/prompt
└── patch/human-bubble-replies.patch   # 从验证分支导出的最终状态 patch
```

## 快速使用方式

### 1. 让 Agent 执行 patch

把这段英文给你的 coding agent：

```text
Please read agent/PATCH_INSTRUCTIONS.md and follow it exactly. Apply the Human Bubble Replies patch to the target OpenClaw source tree, run the required tests, and report the verification output. Do not implement a Telegram-side fan-out plugin. Preserve the marker format <bubble:800>.
```

### 2. 安装/启用 skill

把 `skill/SKILL.md` 作为模型技能或 prompt 规则提供给你的 OpenClaw agent。

核心规则是：

- 只在 Telegram 私聊日常短回复中使用。
- 模型只输出一条 final 文本。
- 用 `<bubble:delayMs>` 表示气泡边界。
- 不要多次调用 `message(send)`。
- 不要直接调用 Telegram API。

### 3. 验收

最小验收：

```text
模型输出：第一条<bubble:700>第二条
Telegram 实际收到：两条独立消息
marker 不可见
两条之间有约 700ms 延迟
```

完整验收见：[`docs/verification.md`](docs/verification.md)

## 已验证的真实问题

真实 Telegram 测试中踩过两个坑：

1. **coalescer 合并问题**  
   marker 被正确解析，但两条短 payload 被 block reply coalescer 合并成一条带空行的消息。

2. **short final 旁路问题**  
   极短 final 回复可能不走 block streaming parser，而是直接进入 final followup delivery。Telegram 会吃掉 `<` `>`，用户只看到 `bubble:700`。

因此最终方案必须同时覆盖：

- block streaming reply 路径
- followup final delivery 路径

详细记录见：[`docs/live-debug-notes.md`](docs/live-debug-notes.md)

## Marker 格式

最终格式固定为：

```text
<bubble:800>
```

不要改成：

```text
<|bubble:800|>
```

原因：OpenClaw 的 model special-token sanitizer 会清理 `<|...|>`。

也不推荐：

```text
⟦bubble:800⟧
```

原因：冲突低，但不常见，模型不一定容易稳定学会。

## 适用范围

适合：

- Telegram 私聊
- 日常聊天
- 短回复
- 情绪/节奏自然拆分

不适合：

- 代码、调试、技术方案
- 长文、列表、表格、JSON
- 媒体/TTS/sticker/tool flow
- 医疗、安全、紧急提醒
- 群聊默认场景

## 当前验证状态

在验证分支中，以下 targeted tests 已通过：

```text
human-bubble-markers.test.ts                         10 passed
block-reply-pipeline.test.ts                         18 passed
followup-runner.test.ts                              31 passed
pi-embedded-subscribe.human-bubble-block-reply.test   2 passed
```

真实 Telegram DM 测试也已成功：`<bubble:700>` 被拆成两条独立消息，marker 不可见。

## 许可证

按你发布时选择的许可证处理。建议开源前补一个 `LICENSE` 文件。
