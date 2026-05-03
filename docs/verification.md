# 验证与验收规范

## 自动测试

最小测试命令：

```bash
corepack pnpm test -- \
  src/auto-reply/reply/human-bubble-markers.test.ts \
  src/auto-reply/reply/block-reply-pipeline.test.ts \
  src/auto-reply/reply/followup-runner.test.ts \
  src/agents/pi-embedded-subscribe.human-bubble-block-reply.test.ts
```

当前验证分支通过结果：

```text
human-bubble-markers.test.ts                         10 passed
block-reply-pipeline.test.ts                         18 passed
followup-runner.test.ts                              31 passed
pi-embedded-subscribe.human-bubble-block-reply.test   2 passed
```

合计：`61 passed`。

## 必测场景

### 1. Parser

- `第一条<bubble:700>第二条` 拆成 2 段。
- delay 缺省时使用默认值。
- delay 超范围时 clamp 到安全范围。
- 非 Telegram DM scope 不拆，只清理 marker。
- 代码块、列表、表格、MEDIA、reply tag、多 URL、JSON-like 内容不拆。

### 2. Block streaming path

模型 streaming/block reply 中产生：

```text
第一条<bubble:700>第二条
```

期望：

- `onBlockReply` 收到两条 payload。
- 第一条 `bubbleDelayMs = 0`。
- 第二条 `bubbleDelayMs = 700`。
- final dedupe 不重复发送合并文本。

### 3. Coalescer path

开启 block reply coalescing 时，短 bubble payload 不能被合并成：

```text
第一条

第二条
```

期望：两条独立 payload。

### 4. Short final / followup path

极短 final 回复可能绕过 block streaming，直接进入 followup final delivery。

输入：

```text
第一条<bubble:700>第二条
```

期望：

- final delivery 在 `routeReply` 前完成 fan-out。
- routeReply 被调用两次。
- 第二条按 delay 发送。
- 用户看不到 `<bubble:700>`，也看不到 `bubble:700`。

## 真实 Telegram 验收

在 Telegram DM 发起短测试，使模型输出：

```text
第一条测试消息<bubble:700>第二条测试消息
```

验收：

- Telegram 收到两条独立消息。
- 第一条：`第一条测试消息`
- 第二条：`第二条测试消息`
- 第二条有短延迟。
- 没有 marker 泄露。
- 没有重复合并 final。

## 回滚要求

如果 live patch 导致 gateway 异常：

1. 先恢复 dist 备份。
2. 再重启 gateway。
3. 确认 `openclaw gateway status` 为 running。
4. 确认 Telegram 能正常收发。

不要连续盲目重启；先看 pid、status、日志和端口监听。
