# Human Bubble Replies Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add safe, intentional, human-like multi-bubble replies for Telegram DM casual chat by parsing explicit bubble markers into block replies.

**Architecture:** Implement a small core bubble-marker parser and integrate it into OpenClaw's existing block-reply path before delivery and final dedupe. Add tests first, keep scope narrow, and add the companion skill only after marker stripping and duplicate prevention are proven safe.

**Tech Stack:** TypeScript/JavaScript in OpenClaw core, existing OpenClaw block-reply pipeline, existing Telegram delivery path, Node test runner used by the OpenClaw repository.

---

## File structure and responsibilities

This plan targets the OpenClaw source tree, not this design-only repository. The implementer should create a separate worktree from the OpenClaw repo before starting.

Expected files:

- Create: `src/auto-reply/reply/human-bubble-markers.ts`
  - Pure parser/normalizer for bubble markers.
  - No Telegram-specific delivery code.
  - Exports functions that can be unit-tested without channel runtime.

- Create: `src/auto-reply/reply/human-bubble-markers.test.ts`
  - Unit tests for parser, safety rules, clamping, marker stripping, and dedupe normalization.

- Modify: `src/agents/pi-embedded-subscribe/selection.ts` or equivalent source file that compiles into `dist/selection-*.js`
  - Integrate marker parsing at the block-reply emission boundary.
  - Ensure emitted block replies are split by marker.
  - Ensure raw markers are removed from visible final text.

- Modify: `src/auto-reply/reply/block-reply-pipeline.ts`
  - Ensure final duplicate comparison uses marker-normalized visible text.
  - Add optional per-payload bubble delay support if needed.

- Modify: `src/config/types.*` and schema files only if a config switch is added.
  - For phase one, prefer no public config unless tests reveal it is necessary.

- Create: `skills/human-bubble-replies/SKILL.md` or equivalent workspace skill after parser is safe.
  - Teaches the model when to emit markers.
  - Must be gated to casual Telegram DM.

- Create or modify docs only after behavior is stable:
  - `docs/concepts/streaming.md` may get a small section about explicit bubble boundaries if the feature becomes public.

## Implementation principles

- TDD first.
- Never leak marker text to users.
- Do not bypass OpenClaw channel delivery.
- Do not call Telegram Bot API directly.
- Default to one message when unsure.
- Keep the parser pure and small.
- Avoid broad config churn in the first implementation.

---

### Task 1: Add pure bubble marker parser tests

**Files:**
- Create: `src/auto-reply/reply/human-bubble-markers.test.ts`
- Create: `src/auto-reply/reply/human-bubble-markers.ts`

- [ ] **Step 1: Create the failing parser test file**

