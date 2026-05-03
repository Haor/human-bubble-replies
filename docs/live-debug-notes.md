# 真实环境调试记录

## 背景

模拟测试通过后，将 patch 打进 live runtime：

```text
/usr/lib/node_modules/openclaw
```

真实 Telegram DM 测试暴露了两个模拟阶段没覆盖到的问题。

## 问题一：marker 清理了，但消息被合并

现象：

模型输出：

```text
好，来试一下<bubble:700>如果你看到的是两条消息，就说明 patch 生效了
```

Telegram 收到一条消息，中间是空行：

```text
好，来试一下

如果你看到的是两条消息，就说明 patch 生效了
```

判断：

- marker 不可见，说明 parser 已经生效。
- 但两条短 payload 被 block reply coalescer 合并了。

修复：

- `bubbleDelayMs` payload 不能进入 coalescer。
- 发送前先 flush 现有 coalescer buffer。
- 再独立 `sendPayload(...)`。

## 问题二：用户看到 `bubble:700`

现象：

Telegram 收到：

```text
这次再测一下bubble:700要是还合并...
```

`<` 和 `>` 消失，只剩 `bubble:700`。

判断：

- 这不是 parser 清理成功。
- 这是短 final reply 没有进入 block streaming parser。
- payload 直接走 followup final delivery / `routeReply`。
- Telegram/parse 层把 `<bubble:700>` 当成类 HTML/markup 标签处理，吞掉尖括号。

修复：

- 在 followup final delivery 层也做 human bubble fan-out。
- 在 `routeReply` 之前解析并展开 payload。
- 同样按 `bubbleDelayMs` 串行发送。

## 最终真实测试结果

模型输出：

```text
第一条测试消息<bubble:700>第二条测试消息
```

Telegram 成功收到两条独立消息，marker 不可见。

## 结论

完整实现必须覆盖两条路径：

1. block streaming reply path
2. short final / followup delivery path

只 patch block streaming path 不够。
