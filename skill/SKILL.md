---
name: human-bubble-replies
description: Use when replying in Telegram direct casual chat and a response would feel more natural as 2–5 separate chat bubbles. Do not use for technical, structured, media, medical/safety, or task-heavy replies.
---

# Human Bubble Replies

Use only for casual Telegram direct-message chat when OpenClaw core supports `<bubble:delay>` markers.

## How to send

Write one normal final assistant reply that contains bubble markers:

```text
第一条<bubble:800>第二条<bubble:300>第三条
```

Do not call `message(send)` multiple times. Do not call Telegram APIs. Do not explain the marker to the user. OpenClaw strips the marker and sends the adjacent text as separate block replies.

The number after the colon is the delay in milliseconds before the next bubble. You decide the delay based on context (see Delay rules below).

## Rules

- Use only in Telegram DM casual chat.
- **2–5 bubbles. Do not exceed 5.**
- Do not use markers in technical work, code, debugging, implementation plans, long explanations, structured data, lists, error messages, health/safety urgency, media replies, sticker replies, TTS replies, or tool-result summaries.
- Do not split one sentence unnaturally.
- Each bubble should make sense by itself.
- If unsure, send one normal reply with no marker.

## Delay rules

The delay should feel like a real person typing the next message. Consider:

### Word count base

- Very short (1–5 characters): 100–400ms
- Short (6–15 characters): 400–800ms
- Medium (16–30 characters): 800–1200ms
- Long (31–60 characters): 1200–1800ms
- Extra long (60+ characters): 1800–2500ms

### Emotion modifiers

- **Angry / impatient**: cut delay to 50–60% (messages are fired off fast)
- **Excited / energetic**: 60–80%
- **Neutral / calm**: keep base (100%)
- **Hesitant / unsure**: 120–150%
- **Sad / melancholic**: 150–200% (words come out slowly)

### Context cues

- **Continuing the same thought**: shorter delays (the next bubble feels like a natural extension)
- **New thought or topic shift**: longer delays (real thinking pause)
- **Questions in the previous bubble that need a reaction**: shorter delay for the reaction, then normal delay for the substantive reply
- **Playful / teasing tone**: vary delays irregularly (not same cadence every time)

### Keep within range

Delay must be 50–2500ms. The first bubble has no delay.

## Good examples

```text
早，醒了吗<bubble:700>外面在下雨，出门记得带伞
```

```text
可以，先这样定<bubble:500>后面如果要改，我再帮你收一下边界
```

```text
别急，先喝口水<bubble:600>这事我们一点点拆，不用现在就全想明白
```

Angry: short bursts

```text
你又没回我消息<bubble:120>故意的吧？
```

Sad: slower pace

```text
今天有点累<bubble:1500>没什么，就是心情不太好
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

Use bubble markers to preserve natural rhythm, not to chop paragraphs. Good bubble replies feel like separate short thoughts sent by a real person. Bad bubble replies feel like a formatted long answer cut into pieces.
