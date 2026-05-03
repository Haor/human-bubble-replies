# Human Bubble Replies for OpenClaw

Human Bubble Replies 是一个面向 OpenClaw 的设计与实现指南，用于让 Telegram 私聊中的日常回复自然拆成多个聊天气泡。

模型可以在一条普通回复中写入显式气泡边界：

```text
第一条消息<bubble:700>第二条消息
```

运行时解析 marker 后，会通过正常 OpenClaw reply delivery 链路发送为两条独立消息，并在第二条前等待约 700ms。用户不会看到 `<bubble:700>`。

## 功能特点

- 使用简单的文本 marker：`<bubble:delayMs>`
- 支持 2-3 条短气泡回复
- 适用于 Telegram direct chat 的日常聊天场景
- 保留 OpenClaw 原有 delivery / hooks / dedupe / reply threading 链路
- 覆盖 block streaming replies 和 short final replies 两条路径
- 不依赖 Telegram 侧外部 fan-out 插件
- 不需要模型多次调用 `message(send)`

## 示例

模型输出：

```text
早，醒了吗<bubble:700>外面在下雨，出门记得带伞
```

Telegram 收到：

```text
早，醒了吗
```

约 700ms 后：

```text
外面在下雨，出门记得带伞
```

## 适用场景

适合：

- Telegram 私聊
- 日常聊天
- 短回复
- 情绪、节奏、停顿自然分开的场景

不适合：

- 技术方案、代码、调试输出
- 长文、列表、表格、JSON
- 媒体、TTS、sticker、tool result summary
- 医疗、安全或紧急提醒
- 默认群聊场景

## 给 Coding Agent 的指令

将下面这段指令交给 coding agent。发布到 GitHub 后，可以把文件路径替换成真实链接。

```text
Please read IMPLEMENTATION_GUIDE.md in this repository before changing code. Do not blindly apply a fixed patch. Understand the Human Bubble Replies design, locate the equivalent block-reply and final-delivery paths in the current OpenClaw source tree, then implement the feature according to the guide. Preserve the marker format <bubble:800>. Do not implement a Telegram-side fan-out plugin. Run the required parser, pipeline, final-delivery, and live/manual acceptance checks before reporting completion.
```

GitHub 链接版本示例：

```text
Please read https://github.com/<owner>/<repo>/blob/main/IMPLEMENTATION_GUIDE.md before changing code. Follow the design and verification rules there.
```

## 模型 Prompt 摘要

```text
Use <bubble:delayMs> only in Telegram direct casual chat when a short reply naturally fits 2-3 chat bubbles. Write one normal final reply; do not call message(send) multiple times. Do not use bubble markers for technical answers, lists, code, JSON, media, TTS, stickers, safety/medical urgency, or tool-result summaries. If unsure, send one normal reply without markers.
```

## Marker 格式

推荐格式：

```text
<bubble:800>
```

不推荐使用 `<|bubble:800|>`，因为 OpenClaw 风格的 special-token sanitizer 可能会清理 `<|...|>`。

## 实现指南

完整实现说明见：

```text
IMPLEMENTATION_GUIDE.md
```

指南包含：

- 设计目标与非目标
- parser 规则
- block reply pipeline 集成点
- short final / followup delivery 集成点
- coalescer 与 dedupe 注意事项
- 测试清单
- 手动验收方式
- 常见失败现象与排查方法

## 验收标准

最小验收用例：

```text
第一条测试消息<bubble:700>第二条测试消息
```

预期结果：

- Telegram 收到两条独立消息
- marker 不可见
- 第二条有短延迟
- 不合并成一条空行消息
- 不重复发送 final 合并文本

## License

请在发布前根据需要添加许可证文件。
