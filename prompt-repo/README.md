# Human Bubble Replies Prompt Pack

Prompt/skill pack for OpenClaw human-like multi-bubble casual chat replies.

This repository does not implement the delivery feature by itself. It documents the required OpenClaw core patch and provides the skill prompt that teaches models when and how to emit bubble markers.

## Marker

```text
<bubble:800>
```

Example model output:

```text
早，醒了吗<bubble:700>外面在下雨，出门记得带伞
```

OpenClaw core strips the marker and sends separate block replies with the requested delay.

## Required OpenClaw core support

The core implementation lives in an OpenClaw branch/patch, not in this prompt pack:

- `src/auto-reply/reply/human-bubble-markers.ts`
- `src/agents/pi-embedded-subscribe.ts`
- `src/auto-reply/reply/block-reply-pipeline.ts`

Without that patch, this prompt pack should not be enabled: markers may be visible to users.
