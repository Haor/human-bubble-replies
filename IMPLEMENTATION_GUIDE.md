# Human Bubble Replies Implementation Guide

This guide explains how to implement Human Bubble Replies in OpenClaw-like codebases. Do not blindly apply a fixed patch. The source layout may change. Use this guide to find the equivalent paths and preserve the behavior.

## 1. Problem statement

We want the model to emit one normal assistant reply containing explicit bubble boundaries:

```text
First bubble<bubble:700>Second bubble
```

The runtime should deliver this as multiple normal reply payloads:

1. `First bubble`
2. wait about `700ms`
3. `Second bubble`

The marker must never be visible to the user.

## 2. Non-goals

Do not solve this by:

- lowering generic block-streaming chunk thresholds;
- relying on random/natural `humanDelay` alone;
- making the model call `message(send)` multiple times;
- implementing a Telegram-side fan-out plugin that calls the Bot API directly;
- changing the marker format away from `<bubble:800>`.

The feature should reuse the normal reply delivery pipeline as much as possible.

## 3. Mental model

Existing `humanDelay` is a pacing mechanism. It delays between payloads that already exist.

Human Bubble Replies need a boundary mechanism. The model declares where separate payloads should be created.

Therefore the implementation has two responsibilities:

1. parse marker-bearing assistant text into multiple reply payloads;
2. deliver those payloads sequentially, with explicit per-boundary delay.

## 4. Marker protocol

Canonical marker:

```text
<bubble:800>
```

Meaning: start a new bubble after this marker and wait about 800ms before sending that next bubble.

Delay rules:

- parse integer milliseconds;
- default to about `800ms` if omitted/invalid;
- clamp to a safe range, e.g. `50..2500ms`;
- first bubble has delay `0`.

Avoid `<|bubble:800|>` because OpenClaw-style model special token sanitizers often strip `<|...|>`.

## 5. Parser design

Create a small pure parser. Keep it independent of Telegram APIs.

Suggested behavior:

```ts
parseHumanBubbleMarkers(text, scope) ->
  | { ok: true, parts: [{ text, delayMs }...], cleanedText }
  | { ok: false, reason, cleanedText, parts? }
```

Only split when the scope and content are safe:

- channel is Telegram;
- chat type is direct/private;
- text is short, roughly <= 500 chars;
- split result has 2 or 3 non-empty bubbles;
- no fenced code blocks;
- no `MEDIA:` directives;
- no reply tags such as `[[reply_to_current]]`;
- no markdown tables;
- no numbered/bulleted procedural lists;
- no JSON-like structured payloads;
- no multiple URLs;
- no attached media payload.

If unsupported or unsafe, strip the marker and send one normal reply.

Also provide:

```ts
stripHumanBubbleMarkers(text)
normalizeHumanBubbleTextForDedupe(text)
```

Dedupe normalization should remove markers and whitespace differences.

## 6. Find the right code paths

Do not assume file names are stable. Search by behavior.

Useful search queries:

```bash
rg "createBlockReplyPipeline|blockReplyPipeline|onBlockReply"
rg "routeReply|followup|finalPayload|finalPayloads"
rg "coalescer|coalescing|joiner"
rg "hasSentPayload|createBlockReplyContentKey|dedupe"
rg "stripModelSpecialTokens|special token|<\\|"
```

You are looking for four conceptual places:

1. assistant text becomes block reply payloads;
2. block reply payloads are queued/sent/coalesced;
3. final/followup payloads are sent when block streaming does not emit;
4. final dedupe decides whether a final payload was already streamed.

## 7. Block streaming path

When assistant text is consumed into a block reply payload, run the parser before enqueueing/sending.

If parsing succeeds, emit multiple payloads:

```ts
{ text: parts[0].text, bubbleDelayMs: 0 }
{ text: parts[1].text, bubbleDelayMs: parts[1].delayMs }
```

Preserve metadata such as reply threading, voice flags, reasoning flags, etc., unless unsafe.

