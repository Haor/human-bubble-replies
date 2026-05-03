# Agent Instructions: Apply Human Bubble Replies to OpenClaw

You are applying the Human Bubble Replies patch to an OpenClaw source tree.

Read this file first. Follow the workflow exactly. Do not implement a Telegram-side fan-out plugin.

## Goal

Support explicit casual-chat bubble boundaries in normal assistant text:

```text
First bubble<bubble:700>Second bubble
```

OpenClaw must send this as separate normal reply payloads, preserving the regular delivery pipeline and hiding the marker from the user.

## Non-goals

Do not:

- lower generic block-streaming chunk thresholds to simulate bubbles
- call Telegram Bot API directly from a plugin
- make the model call `message(send)` multiple times as the default mechanism
- change the marker format away from `<bubble:800>`
- expose markers to users
- split technical, structured, media, TTS, sticker, or safety-critical replies

## Required implementation areas

Patch the target OpenClaw source tree in these areas:

1. Human bubble parser
   - Add or update `src/auto-reply/reply/human-bubble-markers.ts`.
   - Parse `<bubble:delayMs>` markers.
   - Validate safe scope/content.
   - Strip markers when unsupported or invalid.
   - Normalize marker-stripped text for final dedupe.

2. Block streaming emission
   - Update embedded/block reply emission so marker-split text becomes multiple reply payloads.
   - Each split payload should carry `bubbleDelayMs`.

3. Block reply pipeline
   - Honor `bubbleDelayMs` before sending that payload.
   - Do not let `bubbleDelayMs` payloads be merged by the coalescer.
   - Normalize markers for final dedupe.

4. Followup final delivery
   - Short final replies may bypass block streaming.
   - Expand human bubble markers in followup/final delivery before `routeReply` or dispatcher delivery.
   - Apply the same delay semantics.

5. Type surface
   - Add internal `bubbleDelayMs?: number` to the reply payload type used by core.

## Scope and safety rules

Only split when all of these are true:

- channel is Telegram
- chat type is direct/private
- text is short, approximately <= 500 characters
- split result has 2 or 3 non-empty bubbles
- no fenced code blocks
- no `MEDIA:` directives
- no reply tags such as `[[reply_to_current]]`
- no markdown tables
- no numbered/bulleted step lists
- no JSON-like structured payloads
- no multiple URLs
- no media payload attached

When unsupported or unsafe, strip the marker and send a single normal reply.

## Required tests

Add or update tests covering:

1. Parser behavior
   - valid 2-3 bubble split
   - delay parsing/clamping/defaulting
   - unsupported scope strips marker
   - unsafe content strips marker
   - dedupe normalization removes markers

2. Block streaming behavior
   - marker text becomes multiple block replies
   - invalid marker text becomes one cleaned reply

3. Pipeline behavior
   - `bubbleDelayMs` delays delivery
   - `bubbleDelayMs` payloads bypass coalescing
   - final dedupe ignores markers

4. Followup final delivery behavior
   - a short final payload such as `First<bubble:700>Second` is routed as two payloads
   - the second payload is delayed
   - marker is not visible to the user

## Verification command

Run targeted tests at minimum:

```bash
corepack pnpm test -- \
  src/auto-reply/reply/human-bubble-markers.test.ts \
  src/auto-reply/reply/block-reply-pipeline.test.ts \
  src/auto-reply/reply/followup-runner.test.ts \
  src/agents/pi-embedded-subscribe.human-bubble-block-reply.test.ts
```

Report exact pass/fail counts.

## Manual acceptance test

After installing the patched runtime, test in Telegram DM:

```text
First bubble<bubble:700>Second bubble
```

Expected result:

- Telegram receives two separate messages.
- The marker is not visible.
- The second message arrives after a short delay.
- No duplicate final merged message is sent.

## Debugging guidance

If the marker disappears and the message becomes one payload with a blank line, check the block reply coalescer. Bubble payloads must bypass coalescing.

If the user sees `bubble:700` without angle brackets, the short final reply likely bypassed block streaming and went directly to final delivery / routeReply. Add final delivery fan-out before routeReply.

If the original merged text is sent again after successful block replies, check final dedupe normalization.

## Completion criteria

The patch is complete only when:

- all targeted tests pass
- live/manual Telegram DM test passes
- marker is never visible to the user
- both block-streaming and short-final paths are covered
- invalid/unsupported content degrades to one marker-stripped message
