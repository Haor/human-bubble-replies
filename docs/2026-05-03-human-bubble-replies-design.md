# Human Bubble Replies — Phase 1 Design

Date: 2026-05-03
Status: Draft for review
Scope: Design only; no implementation in this repository yet.

## Goal

Make Telegram direct-message chatting feel more human by allowing the assistant to intentionally split a single casual reply into two or three separate chat bubbles, with small per-bubble pauses.

Example target feel:

```text
早~
[wait 1s]
睡到快中午，真有你的杂鱼
[wait 300ms]
话说胸口还痛吗？
```

This is different from long-reply chunking. The goal is not to cut long text by size. The goal is to preserve conversational rhythm when a human would naturally send several short messages.

## Non-goals

- Do not lower global block-streaming thresholds as the main solution.
- Do not make every reply multi-bubble.
- Do not split task replies, code replies, structured explanations, errors, tool summaries, or privacy-sensitive content.
- Do not bypass OpenClaw's normal delivery path unless used only as a throwaway prototype.
- Do not rely on the model calling `message(send)` multiple times for ordinary chat.

## Existing OpenClaw mechanisms

OpenClaw already has three relevant pieces:

1. **Block replies**
   - Assistant output can be emitted as normal channel messages before the final answer.
   - This is the closest existing concept to multiple chat bubbles.

2. **Human delay**
   - `agents.defaults.humanDelay` can add randomized pauses between block replies.
   - It currently applies only to block replies, not final replies or tool summaries.

3. **Final duplicate suppression**
   - The block reply pipeline tries to avoid sending the same final answer again after block replies were already delivered.
   - Any bubble-marker design must preserve this guarantee.

These mechanisms match the desired shape, but they currently split by generated chunks/length/paragraphs. They do not understand intentional bubble boundaries.

## Candidate approaches

### Approach A — Lower block-streaming thresholds

Configure block streaming to emit smaller chunks more often.

Pros:
- No code change.
- Uses existing `block replies + humanDelay`.

Cons:
- Splits by size, not human intent.
- Can fragment long technical replies.
- Does not support explicit per-bubble delays.
- High risk of annoying output.

Decision: reject.

### Approach B — Skill asks model to call `message(send)` multiple times

The assistant sends each bubble through the message tool and then returns `NO_REPLY`.

Pros:
- Works with current OpenClaw.
- Each bubble is a real Telegram message.
- No core patch needed.

Cons:
- Unstable as a default behavior; model may forget `NO_REPLY`.
- Adds tool-call overhead to normal chat.
- Interacts badly with sticker/TTS/media workflows.
- Hard to enforce only in casual chat.

Decision: useful as a manual escape hatch, but not the default architecture.

### Approach C — `message_sending` plugin intercepts marker and sends Telegram messages itself

A plugin detects a marker like `⟦bubble:1000⟧`, manually sends several Telegram messages, and cancels the original outbound message.

Pros:
- Can be built without modifying OpenClaw core.
- Good for quick proof of concept.

Cons:
- Bypasses normal delivery internals.
- Risks inconsistent `message_sent` hooks, retry behavior, reply threading, link preview policy, media handling, TTS, and delivery diagnostics.
- If the plugin fails mid-send, it can leak raw markers or drop messages.

Decision: acceptable only as a temporary POC if core changes are blocked. Not recommended for the durable implementation.

### Approach D — Add explicit bubble boundaries to the block-reply layer

Teach the assistant, via a skill/system guidance, to emit explicit bubble markers in casual chat. Add a narrow core seam that converts those markers into multiple block replies before final delivery.

Example internal assistant text:

```text
早~⟦bubble:1000⟧睡到快中午，真有你的杂鱼⟦bubble:300⟧话说胸口还痛吗？
```

Delivered messages:

```text
早~
```

```text
睡到快中午，真有你的杂鱼
```

```text
话说胸口还痛吗？
```

Pros:
- Reuses OpenClaw's existing block-reply delivery path.
- Keeps replyTo/thread routing, delivery hooks, dedupe, media policy, and diagnostics close to current behavior.
- Marker is explicit, so splits are model-intended rather than length-based.
- Skill can control style without making the model use tools.

Cons:
- Requires a small OpenClaw core patch.
- Must be careful to avoid final duplicate sends.
- Needs strict scope gating to prevent marker leaks in serious outputs.

Decision: recommended.

## Recommended architecture

Use **Approach D: explicit bubble boundaries in the block-reply layer**, paired with a style skill.

There are two cooperating pieces:

1. **Bubble style skill**
   - Teaches the model when it may use bubble markers.
   - Only casual chat, especially Telegram DM.
   - Keeps usage sparse and natural.
   - Avoids menus, structured replies, code, debugging, media, and serious health/safety replies.

