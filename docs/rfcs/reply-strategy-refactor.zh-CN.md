# RFC: 流式回复策略模式重构

- **状态**: Implemented
- **作者**: @aoxiaotian-ai
- **日期**: 2026-03-17
- **关联 Issue**: #360, #361, #314

---

## 1. 背景与动机

### 1.1 现状

`inbound-handler.ts` 共 1599 行，其中 `handleDingTalkMessage` 函数从第 269 行延伸到第 1599 行（约 1330 行）。流式回复相关逻辑（第 835 行起）约 760 行，结构如下：

```
handleDingTalkMessage()                           // 269-1599
├── Step 1-2: 自消息过滤 + 鉴权                     // 278-830
├── Step 3: 模式选择 + 卡片创建                      // 835-864
├── 引用消息恢复 / 媒体下载 / 附件抽取               // 866-1165
├── 上下文组装 + dispatch 准备                       // 1167-1285
├── Session lock 包裹                               // 1290-1598
│   ├── controller 创建                             // 1296-1300
│   ├── dispatchReply                               // 1302-1483
│   │   └── deliver 回调 (~180行)                   // 1308-1454
│   │       ├── media 投递                          // 1309-1347
│   │       ├── 空 payload 守卫                     // 1363-1369
│   │       ├── card final 分支                     // 1377-1393
│   │       ├── card tool 分支                      // 1396-1423
│   │       ├── media delivery (all modes)          // 1425-1428
│   │       └── non-card fallback                   // 1431-1446
│   ├── replyOptions                                // 1456-1480
│   ├── dispatch error -> card 收尾                  // 1484-1498
│   ├── Step 5: post-dispatch finalize (~90行)      // 1501-1576
│   └── finally: lock释放 + ack reaction 回收        // 1578-1598
```

### 1.2 问题

| 编号 | 问题 | 影响 |
|------|------|------|
| P1 | deliver 回调是一个 ~180 行的闭包，4 层 if-else 分支，捕获外层 10+ 个可变变量 | 难以理解、容易改错 |
| P2 | card/markdown 的 replyOptions 混在同一个对象字面量，用三元表达式切换 | `disableBlockStreaming` 逻辑混乱导致 #360 |
| P3 | Step 5 finalize 与 deliver 强耦合但物理隔开 90 行，共享 `finalTextForFallback`、`cardFinalized` 等变量 | fallback 路径创建重复卡片（#361 后遗症） |
| P4 | 每次新增功能（如 #314 的 dynamic reaction）都往闭包外层堆变量（~10 个） | 函数膨胀，状态交叉 |
| P5 | deliver 回调内部无法独立单测，必须 mock 整个 `handleDingTalkMessage` | 测试成本高 |

### 1.3 近期 Bug 根因回溯

| Bug | 根因 | 本质问题 |
|-----|------|---------|
| #360 markdown 消息被拆分 | `disableBlockStreaming` 在 markdown 模式下为 `undefined` | card/markdown 配置混在一起，修改一个容易影响另一个（P2） |
| #361 后 card fallback 创建重复卡片 | Step 5 fallback 调用 `sendMessage` 内部又走了 `sendProactiveCardText` | finalize/fallback 逻辑散落在主函数，缺乏封装（P3） |

---

## 2. 设计目标

1. **职责分离**: deliver 回调 + replyOptions + finalize 封装进策略对象，主函数只做协调
2. **状态隔离**: Card 和 Markdown 各自管理自己的内部状态，互不感知
3. **可扩展性**: 新功能（dynamic reaction 等）可以通过组合/装饰器模式加入，不污染核心流程
4. **渐进重构**: 分步实施，每步独立可测、可 review、可回滚
5. **零功能变更**: 用户可感知行为完全不变

### 非目标

- 不改变 runtime API 的调用方式（`dispatchReplyWithBufferedBlockDispatcher` 接口不变）
- 不改变 `card-service.ts`、`card-draft-controller.ts`、`send-service.ts` 的公开接口
- 不改变配置格式

