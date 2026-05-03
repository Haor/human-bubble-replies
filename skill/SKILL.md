---
name: human-bubble-replies
description: Use when replying in Telegram direct casual chat and a short response would feel more natural as two or three separate chat bubbles. Do not use for technical, structured, media, medical/safety, or task-heavy replies.
---

# Human Bubble Replies

Use only for casual Telegram direct-message chat when OpenClaw core supports `<bubble:delay>` markers.

## How to send

Write one normal final assistant reply that contains bubble markers:

```text
第一条<bubble:800>第二条<bubble:300>第三条
```

Do not call `message(send)` multiple times. Do not call Telegram APIs. Do not explain the marker to the user. OpenClaw strips the marker and sends the adjacent text as separate block replies.

The number is a delay hint in milliseconds before the next bubble. Keep it between 50 and 2500.

## Rules

- Use only in Telegram DM casual chat.
- Prefer 2 bubbles. Use 3 only occasionally. Never more than 3.
- Do not use markers in technical work, code, debugging, implementation plans, long explanations, structured data, lists, error messages, health/safety urgency, media replies, sticker replies, TTS replies, or tool-result summaries.
- Do not split one sentence unnaturally.
- Each bubble should make sense by itself.
- If unsure, send one normal reply with no marker.

## Generic good examples

```text
早，醒了吗<bubble:700>外面在下雨，出门记得带伞
```

```text
可以，先这样定<bubble:500>后面如果要改，我再帮你收一下边界
```

```text
别急，先喝口水<bubble:600>这事我们一点点拆，不用现在就全想明白
```

## Bad examples

Technical/task reply:

```text
这里是实现方案：<bubble:800>1. 修改 hook<bubble:800>2. 写测试
```

Health/safety urgency:

```text
如果胸痛加重<bubble:800>请立刻就医
```

Sticker/tool flow:

```text
我先选贴纸<bubble:500>然后再发文字
```

## Principle

Use bubble markers to preserve natural rhythm, not to chop paragraphs. Good bubble replies feel like separate short thoughts. Bad bubble replies feel like a formatted long answer cut into pieces.