Create `src/auto-reply/reply/human-bubble-markers.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import {
  MAX_HUMAN_BUBBLE_PARTS,
  normalizeHumanBubbleTextForDedupe,
  parseHumanBubbleMarkers,
  stripHumanBubbleMarkers,
} from "./human-bubble-markers";

describe("human bubble markers", () => {
  test("splits valid marker text into ordered bubbles", () => {
    const result = parseHumanBubbleMarkers(
      "早~⟦bubble:1000⟧睡到快中午，真有你的杂鱼⟦bubble:300⟧胸口还痛吗？",
      { channel: "telegram", chatType: "direct" },
    );

    expect(result.ok).toBe(true);
    expect(result.parts).toEqual([
      { text: "早~", delayMs: 0 },
      { text: "睡到快中午，真有你的杂鱼", delayMs: 1000 },
      { text: "胸口还痛吗？", delayMs: 300 },
    ]);
    expect(result.cleanedText).toBe("早~睡到快中午，真有你的杂鱼胸口还痛吗？");
  });

  test("clamps delay to safe bounds", () => {
    const result = parseHumanBubbleMarkers(
      "a⟦bubble:1⟧b⟦bubble:99999⟧c",
      { channel: "telegram", chatType: "direct" },
    );

    expect(result.ok).toBe(true);
    expect(result.parts.map((part) => part.delayMs)).toEqual([0, 50, 2500]);
  });

  test("caps bubbles at MAX_HUMAN_BUBBLE_PARTS", () => {
    const result = parseHumanBubbleMarkers(
      "a⟦bubble:100⟧b⟦bubble:100⟧c⟦bubble:100⟧d⟦bubble:100⟧e",
      { channel: "telegram", chatType: "direct" },
    );

    expect(result.ok).toBe(true);
    expect(result.parts.length).toBe(MAX_HUMAN_BUBBLE_PARTS);
    expect(result.parts.map((part) => part.text)).toEqual(["a", "b", "c"]);
  });

  test("rejects group chat scope", () => {
    const result = parseHumanBubbleMarkers(
      "早⟦bubble:1000⟧醒了？",
      { channel: "telegram", chatType: "group" },
    );

    expect(result.ok).toBe(false);
    expect(result.reason).toBe("unsupported_scope");
    expect(result.cleanedText).toBe("早醒了？");
  });

  test("rejects unsupported channel", () => {
    const result = parseHumanBubbleMarkers(
      "早⟦bubble:1000⟧醒了？",
      { channel: "discord", chatType: "direct" },
    );

    expect(result.ok).toBe(false);
    expect(result.reason).toBe("unsupported_scope");
    expect(result.cleanedText).toBe("早醒了？");
  });

  test("rejects code fences", () => {
    const result = parseHumanBubbleMarkers(
      "```ts\nconst x = '⟦bubble:1000⟧';\n```",
      { channel: "telegram", chatType: "direct" },
    );

    expect(result.ok).toBe(false);
    expect(result.reason).toBe("unsafe_content");
  });

  test("rejects media directives", () => {
    const result = parseHumanBubbleMarkers(
      "图在这⟦bubble:1000⟧MEDIA:/tmp/a.png",
      { channel: "telegram", chatType: "direct" },
    );

    expect(result.ok).toBe(false);
    expect(result.reason).toBe("unsafe_content");
  });

  test("rejects list-like structured text", () => {
    const result = parseHumanBubbleMarkers(
      "计划：⟦bubble:1000⟧1. 写测试\n2. 写实现",
      { channel: "telegram", chatType: "direct" },
    );

    expect(result.ok).toBe(false);
    expect(result.reason).toBe("unsafe_content");
  });

  test("strips malformed marker-like tokens", () => {
    expect(stripHumanBubbleMarkers("早⟦bubble:oops⟧醒了？")).toBe("早醒了？");
    expect(stripHumanBubbleMarkers("早⟦bubble⟧醒了？")).toBe("早醒了？");
  });

  test("normalizes marker text for final dedupe", () => {
    expect(normalizeHumanBubbleTextForDedupe("早~⟦bubble:1000⟧睡醒了吗"))
      .toBe(normalizeHumanBubbleTextForDedupe("早~睡醒了吗"));
  });
});
```

- [ ] **Step 2: Create minimal module stub**

Create `src/auto-reply/reply/human-bubble-markers.ts`:

```ts
export const MAX_HUMAN_BUBBLE_PARTS = 3;

export type HumanBubbleScope = {
  channel?: string;
  chatType?: string;
};

export type HumanBubblePart = {
  text: string;
  delayMs: number;
};

export type HumanBubbleParseResult =
  | {
      ok: true;
      parts: HumanBubblePart[];
      cleanedText: string;
    }
  | {
      ok: false;
      reason:
        | "no_markers"
        | "unsupported_scope"
        | "unsafe_content"
        | "not_enough_parts"
        | "too_long";
      cleanedText: string;
      parts?: HumanBubblePart[];
    };

export function stripHumanBubbleMarkers(text: string): string {
  return text;
}

export function normalizeHumanBubbleTextForDedupe(text: string): string {
  return text.replace(/\s+/g, "").trim();
}

export function parseHumanBubbleMarkers(
  text: string,
  _scope: HumanBubbleScope = {},
): HumanBubbleParseResult {
  return {
    ok: false,
    reason: "no_markers",
    cleanedText: text,
  };
}
```

- [ ] **Step 3: Run tests and confirm failure**

Run from the OpenClaw repo root:

```bash
pnpm test -- src/auto-reply/reply/human-bubble-markers.test.ts
```

Expected: tests fail because parser stub does not split or strip markers.

- [ ] **Step 4: Commit failing tests and stub**