---

## 3. 详细设计

### 3.1 策略接口

```typescript
// src/reply-strategy.ts

/** deliver 回调接收到的标准化载荷 */
export interface DeliverPayload {
  text?: string;
  mediaUrls: string[];
  kind: "block" | "final" | "tool";
}

/** 回复策略接口 — card/markdown 各实现一个 */
export interface ReplyStrategy {
  /** 返回挂接给 runtime 的 replyOptions */
  getReplyOptions(): {
    disableBlockStreaming: boolean;
    onPartialReply?: (payload: { text?: string }) => void;
    onReasoningStream?: (payload: { text?: string }) => void;
    onAssistantMessageStart?: () => void;
  };

  /** deliver 回调的具体实现，按 kind 分发 */
  deliver(payload: DeliverPayload): Promise<void>;

  /** dispatch 正常结束后的终态处理 */
  finalize(): Promise<void>;

  /** dispatch 异常时的清理 */
  abort(error: Error): Promise<void>;

  /** 获取最终文本（供外部消费，如日志） */
  getFinalText(): string | undefined;
}
```

### 3.2 策略共享参数

两个策略实现共用的上下文参数，提取为独立类型：

```typescript
export interface ReplyStrategyContext {
  config: DingTalkConfig;
  to: string;
  sessionWebhook: string;
  senderId: string;
  isDirect: boolean;
  accountId: string;
  storePath: string;
  groupId?: string;
  log?: Logger;
  deliverMedia: (urls: string[]) => Promise<void>;
}
```

### 3.3 MarkdownReplyStrategy

最简单的策略，只关心 `kind=final` 的投递：

```typescript
// src/reply-strategy-markdown.ts

export function createMarkdownReplyStrategy(
  ctx: ReplyStrategyContext,
): ReplyStrategy {

  let finalText: string | undefined;

  return {
    getReplyOptions() {
      return { disableBlockStreaming: true };
    },

    async deliver(payload) {
      if (payload.mediaUrls.length > 0) {
        await ctx.deliverMedia(payload.mediaUrls);
      }
      if (payload.kind === "final" && payload.text) {
        finalText = payload.text;
        await sendMessage(ctx.config, ctx.to, payload.text, {
          sessionWebhook: ctx.sessionWebhook,
          atUserId: !ctx.isDirect ? ctx.senderId : null,
          log: ctx.log,
          accountId: ctx.accountId,
          storePath: ctx.storePath,
          conversationId: ctx.groupId,
        });
      }
    },

    async finalize() { /* no-op */ },
    async abort() { /* no-op */ },
    getFinalText() { return finalText; },
  };
}
```

**内部状态**: 仅 1 个变量（`finalText`），对比现有 deliver 回调中 markdown 路径隐式依赖的 5+ 个外层变量。

### 3.4 CardReplyStrategy

封装 controller 生命周期、deliver 分支、finalize、fallback：