If parsing fails but markers existed, emit one cleaned payload with markers stripped.

## 8. Block reply pipeline path

The pipeline must honor `bubbleDelayMs`.

Before sending a payload:

```ts
if (payload.bubbleDelayMs > 0) await sleep(clampedDelay)
```

Important: bubble payloads must not be coalesced. If a generic coalescer merges short payloads, you will get one Telegram message with blank lines.

Correct behavior:

```ts
if (payload.bubbleDelayMs is finite number) {
  flush existing coalescer buffer
  send this payload directly through the normal send chain
  return
}
```

## 9. Short final / followup delivery path

This is the easy path to miss.

Short final replies may bypass block streaming entirely and go straight to final delivery, followup delivery, `routeReply`, or a dispatcher.

If that happens, Telegram/HTML rendering may eat `<` and `>`, leaving the user with `bubble:700` visible.

Therefore, add a second fan-out point in final/followup delivery before the final route/send call.

Conceptually:

```ts
const payloads = expandHumanBubblePayloads(finalPayloads, scope)
for (const payload of payloads) {
  if (payload.bubbleDelayMs > 0) await sleep(payload.bubbleDelayMs)
  await routeOrDispatch(payload)
}
```

Again, if parsing is unsafe/unsupported, strip markers and send one payload.

## 10. Final dedupe

If block replies already sent split bubbles, the final merged assistant text must not be sent again.

Any content key or coverage comparison should use marker-stripped text:

```ts
normalizeHumanBubbleTextForDedupe("First<bubble:700>Second")
== normalizeHumanBubbleTextForDedupe("FirstSecond")
```

Apply this where the runtime checks whether final payload text has already been streamed.

## 11. Type surface

If the codebase has a shared reply payload type, add an internal optional field:

```ts
bubbleDelayMs?: number
```

Do not expose this as a user-facing API unless the project intentionally wants to.

## 12. Tests to write

At minimum, write tests for these behaviors.

### Parser tests

- valid 2-bubble split;
- valid 3-bubble split;
- default/clamped delay;
- unsupported scope strips marker;
- unsafe content strips marker;
- dedupe normalization removes markers.

### Block emission tests

- marker text creates multiple block reply payloads;
- invalid marker text creates one cleaned payload.

### Pipeline tests

- `bubbleDelayMs` delays send;
- `bubbleDelayMs` payloads bypass coalescing;
- final dedupe ignores markers.

### Final delivery tests

- short final payload `First<bubble:700>Second` is routed as two payloads;
- second payload is delayed;
- marker is not passed to the final route/send call.

## 13. Manual acceptance test

In Telegram DM, force or prompt the model to output:

```text
First bubble<bubble:700>Second bubble
```

Expected:

- two separate Telegram messages;
- no `<bubble:700>`;
- no `bubble:700`;
- second message delayed;
- no duplicate merged final message.

## 14. Debugging guide

### Symptom: one message with a blank line

Likely cause: parser worked, but coalescer merged split payloads.

Fix: bubble payloads bypass coalescing.

### Symptom: user sees `bubble:700`

Likely cause: short final bypassed block parser and went directly to final delivery / routeReply.

Fix: add final/followup delivery fan-out before route/send.

### Symptom: original merged reply is sent after split bubbles

Likely cause: final dedupe compares raw text with markers.

Fix: marker-stripped dedupe normalization.

### Symptom: marker visible as `<bubble:700>`

Likely cause: no parser was reached, or parser scope rejected without stripping.

Fix: every unsupported path should strip markers before delivery.

## 15. Completion criteria

The implementation is complete only when:

- parser, block emission, pipeline, and final delivery tests pass;
- live/manual Telegram DM acceptance passes;
- markers are never visible to users;
- invalid content degrades to one marker-stripped message;
- no Telegram-side direct API fan-out is used;
- existing media, reply threading, TTS, sticker, and tool-result flows are not broken.