2. **Core bubble-boundary parser**
   - Runs before block replies are emitted and before final duplicate comparison.
   - Parses marker syntax.
   - Emits separate block reply payloads.
   - Removes markers from final visible text.
   - Ensures final dedupe compares normalized text without markers.

## Marker format

Recommended marker:

```text
⟦bubble:1000⟧
```

Meaning: split here and wait around 1000ms before the next bubble.

Rules:

- Delay is in milliseconds.
- Allowed delay range: 50–2500ms.
- Invalid delay falls back to configured default, likely 800ms.
- More than 3 bubbles is capped or ignored.
- Markers are internal-only and must never be visible to the user.

Alternative aliases can be considered later, but one canonical marker keeps phase one simple.

## Trigger scope

Bubble splitting should be enabled only when all of these are true:

- Channel is Telegram.
- Chat type is direct message.
- Payload is text-only.
- Payload is a normal assistant reply, not an error, reasoning block, compaction notice, tool summary, approval prompt, or TTS-only message.
- Text contains valid bubble markers.
- Text does not contain code fences.
- Text is below a conservative length limit, e.g. 500 characters.
- Split would produce 2–3 non-empty bubbles.

If any condition fails, the system should strip markers safely or ignore the feature depending on where validation occurs.

## Split safety rules

Do not split when text contains:

- fenced code blocks
- markdown tables
- numbered or bulleted lists
- multiple URLs
- `MEDIA:` directives
- reply tags like `[[reply_to_current]]`
- obvious structured output such as JSON/XML/SQL
- error text or stack traces
- tool result summaries

The safe default is one message.

## Delay behavior

Phase 1 recommendation:

- Honor explicit marker delay per boundary.
- Clamp delay to 50–2500ms.
- Add small jitter, e.g. ±15%, unless delay is below 150ms.
- If no delay is provided in a future marker variant, use `humanDelay` natural mode.

This should be implemented near block-reply sending, not by sleeping inside model generation.

## Final dedupe requirement

This is the most important correctness requirement.

If block replies send:

```text
早~
睡到快中午，真有你的杂鱼
话说胸口还痛吗？
```

The final answer must not later send the combined raw text again.

Therefore the implementation must normalize bubble markers before duplicate comparison.

Required invariant:

```text
normalizeForDedupe("早~⟦bubble:1000⟧睡到快中午")
== normalizeForDedupe("早~睡到快中午")
```

The system must compare visible text, not raw marker text.

## Failure modes and fallback

### Marker parser fails

Fallback: remove marker-like tokens and send one cleaned message.

### One bubble send fails after earlier bubbles were sent

Fallback: do not resend already delivered bubbles. Log the failure. Avoid sending the raw full message unless no visible bubble was delivered.

### Marker appears in unsupported channel

Fallback: strip marker and send one message.

### Model overuses markers

Fallback: cap bubbles at 3 and add skill guidance. Later add telemetry and throttling if needed.

### Marker leaks to user

This is a release-blocking bug. Tests must cover malformed and unsupported cases.

## Skill guidance draft

The skill should instruct:

- Use bubble markers only in Telegram direct casual chat.
- Use them when a human would naturally send separate short messages.
- Prefer 2 bubbles; 3 is occasional; never more.
- Do not use markers for technical work, code, long explanations, medical/safety urgency, or structured data.
- Do not split a single sentence unnaturally.
- Keep each bubble readable alone.

Good:

```text
早~⟦bubble:800⟧睡到快中午，真有你的杂鱼⟦bubble:300⟧胸口还痛吗？
```

Bad:

```text
这里是实现方案：⟦bubble:1000⟧1. 修改 hook⟦bubble:1000⟧2. 写测试
```

## Testing plan

Design-level test cases:

1. Casual Telegram DM with two markers produces three messages.
2. Explicit delays are clamped and applied in order.
3. Final dedupe does not send the raw combined message.
4. No split in group chat.
5. No split when media is attached.
6. No split inside code fences.
7. Malformed marker is stripped or safely ignored.
8. More than 3 bubbles is capped.
9. Text without markers is unchanged.
10. Tool summaries and error replies are unchanged.

## Open questions

1. Should marker text be persisted in session transcripts, or should transcripts store the cleaned visible text?
   - Recommendation: persist the assistant's original model output for debugging, but ensure user-visible replay/delivery uses cleaned visible text.

2. Should delay be exact or interpreted as a hint?
   - Recommendation: treat it as a hint with clamp and jitter.

3. Should the first implementation be core-only or include the skill immediately?
   - Recommendation: implement parser and safety first, then add skill guidance after tests prove markers cannot leak.

## Phase 1 decision

Recommended path:

1. Keep this repository as the design/spec workspace.
2. After review, write an implementation plan.
3. Implementation should target a small OpenClaw core patch, not a Telegram-send bypass plugin.
4. Add the companion skill only after the core parser is safe.