```typescript
// src/reply-strategy-card.ts

export function createCardReplyStrategy(
  ctx: ReplyStrategyContext & { card: AICardInstance },
): ReplyStrategy {

  const controller = createCardDraftController({
    card: ctx.card,
    log: ctx.log,
  });
  let finalTextForFallback: string | undefined;

  return {
    getReplyOptions() {
      return {
        disableBlockStreaming: true,

        onAssistantMessageStart: () => {
          controller.notifyNewAssistantTurn();
        },

        onPartialReply: ctx.config.cardRealTimeStream
          ? (p) => { if (p.text) controller.updateAnswer(p.text); }
          : undefined,

        onReasoningStream: (p) => {
          if (p.text) controller.updateReasoning(p.text);
        },
      };
    },

    async deliver(payload) {
      // 空内容守卫（card final 例外）
      if (!payload.text && payload.mediaUrls.length === 0) {
        if (payload.kind !== "final") return;
      }

      // ---- final: 延迟到 finalize ----
      if (payload.kind === "final") {
        if (payload.mediaUrls.length > 0) {
          await ctx.deliverMedia(payload.mediaUrls);
        }
        if (payload.text) finalTextForFallback = payload.text;
        return;
      }

      // ---- tool: 追加写入卡片 ----
      if (payload.kind === "tool") {
        if (controller.isFailed() || isCardInTerminalState(ctx.card.state)) return;
        await controller.flush();
        await controller.waitForInFlight();
        const toolText = payload.text
          ? formatContentForCard(payload.text, "tool") : "";
        if (toolText) {
          await sendMessage(ctx.config, ctx.to, toolText, {
            sessionWebhook: ctx.sessionWebhook,
            atUserId: !ctx.isDirect ? ctx.senderId : null,
            log: ctx.log,
            card: ctx.card,
            accountId: ctx.accountId,
            storePath: ctx.storePath,
            conversationId: ctx.groupId,
            cardUpdateMode: "append",
          });
        }
        return;
      }

      // ---- block: 仅投递 media（disableBlockStreaming=true 下通常不会触发）----
      if (payload.mediaUrls.length > 0) {
        await ctx.deliverMedia(payload.mediaUrls);
      }
    },

    async finalize() {
      if (ctx.card.state === AICardStatus.FINISHED) return;

      // 卡片失败 -> markdown 降级
      if (ctx.card.state === AICardStatus.FAILED || controller.isFailed()) {
        const fallback = finalTextForFallback
          || controller.getLastAnswerContent()
          || controller.getLastContent()
          || ctx.card.lastStreamedContent;
        if (fallback) {
          // 绕过 sendMessage 避免 sendProactiveCardText 创建第二张卡片
          if (ctx.sessionWebhook) {
            await sendBySession(ctx.config, ctx.sessionWebhook, fallback, {
              atUserId: !ctx.isDirect ? ctx.senderId : null,
              log: ctx.log,
            });
          } else {
            await sendProactiveTextOrMarkdown(
              { ...ctx.config, messageType: "markdown" },
              ctx.to, fallback,
              { log: ctx.log, accountId: ctx.accountId },
            );
          }
        }
        return;
      }

      // 正常 finalize
      await controller.flush();
      await controller.waitForInFlight();
      controller.stop();
      const finalText = controller.getLastAnswerContent()
        || finalTextForFallback || "Done";
      await finishAICard(ctx.card, finalText, ctx.log);
    },

    async abort(error) {
      controller.stop();
      await controller.waitForInFlight();
      if (!isCardInTerminalState(ctx.card.state)) {
        try {
          await finishAICard(ctx.card, "处理失败", ctx.log);
        } catch {
          ctx.card.state = AICardStatus.FAILED;
          ctx.card.lastUpdated = Date.now();
        }
      }
    },

    getFinalText() {
      return controller.getLastAnswerContent() || finalTextForFallback;
    },
  };
}
```

**内部状态**: 2 个变量（`controller` + `finalTextForFallback`），对比现有代码中 card 路径隐式依赖的 `controller`、`cardFinalized`、`finalTextForFallback`、`useCardMode`、`currentAICard` 等 5+ 个外层变量。

### 3.5 工厂函数

```typescript
// src/reply-strategy.ts（导出）

export function createReplyStrategy(
  params: ReplyStrategyContext & {
    card: AICardInstance | undefined;
    useCardMode: boolean;
  },
): ReplyStrategy {
  if (params.useCardMode && params.card) {
    return createCardReplyStrategy({ ...params, card: params.card });
  }
  return createMarkdownReplyStrategy(params);
}
```

### 3.6 inbound-handler.ts 简化后

原来 1290-1598 的约 310 行变成约 60 行：

