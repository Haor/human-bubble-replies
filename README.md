# Human Bubble Replies：OpenClaw 日常聊天多气泡回复设计

这个仓库记录一种已经在真实 Telegram DM 中验证成功的 OpenClaw 改造思路：让模型在一条普通回复里写出显式气泡边界，例如：

```text
第一条测试消息<bubble:700>第二条测试消息
```

OpenClaw core 负责把它拆成多条正常 reply payload，并按 delay 串行发送。用户不会看到 `<bubble:700>`。

## 这个仓库是什么

这是一个**实现说明仓**，不是 OpenClaw 官方 PR，也不是可直接安装的 Telegram 插件。

它的目标是教会另一个 coding agent：

- 为什么不能只调 `humanDelay` 或降低 chunk 阈值；
- 为什么不要做 Telegram 侧外部 fan-out 插件；
- 应该在 OpenClaw 哪些发送链路里找入口；
- 怎么设计 parser / dedupe / delay / fallback；
- 真实环境失败时怎么定位；
- 怎么验收。

也就是“授人以渔”，不是要求 agent 机械套用某个固定 patch。OpenClaw 代码会变，但这套实现原则和排查路径仍然有效。

## 核心概念

### humanDelay 是节奏器，不是分镜器

OpenClaw 原有 `humanDelay` 只会在**已经存在的多条 block replies** 之间加停顿。它不决定一句话应该拆成几条。

Human Bubble Replies 要补的是前一步：让模型明确声明“这里应该成为新气泡”。

```text
第一条<bubble:800>第二条
```

### marker 只是模型到 core 的内部协议

最终 marker 固定为：

```text
<bubble:800>
```

不要用 `<|bubble:800|>`，因为 OpenClaw 可能会把 `<|...|>` 当作模型特殊 token 清理掉。

`⟦bubble:800⟧` 冲突少，但不常见，模型不一定稳定学会，所以也不作为推荐格式。

### 不能只改一条链路

真实测试证明，OpenClaw 至少有两条需要覆盖的发送路径：

1. **block streaming / block reply path**  
   长一点或 streaming 中产生的 assistant text 会进入 block reply emission 和 block reply pipeline。

2. **short final / followup delivery path**  
   很短的 final 回复可能绕过 block parser，直接进入 final delivery / `routeReply`。

如果只改第一条，短回复会漏。真实现象是 Telegram 吃掉 `<` `>`，用户看到：

```text
第一条bubble:700第二条
```

所以最终实现必须同时覆盖这两条路径。

## 推荐给 Agent 的英文指令

把下面这段发给 coding agent 即可。等你把仓库发到 GitHub 后，把 `IMPLEMENTATION_GUIDE.md` 换成真实链接：

```text
Please read IMPLEMENTATION_GUIDE.md in this repository before changing code. Do not blindly apply a fixed patch. Understand the Human Bubble Replies design, locate the equivalent block-reply and final-delivery paths in the current OpenClaw source tree, then implement the feature according to the guide. Preserve the marker format <bubble:800>. Do not implement a Telegram-side fan-out plugin. Run the required parser, pipeline, final-delivery, and live/manual acceptance checks before reporting completion.
```

如果是远程仓库，可以改成：

```text
Please read https://github.com/<owner>/<repo>/blob/main/IMPLEMENTATION_GUIDE.md before changing code. Follow the design and verification rules there.
```

## 仓库结构

```text
README.md                  # 中文说明，给人看
IMPLEMENTATION_GUIDE.md    # 英文实现指南，给 coding agent 看
```

## 最小验收

让模型输出：

```text
第一条测试消息<bubble:700>第二条测试消息
```

Telegram DM 应收到两条独立消息：

```text
第一条测试消息
```

约 700ms 后：

```text
第二条测试消息
```

并且：

- 不显示 `<bubble:700>`；
- 不显示 `bubble:700`；
- 不合并成一条空行消息；
- 不在 final 阶段重复发送合并文本。

## 模型侧 prompt / skill 要点

给模型的规则可以很短：

```text
Use <bubble:delayMs> only in Telegram direct casual chat when a short reply naturally fits 2-3 chat bubbles. Write one normal final reply; do not call message(send) multiple times. Do not use bubble markers for technical answers, lists, code, JSON, media, TTS, stickers, safety/medical urgency, or tool-result summaries. If unsure, send one normal reply without markers.
```

泛用示例：

```text
早，醒了吗<bubble:700>外面在下雨，出门记得带伞
```

```text
可以，先这样定<bubble:500>后面如果要改，我再帮你收一下边界
```

```text
别急，先喝口水<bubble:600>这事我们一点点拆，不用现在就全想明白
```

## 真实调试中踩过的坑

### 1. marker 被解析了，但消息被合并

现象：Telegram 收到一条消息，中间是空行。

原因：split 后的短 payload 又被 block reply coalescer 合并。

修复原则：带 `bubbleDelayMs` 的 payload 必须绕过 coalescer，必要时先 flush 已有 buffer。

### 2. 用户看到 `bubble:700`

现象：`<` 和 `>` 不见了，只剩 `bubble:700`。

原因：短 final 没经过 block parser，直接走 final delivery / Telegram rendering。

修复原则：在 final/followup delivery 层也做 bubble fan-out，位置要在 `routeReply` 或最终 dispatcher 前。

### 3. 成功拆了，但 final 又重复发一遍

原因：final dedupe 没把 marker 清理后再比较。

修复原则：dedupe key 和 final coverage 判断都要用 marker-stripped text。

## 许可证

发布前请补一个 `LICENSE`。