```bash
git add src/auto-reply/reply/human-bubble-markers.ts src/auto-reply/reply/human-bubble-markers.test.ts
git commit -m "test: add human bubble marker parser coverage"
```

---

### Task 2: Implement parser, stripping, and dedupe normalization

**Files:**
- Modify: `src/auto-reply/reply/human-bubble-markers.ts`
- Test: `src/auto-reply/reply/human-bubble-markers.test.ts`

- [ ] **Step 1: Replace parser stub with implementation**

Replace `src/auto-reply/reply/human-bubble-markers.ts` with:

```ts
export const MAX_HUMAN_BUBBLE_PARTS = 3;
export const HUMAN_BUBBLE_MIN_DELAY_MS = 50;
export const HUMAN_BUBBLE_MAX_DELAY_MS = 2500;
export const HUMAN_BUBBLE_DEFAULT_DELAY_MS = 800;
export const HUMAN_BUBBLE_MAX_TEXT_CHARS = 500;

const HUMAN_BUBBLE_MARKER_RE = /⟦bubble(?::([^\]⟧]*))?⟧/g;
const ANY_HUMAN_BUBBLE_MARKER_RE = /⟦bubble(?::[^\]⟧]*)?⟧/g;

export type HumanBubbleScope = {
  channel?: string;
  chatType?: string;
};

export type HumanBubblePart = {
  text: string;
  delayMs: number;
};

export type HumanBubbleParseResult =
  | {
      ok: true;
      parts: HumanBubblePart[];
      cleanedText: string;
    }
  | {
      ok: false;
      reason:
        | "no_markers"
        | "unsupported_scope"
        | "unsafe_content"
        | "not_enough_parts"
        | "too_long";
      cleanedText: string;
      parts?: HumanBubblePart[];
    };

function normalizeLower(value: string | undefined): string {
  return String(value ?? "").trim().toLowerCase();
}

function clampDelay(raw: string | undefined): number {
  const parsed = Number.parseInt(String(raw ?? ""), 10);
  const value = Number.isFinite(parsed) ? parsed : HUMAN_BUBBLE_DEFAULT_DELAY_MS;
  return Math.min(HUMAN_BUBBLE_MAX_DELAY_MS, Math.max(HUMAN_BUBBLE_MIN_DELAY_MS, value));
}

function hasUnsafeContent(text: string): boolean {
  if (/```/.test(text)) return true;
  if (/^\s*MEDIA:/m.test(text)) return true;
  if (/\[\[\s*reply_to(?::|_)current/i.test(text)) return true;
  if (/^\s*(?:[-*+]\s+|\d+[.)]\s+)/m.test(text)) return true;
  if (/\|[^\n]*\|[^\n]*\n\s*\|?\s*:?-{3,}/m.test(text)) return true;
  const urlCount = (text.match(/https?:\/\//g) ?? []).length;
  if (urlCount > 1) return true;
  const trimmed = text.trim();
  if (/^[{\[]/.test(trimmed) && /[}\]]$/.test(trimmed)) return true;
  return false;
}

export function stripHumanBubbleMarkers(text: string): string {
  return String(text ?? "").replace(ANY_HUMAN_BUBBLE_MARKER_RE, "");
}

export function normalizeHumanBubbleTextForDedupe(text: string): string {
  return stripHumanBubbleMarkers(text).replace(/\s+/g, "").trim();
}

function isSupportedScope(scope: HumanBubbleScope): boolean {
  return normalizeLower(scope.channel) === "telegram" && normalizeLower(scope.chatType) === "direct";
}

export function parseHumanBubbleMarkers(
  text: string,
  scope: HumanBubbleScope = {},
): HumanBubbleParseResult {
  const source = String(text ?? "");
  const cleanedText = stripHumanBubbleMarkers(source);

  if (!ANY_HUMAN_BUBBLE_MARKER_RE.test(source)) {
    return { ok: false, reason: "no_markers", cleanedText: source };
  }

  ANY_HUMAN_BUBBLE_MARKER_RE.lastIndex = 0;

  if (!isSupportedScope(scope)) {
    return { ok: false, reason: "unsupported_scope", cleanedText };
  }

  if (source.length > HUMAN_BUBBLE_MAX_TEXT_CHARS) {
    return { ok: false, reason: "too_long", cleanedText };
  }

  if (hasUnsafeContent(source)) {
    return { ok: false, reason: "unsafe_content", cleanedText };
  }

  const parts: HumanBubblePart[] = [];
  let cursor = 0;
  let nextDelayMs = 0;

  HUMAN_BUBBLE_MARKER_RE.lastIndex = 0;
  for (const match of source.matchAll(HUMAN_BUBBLE_MARKER_RE)) {
    const index = match.index ?? 0;
    const segment = source.slice(cursor, index).trim();
    if (segment) {
      parts.push({ text: segment, delayMs: nextDelayMs });
    }
    nextDelayMs = clampDelay(match[1]);
    cursor = index + match[0].length;
    if (parts.length >= MAX_HUMAN_BUBBLE_PARTS) break;
  }

  if (parts.length < MAX_HUMAN_BUBBLE_PARTS) {
    const tail = source.slice(cursor).trim();
    if (tail) parts.push({ text: tail, delayMs: nextDelayMs });
  }

  const cappedParts = parts.slice(0, MAX_HUMAN_BUBBLE_PARTS);
  if (cappedParts.length < 2) {
    return { ok: false, reason: "not_enough_parts", cleanedText, parts: cappedParts };
  }

  return {
    ok: true,
    parts: cappedParts,
    cleanedText,
  };
}
```

- [ ] **Step 2: Run parser tests**

```bash
pnpm test -- src/auto-reply/reply/human-bubble-markers.test.ts
```

Expected: all tests pass.

- [ ] **Step 3: Commit parser implementation**

```bash
git add src/auto-reply/reply/human-bubble-markers.ts src/auto-reply/reply/human-bubble-markers.test.ts
git commit -m "feat: parse human bubble reply markers"
```

---

### Task 3: Add block-reply emission integration tests

**Files:**
- Modify existing test near `src/agents/pi-embedded-subscribe/selection*.test.ts`, or create: `src/agents/pi-embedded-subscribe/human-bubble-block-reply.test.ts`
- Modify later: `src/agents/pi-embedded-subscribe/selection.ts`

- [ ] **Step 1: Locate current selection tests**

Run:

```bash
rg -n "emitBlockChunk|flushBlockReplyBuffer|onBlockReply|message_end|text_end" src test tests -g "*.test.ts" -g "*.spec.ts"
```

Expected: identify the test file that already exercises assistant stream/block reply behavior. If no suitable file exists, create `src/agents/pi-embedded-subscribe/human-bubble-block-reply.test.ts`.

- [ ] **Step 2: Add failing integration-style test**

Add this test to the selected file, adapting only helper names to the existing test harness:

```ts
test("emits human bubble markers as separate block replies", async () => {
  const blockReplies: Array<{ text?: string; bubbleDelayMs?: number }> = [];

  const ctx = createSubscribeTestContext({
    channel: "telegram",
    chatType: "direct",
    blockReplyBreak: "message_end",
    onBlockReply: async (payload) => {
      blockReplies.push({
        text: payload.text,
        bubbleDelayMs: (payload as { bubbleDelayMs?: number }).bubbleDelayMs,
      });
    },
  });

  await feedAssistantMessageEnd(ctx, {
    text: "早~⟦bubble:1000⟧睡到快中午，真有你的杂鱼⟦bubble:300⟧胸口还痛吗？",
  });

  expect(blockReplies).toEqual([
    { text: "早~", bubbleDelayMs: 0 },
    { text: "睡到快中午，真有你的杂鱼", bubbleDelayMs: 1000 },
    { text: "胸口还痛吗？", bubbleDelayMs: 300 },
  ]);
});
```

If the existing harness does not expose `createSubscribeTestContext` or `feedAssistantMessageEnd`, use the local equivalents in that test file. Do not invent a parallel harness if one exists.

- [ ] **Step 3: Add unsupported-scope integration test**

```ts
test("strips human bubble markers in unsupported block reply scope", async () => {
  const blockReplies: Array<{ text?: string }> = [];

  const ctx = createSubscribeTestContext({
    channel: "telegram",
    chatType: "group",
    blockReplyBreak: "message_end",
    onBlockReply: async (payload) => {
      blockReplies.push({ text: payload.text });
    },
  });

  await feedAssistantMessageEnd(ctx, {
    text: "早~⟦bubble:1000⟧醒了？",
  });

  expect(blockReplies).toEqual([{ text: "早~醒了？" }]);
});
```

- [ ] **Step 4: Run tests and confirm failure**

```bash
pnpm test -- <selected-test-file>
```

Expected: tests fail because integration does not yet split marker text.

- [ ] **Step 5: Commit failing integration tests**

```bash
git add <selected-test-file>
git commit -m "test: cover human bubble block replies"
```

---

### Task 4: Integrate parser into block-reply emission boundary

**Files:**
- Modify: `src/agents/pi-embedded-subscribe/selection.ts` or equivalent source file
- Modify type if needed: `src/auto-reply/reply-payload.ts` or `src/agents/pi-embedded-payloads.ts`

- [ ] **Step 1: Add optional delay metadata to block reply payload type**

Find the `BlockReplyPayload` type. If it is separate from `ReplyPayload`, add:

```ts
export type BlockReplyPayload = ReplyPayload & {
  bubbleDelayMs?: number;
};
```

If `BlockReplyPayload` is already an object type, add only:

```ts
bubbleDelayMs?: number;
```

- [ ] **Step 2: Import parser in selection module**

At the top of the block-reply selection source file, add:

```ts
import {
  parseHumanBubbleMarkers,
  stripHumanBubbleMarkers,
} from "../../auto-reply/reply/human-bubble-markers";
```

Adjust relative path to match the actual file location.

- [ ] **Step 3: Resolve bubble scope from existing session context**

Near `emitBlockChunk`, add a small helper:

```ts
function resolveHumanBubbleScope(params: SubscribeEmbeddedPiSessionParams): {
  channel?: string;
  chatType?: string;
} {
  return {
    channel: params.messageProvider ?? params.messageChannel,
    chatType: params.groupId ? "group" : "direct",
  };
}
```

If the local context already carries `ChatType`, prefer that exact value.

- [ ] **Step 4: Split marker chunks before `emitBlockReply`**

Inside `emitBlockChunk`, after reply directives have been consumed and after `cleanedText` is available, replace the single `emitBlockReply({ text: cleanedText, ... })` with:

```ts
const bubbleResult = parseHumanBubbleMarkers(
  cleanedText,
  resolveHumanBubbleScope(params),
);

if (bubbleResult.ok) {
  for (const part of bubbleResult.parts) {
    emitBlockReply({
      text: part.text,
      mediaUrls: mediaUrls?.length ? mediaUrls : void 0,
      audioAsVoice,
      replyToId,
      replyToTag,
      replyToCurrent,
      bubbleDelayMs: part.delayMs,
    }, { assistantMessageIndex: options?.assistantMessageIndex ?? state.assistantMessageIndex });
  }
  return;
}

emitBlockReply({
  text: stripHumanBubbleMarkers(cleanedText),
  mediaUrls: mediaUrls?.length ? mediaUrls : void 0,
  audioAsVoice,
  replyToId,
  replyToTag,
  replyToCurrent,
}, { assistantMessageIndex: options?.assistantMessageIndex ?? state.assistantMessageIndex });
```

If media is present, do not split. Use cleaned fallback:

```ts
const hasAnyMedia = Boolean(mediaUrls?.length) || Boolean(audioAsVoice);
if (hasAnyMedia) {
  emitBlockReply({
    text: stripHumanBubbleMarkers(cleanedText),
    mediaUrls: mediaUrls?.length ? mediaUrls : void 0,
    audioAsVoice,
    replyToId,
    replyToTag,
    replyToCurrent,
  }, { assistantMessageIndex: options?.assistantMessageIndex ?? state.assistantMessageIndex });
  return;
}
```

- [ ] **Step 5: Run integration tests**

```bash
pnpm test -- <selected-test-file>
```

Expected: new block-reply tests pass.

- [ ] **Step 6: Run parser tests again**

```bash
pnpm test -- src/auto-reply/reply/human-bubble-markers.test.ts
```

Expected: pass.

- [ ] **Step 7: Commit block emission integration**

```bash
git add src/agents/pi-embedded-subscribe/selection.ts src/auto-reply/reply-payload.ts src/agents/pi-embedded-payloads.ts <selected-test-file>
git commit -m "feat: emit human bubble markers as block replies"
```

Only include files that actually changed.

---

### Task 5: Apply per-bubble delay in block reply pipeline

**Files:**
- Modify: `src/auto-reply/reply/block-reply-pipeline.ts`
- Test: existing block reply pipeline test, or create `src/auto-reply/reply/block-reply-pipeline.human-bubble.test.ts`

- [ ] **Step 1: Add delay test**

Create or modify a block-reply-pipeline test:

```ts
import { describe, expect, test, vi } from "vitest";
import { createBlockReplyPipeline } from "./block-reply-pipeline";

describe("block reply human bubble delay", () => {
  test("waits for bubbleDelayMs before sending later bubble payloads", async () => {
    vi.useFakeTimers();
    const sent: string[] = [];

    const pipeline = createBlockReplyPipeline({
      timeoutMs: 10_000,
      onBlockReply: async (payload) => {
        sent.push(payload.text ?? "");
      },
    });

    pipeline.enqueue({ text: "早~", bubbleDelayMs: 0 });
    pipeline.enqueue({ text: "睡到快中午", bubbleDelayMs: 1000 });

    await vi.runOnlyPendingTimersAsync();
    expect(sent).toEqual(["早~"]);

    await vi.advanceTimersByTimeAsync(999);
    expect(sent).toEqual(["早~"]);

    await vi.advanceTimersByTimeAsync(1);
    await pipeline.flush({ force: true });
    expect(sent).toEqual(["早~", "睡到快中午"]);

    vi.useRealTimers();
  });
});
```

Adapt imports to the existing test style if needed.

- [ ] **Step 2: Run test and confirm failure**

```bash
pnpm test -- <block-reply-pipeline-test-file>
```

Expected: fails because `bubbleDelayMs` is not honored.

- [ ] **Step 3: Implement delay support**

In `src/auto-reply/reply/block-reply-pipeline.ts`, inside the queued send chain before calling `onBlockReply(payload, ...)`, add:

```ts
const explicitBubbleDelayMs = typeof payload.bubbleDelayMs === "number"
  && Number.isFinite(payload.bubbleDelayMs)
  ? Math.max(0, Math.min(2500, Math.floor(payload.bubbleDelayMs)))
  : 0;

if (explicitBubbleDelayMs > 0) {
  await sleep(explicitBubbleDelayMs);
}
```

This delay should be independent from existing `humanDelay`. If existing `humanDelay` also applies to block replies, avoid double delay for explicit bubble markers by using this rule:

```ts
const shouldApplyHumanDelay = explicitBubbleDelayMs <= 0;
```

- [ ] **Step 4: Run delay tests**

```bash
pnpm test -- <block-reply-pipeline-test-file>
```

Expected: pass.

- [ ] **Step 5: Run previous parser and integration tests**

```bash
pnpm test -- src/auto-reply/reply/human-bubble-markers.test.ts
pnpm test -- <selected-integration-test-file>
```

Expected: pass.

- [ ] **Step 6: Commit delay support**

```bash
git add src/auto-reply/reply/block-reply-pipeline.ts <block-reply-pipeline-test-file>
git commit -m "feat: honor human bubble block delays"
```

---

### Task 6: Prevent final duplicate sends with marker-normalized comparison

**Files:**
- Modify: `src/auto-reply/reply/block-reply-pipeline.ts`
- Test: existing block reply pipeline dedupe tests, or create one.

- [ ] **Step 1: Add final dedupe test**

Add:

```ts
import { normalizeHumanBubbleTextForDedupe } from "./human-bubble-markers";

test("human bubble marker normalization matches visible final text", () => {
  expect(normalizeHumanBubbleTextForDedupe("早~⟦bubble:1000⟧睡醒了吗"))
    .toBe(normalizeHumanBubbleTextForDedupe("早~睡醒了吗"));
});
```

Add a pipeline-level test if the existing pipeline exposes `hasSentPayload`:

```ts
test("hasSentPayload ignores bubble markers when checking final duplicate", async () => {
  const pipeline = createBlockReplyPipeline({
    timeoutMs: 10_000,
    onBlockReply: async () => {},
  });

  pipeline.enqueue({ text: "早~" });
  pipeline.enqueue({ text: "睡醒了吗" });
  await pipeline.flush({ force: true });

  expect(pipeline.hasSentPayload({ text: "早~⟦bubble:1000⟧睡醒了吗" })).toBe(true);
});
```

- [ ] **Step 2: Run dedupe test and confirm failure**

```bash
pnpm test -- <block-reply-pipeline-test-file>
```

Expected: pipeline-level `hasSentPayload` fails until normalization is integrated.

- [ ] **Step 3: Use marker-normalized text in content keys**

In `src/auto-reply/reply/block-reply-pipeline.ts`, import:

```ts
import { normalizeHumanBubbleTextForDedupe } from "./human-bubble-markers";
```

Where text is normalized for comparison, change:

```ts
const normalize = (text: string) => text.replace(/\s+/g, "");
```

to:

```ts
const normalize = (text: string) => normalizeHumanBubbleTextForDedupe(text);
```

If `createBlockReplyContentKey` uses raw text, update it to use stripped marker text:

```ts
const normalizedText = normalizeHumanBubbleTextForDedupe(reply.trimmedText);
```

Do not change media URL key behavior.

- [ ] **Step 4: Run dedupe tests**

```bash
pnpm test -- <block-reply-pipeline-test-file>
```

Expected: pass.

- [ ] **Step 5: Run all human bubble tests**

```bash
pnpm test -- src/auto-reply/reply/human-bubble-markers.test.ts
pnpm test -- <selected-integration-test-file>
pnpm test -- <block-reply-pipeline-test-file>
```

Expected: pass.

- [ ] **Step 6: Commit dedupe fix**

```bash
git add src/auto-reply/reply/block-reply-pipeline.ts <block-reply-pipeline-test-file>
git commit -m "fix: normalize human bubble markers for final dedupe"
```

---

### Task 7: Add skill guidance after core safety is proven

**Files:**
- Create: `skills/human-bubble-replies/SKILL.md`
- Modify skill registry only if OpenClaw requires explicit skill installation metadata.

- [ ] **Step 1: Create skill file**

Create `skills/human-bubble-replies/SKILL.md`:

```md
---
name: human-bubble-replies
description: Use in Telegram direct casual chat when a short reply would feel more natural as two or three separate chat bubbles. Do not use for technical, structured, medical/safety, media, or task-heavy replies.
---

# Human Bubble Replies

Use this skill only for casual Telegram direct-message chat.

When a human would naturally send a short reply as multiple small messages, you may insert explicit bubble markers into the final assistant text:

```text
第一条⟦bubble:800⟧第二条⟦bubble:300⟧第三条
```

The marker means: split here and wait roughly that many milliseconds before the next bubble.

## Rules

- Use only in Telegram DM casual chat.
- Prefer 2 bubbles. 3 bubbles is occasional. Never more than 3.
- Do not use markers in technical work, code, debugging, implementation plans, long explanations, structured data, lists, error messages, health/safety urgency, or media replies.
- Do not split one sentence unnaturally.
- Each bubble should make sense by itself.
- If unsure, do not use markers.

## Good examples

```text
早~⟦bubble:800⟧睡到快中午，真有你的杂鱼⟦bubble:300⟧胸口还痛吗？
```

```text
行吧，先点外卖⟦bubble:600⟧下雨天窝着确实舒服~
```

## Bad examples

```text
这里是实现方案：⟦bubble:800⟧1. 修改 hook⟦bubble:800⟧2. 写测试
```

```text
胸痛如果加重⟦bubble:800⟧请立刻就医
```
```

- [ ] **Step 2: Do not enable skill globally until tests pass in a real Telegram DM**

Run the core tests first. Then manually load the skill only for a controlled chat test.

- [ ] **Step 3: Commit skill**

```bash
git add skills/human-bubble-replies/SKILL.md
git commit -m "docs: add human bubble reply skill guidance"
```

---

### Task 8: Manual Telegram validation

**Files:**
- No code files required.
- Optional: `docs/manual-tests/human-bubble-replies.md`

- [ ] **Step 1: Create manual test notes**

Create `docs/manual-tests/human-bubble-replies.md`:

```md
# Human Bubble Replies Manual Tests

## Environment

- Channel: Telegram DM
- Feature: human bubble marker split
- Date:
- Tester:

## Tests

### Casual greeting

Input prompt:

```text
早啊
```

Expected:

- Assistant sends 2–3 short bubbles.
- No marker text visible.
- No duplicate final combined reply.

Actual:

```text
[fill during manual run]
```

### Technical reply should not split

Input prompt:

```text
解释一下这个 implementation plan 哪里需要改
```

Expected:

- Single normal technical reply.
- No bubble markers visible.

Actual:

```text
[fill during manual run]
```

### Media or sticker flow should not split

Input prompt:

```text
发一张图/贴纸相关回复
```

Expected:

- Existing media/sticker behavior unchanged.
- No bubble markers visible.

Actual:

```text
[fill during manual run]
```
```

- [ ] **Step 2: Run manual Telegram DM test**

Use a casual prompt where skill guidance is active. Verify messages arrive as multiple bubbles with no raw marker.

- [ ] **Step 3: Check logs for errors**

Run:

```bash
openclaw gateway status
```

If logs are available:

```bash
openclaw logs --tail 200
```

Expected: no delivery errors from human bubble handling.

- [ ] **Step 4: Commit manual notes**

```bash
git add docs/manual-tests/human-bubble-replies.md
git commit -m "test: document human bubble Telegram validation"
```

---

### Task 9: Documentation and release guardrails

**Files:**
- Modify: `docs/concepts/streaming.md`
- Optional modify: config reference only if a config flag was added.

- [ ] **Step 1: Add a short docs section**

In `docs/concepts/streaming.md`, under “Human-like pacing between blocks”, add:

```md
### Explicit casual-chat bubble boundaries

Telegram DM casual-chat replies can optionally use explicit bubble boundaries
when the human-bubble-replies skill is active. The model emits internal marker
syntax such as `⟦bubble:800⟧`; OpenClaw strips the marker and sends the adjacent
text as separate block replies with a short delay.

This mechanism is intentionally narrow:

- Telegram direct messages only.
- Text-only casual replies only.
- Maximum three bubbles.
- Markers are stripped before user-visible delivery and final duplicate checks.
- Structured, code, media, error, and tool-result replies are not split.
```

- [ ] **Step 2: Run docs lint if available**

Check package scripts:

```bash
cat package.json | jq '.scripts'
```

Run the closest docs or markdown check available. If none exists, skip with a note in the commit message body.

- [ ] **Step 3: Commit docs**

```bash
git add docs/concepts/streaming.md
git commit -m "docs: explain explicit human bubble replies"
```

---

## Verification checklist

Before claiming complete:

- [ ] Parser tests pass.
- [ ] Block reply integration tests pass.
- [ ] Block reply delay tests pass.
- [ ] Final dedupe test passes.
- [ ] Existing Telegram delivery tests pass, or a targeted subset passes if full suite is too heavy.
- [ ] Manual Telegram DM test shows multiple bubbles.
- [ ] No raw `⟦bubble` marker appears in Telegram.
- [ ] No duplicate final combined reply appears.
- [ ] Technical replies remain single-message unless naturally block-streamed by existing behavior.

## Self-review against spec

Spec coverage:

- Explicit marker format: Tasks 1–2.
- Trigger scope: Tasks 1–4.
- Safe fallback and marker stripping: Tasks 1–4.
- Per-bubble delay: Task 5.
- Final dedupe: Task 6.
- Skill guidance: Task 7.
- Manual Telegram validation: Task 8.
- Docs: Task 9.

Placeholder scan:

- This plan intentionally names exact target files where known.
- One source file location may differ depending on current OpenClaw source layout: `src/agents/pi-embedded-subscribe/selection.ts`. Task 3 requires locating the exact existing test/source file with `rg` before editing. This is acceptable because the compiled dist currently maps this logic to `dist/selection-*.js`, while source layout may differ by checkout.

Risk notes:

- The highest-risk area is final dedupe. Do not merge without Task 6 passing.
- The second highest-risk area is marker leakage. Do not enable skill before parser stripping tests pass.
- Avoid `message_sending` Telegram side-send implementation unless core integration proves impossible.