```typescript
const releaseSessionLock = await acquireSessionLock(route.sessionKey);
try {
  const strategy = createReplyStrategy({
    config: dingtalkConfig,
    card: currentAICard,
    useCardMode,
    to, sessionWebhook, senderId, isDirect,
    accountId, storePath, groupId, log,
    deliverMedia: deliverMediaAttachments,
  });

  try {
    await rt.channel.reply.dispatchReplyWithBufferedBlockDispatcher({
      ctx,
      cfg,
      dispatcherOptions: {
        responsePrefix: "",
        deliver: async (payload, info) => {
          const mediaUrls = extractMediaUrls(payload);
          await strategy.deliver({
            text: payload.text,
            mediaUrls,
            kind: (info?.kind as DeliverPayload["kind"]) || "block",
          });
        },
      },
      replyOptions: strategy.getReplyOptions(),
    });
  } catch (dispatchErr: any) {
    await strategy.abort(dispatchErr);
    throw dispatchErr;
  }

  await strategy.finalize();
} finally {
  releaseSessionLock();
  // ack reaction 回收逻辑不变
}
```

### 3.7 扩展点：装饰器模式

未来新功能（如 PR #314 的 dynamic reaction）可以通过装饰器包裹策略，不修改核心代码：

```typescript
// src/reply-strategy-with-reaction.ts

export function withDynamicReaction(
  inner: ReplyStrategy,
  params: {
    config: DingTalkConfig;
    msgId: string;
    groupId?: string;
    runtime: unknown;
    log?: Logger;
  },
): ReplyStrategy {
  // 内部维护：currentReaction, heartbeatTimer, eventSubscription
  // 订阅 rt.events.onAgentEvent（带 session/runId 过滤）

  return {
    getReplyOptions: () => inner.getReplyOptions(),

    async deliver(payload) {
      // 可以在投递前/后根据 tool event 更新 reaction
      await inner.deliver(payload);
    },

    async finalize() {
      dispose(); // 停止心跳、取消订阅
      await inner.finalize();
    },

    async abort(error) {
      dispose();
      await inner.abort(error);
    },

    getFinalText: () => inner.getFinalText(),
  };
}
```

使用方式：

```typescript
let strategy = createReplyStrategy({ ... });
if (ackReaction === "emoji") {
  strategy = withDynamicReaction(strategy, { ... });
}
```

---

## 4. 改动前后对比

### 4.1 文件变更

| 文件 | 操作 | 说明 |
|------|------|------|
| `src/reply-strategy.ts` | **新增** | 策略接口 + 类型 + 工厂函数（~50 行） |
| `src/reply-strategy-markdown.ts` | **新增** | Markdown 策略实现（~60 行） |
| `src/reply-strategy-card.ts` | **新增** | Card 策略实现（~180 行） |
| `src/inbound-handler.ts` | **修改** | 删除 deliver/replyOptions/finalize 代码（~250 行），替换为策略调用（~60 行）；净减约 190 行 |
| `tests/unit/reply-strategy-markdown.test.ts` | **新增** | Markdown 策略独立单测 |
| `tests/unit/reply-strategy-card.test.ts` | **新增** | Card 策略独立单测（deliver 各 kind、finalize、abort、fallback） |
| `tests/unit/inbound-handler.test.ts` | **修改** | 现有测试行为断言不变，可能需要调整 mock 粒度 |

### 4.2 状态变量对比

| 变量 | 重构前（inbound-handler 函数作用域） | 重构后 |
|------|-------------------------------------|--------|
| `useCardMode` | 外层可变 `let` | 工厂函数参数，之后不再变化 |
| `currentAICard` | 外层可变 `let` | `CardReplyStrategy` 内部 `ctx.card` |
| `controller` | 外层 `const`，但通过闭包共享 | `CardReplyStrategy` 内部私有 |
| `cardFinalized` | 外层可变 `let` | 不再需要（finalize 由策略管理） |
| `finalTextForFallback` | 外层可变 `let` | `CardReplyStrategy` 内部私有 |

