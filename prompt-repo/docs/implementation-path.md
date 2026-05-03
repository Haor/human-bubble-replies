# Implementation Path

This prompt pack depends on a core OpenClaw patch.

## Core files

- `src/auto-reply/reply/human-bubble-markers.ts`
  - Parses `<bubble:delay>` markers.
  - Validates Telegram DM scope and safe text shape.
  - Strips malformed or unsupported markers.
  - Normalizes text for final dedupe.

- `src/agents/pi-embedded-subscribe.ts`
  - Converts marker-split text into multiple block reply payloads.
  - Falls back to one cleaned payload when scope/content is unsupported.

- `src/auto-reply/reply/block-reply-pipeline.ts`
  - Serializes block replies.
  - Honors `bubbleDelayMs` before sending each later bubble.
  - Uses marker-normalized text for final duplicate suppression.

## Tests

- `src/auto-reply/reply/human-bubble-markers.test.ts`
- `src/agents/pi-embedded-subscribe.human-bubble-block-reply.test.ts`
- `src/auto-reply/reply/block-reply-pipeline.test.ts`

## Runtime requirement

The model should only emit markers when the OpenClaw runtime has this core support. Otherwise markers may leak to the user.