### 4.3 已知 Bug 的自然修复

| Bug | 修复方式 |
|-----|---------|
| `disableBlockStreaming` 混乱 | 每个策略独立声明 `{ disableBlockStreaming: true }`，不存在交叉影响 |
| fallback 创建重复卡片 | `CardReplyStrategy.finalize()` 内部处理降级，绕过 `sendMessage` 的卡片路径 |
| PR #314 状态膨胀 | dynamic reaction 作为装饰器独立存在，不往主函数堆变量 |

---

## 5. 实施计划

分 4 个 PR，每个独立可 review、可合并、可回滚：

| 阶段 | PR 内容 | 验证标准 |
|------|---------|---------|
| **PR 1** | 新增 `reply-strategy.ts` 接口 + `reply-strategy-markdown.ts`；inbound-handler 中 markdown 路径切到策略 | 全部 475 个测试通过；markdown 模式行为不变 |
| **PR 2** | 新增 `reply-strategy-card.ts`；deliver 的 card 分支、Step 5 finalize、abort 搬入策略；删除 inbound-handler 对应代码 | 全部测试通过；card 模式行为不变 |
| **PR 3** | `deliverMediaAttachments` 提取为独立函数；清理 inbound-handler 中残留的状态变量和 import | 全部测试通过；无功能变化 |
| **PR 4**（可选） | `reply-strategy-with-reaction.ts`，为 #314 的 dynamic reaction 提供集成点 | 新增测试覆盖 reaction 装饰器 |

### 风险控制

- **每个 PR 合入前**: 475+ 测试全部通过
- **回滚粒度**: 每个 PR 独立，revert 单个 PR 不影响其他
- **PR 1-2 之间**: 可以暂停，此时系统处于「markdown 走策略、card 走旧路径」的混合状态，功能完全正常
- **类型检查**: 策略接口提供编译期约束，新策略必须实现所有方法

---

## 6. 替代方案

### 6.1 Class 继承而非函数式策略

可以用 `abstract class ReplyStrategy` + `CardReplyStrategy extends ReplyStrategy` 的方式。**不采用**，原因：

- 项目风格偏向函数式（`createCardDraftController`、`createDraftStreamLoop` 都是工厂函数）
- 函数式策略更容易组合（装饰器模式天然契合）
- 状态更少、闭包更清晰

### 6.2 在 deliver 回调内部做 if-else 但提取子函数

只把各分支提取为 `deliverCardFinal()`、`deliverCardTool()` 等独立函数，不引入策略接口。**不采用**，原因：

- 这些子函数仍然需要共享外层大量可变状态（通过参数传递会很冗长）
- 无法解决 replyOptions 和 finalize 的耦合问题
- 新功能（dynamic reaction）仍然会往主函数堆状态

### 6.3 完全重写 inbound-handler

把整个 1600 行文件拆成多个模块。**不采用**，原因：

- 改动面过大，review 困难
- 函数前半段（鉴权、引用恢复、上下文组装）结构清晰，不需要重构
- 风险过高，与渐进重构的原则冲突

---

## 7. 未解决问题

1. **`deliverMediaAttachments` 的归属**: 当前是 deliver 回调内的嵌套函数。重构后可以作为 `ReplyStrategyContext.deliverMedia` 注入，也可以提取为独立模块函数。PR 3 再决定。

2. **ack reaction 回收逻辑**: 当前在 finally 块中，不属于回复策略的职责。保持不变，不纳入策略接口。

3. **`sendMessage` 的卡片路径**: `send-service.ts` 中 `sendMessage` 在 `messageType=card` 时会调用 `sendProactiveCardText`，这是 fallback 创建重复卡片的根因之一。本次重构通过策略内部绕过 `sendMessage` 解决，不修改 `sendMessage` 本身。长期可以考虑让 `sendMessage` 接受 `forceMarkdown` 参数。
