# 前端 Token 用量统计逻辑入门指南

> 适用对象：刚接手 DeerFlow 前端的开发者。
> 本文只讲一件事：**界面上的 token 数字（头部统计、上下文窗口占比、每回合/每步骤明细）是怎么算出来、怎么实时跳动的**。
> 相关代码全部在 `frontend/src/`，阅读本文前建议先通读 `frontend/AGENTS.md`。

---

## 1. 一分钟看懂全链路

```
                三个数据源
  ┌───────────────────────────────┬──────────────────────────────┬────────────────────────────┐
  │ ① 后端持久化聚合               │ ② SSE 流内实时 usage_metadata │ ③ 子代理事件                 │
  │ GET /api/threads/{id}/        │ AI 消息上的 usage_metadata /  │ task_running 事件 +         │
  │     token-usage               │ additional_kwargs.usage_metadata│ 终态 ToolMessage 元数据    │
  └───────────────┬───────────────┴───────────────┬──────────────┴─────────────┬──────────────┘
                  ▼                               ▼                             ▼
        useThreadTokenUsage                useStream(thread.messages)     core/tasks/lifecycle.ts
        (TanStack Query)                   + 基线机制 → pendingMessages      + subtask-result.ts
                  │                               │
                  ▼                               ▼
        threadTokenUsageToTokenUsage        accumulateUsage(pendingMessages)
        selectContextUsage (context %)      （按 message.id 去重）
                  │                               │
                  └───────────────┬───────────────┘
                                  ▼
                  selectHeaderTokenUsage
                  = 后端累计 + 本次流式增量（仅流式期间）
                                  ▼
      ┌───────────────────────────┴───────────────────────────┐
      ▼                                                       ▼
 TokenUsageIndicator                                  MessageTokenUsageList /
 (头部胶囊 + 下拉, 四档视图预设)                          MessageTokenUsageDebugList
      └─────────────── ContextUsageBadge（token 统计关闭时接管头部位置）
```

核心原则一句话：**`core/messages/usage.ts` 是唯一的计算入口，`core/threads/hooks.ts` 负责"实时增量"的基线，后端持久化聚合 + 流内实时用量在头部合并展示，三者各司其职。**

---

## 2. 关键文件地图

| 文件 | 职责 |
| --- | --- |
| `core/messages/usage.ts` | **计算核心**。`getUsageMetadata` 提取、`accumulateUsage` 去重聚合、`normalizeTokenUsage` 校验、`selectHeaderTokenUsage` 头部合并、`formatTokenCount` 格式化 |
| `core/messages/usage-model.ts` | 视图预设（off/summary/per_turn/debug）+ debug 步骤归属解析（`token_usage_attribution`） |
| `core/threads/token-usage.ts` | 线程聚合：后端响应 → UI 形状的映射（`threadTokenUsageToTokenUsage`、`selectContextUsage`）+ placeholder 线程隔离 |
| `core/threads/hooks.ts` | **实时增量中枢**。`useThreadTokenUsage` hook、`pendingUsageBaselineMessageIdsRef` 基线、`getMessagesAfterBaseline` |
| `core/tasks/lifecycle.ts` / `subtask-result.ts` | 子代理两条用量表面：`task_running` 事件与终态 ToolMessage 元数据 |
| `components/workspace/token-usage-indicator.tsx` | 头部胶囊按钮 + 下拉面板（input/output/total + context 占比 + 预设切换） |
| `components/workspace/context-usage-badge.tsx` | token 统计关闭时接管头部位置的上下文窗口徽标（占位符常驻） |
| `components/workspace/messages/message-token-usage.tsx` | 消息组尾部的回合汇总 / debug 步骤明细 |
| `app/workspace/{chats,agents/[agent_name]/chats}/[thread_id]/page.tsx` | 接线处：把三个数据源喂给头部组件 |

---

## 3. 三个数据源

| 数据源 | 来源 | 用途 |
| --- | --- | --- |
| **后端持久化聚合** | `GET /api/threads/{id}/token-usage`（`useThreadTokenUsage` 拉取，TanStack Query 缓存） | 累计 input/output/total、按模型/调用方拆分、`context_usage` 上下文窗口占比 |
| **SSE 流内实时用量** | AI 消息上的 `usage_metadata` / `additional_kwargs.usage_metadata`（后端 PR #1218 附加，SDK 未类型化） | 运行中实时跳动的数字 |
| **子代理事件** | `task_running` 事件与终态 ToolMessage 元数据 | 子代理卡片上的用量 |

页面接线（`app/workspace/chats/[thread_id]/page.tsx`）：

```tsx
const backendTokenUsage = threadTokenUsageToTokenUsage(threadTokenUsage.data); // ① 持久化累计
const contextUsage = selectContextUsage(threadTokenUsage.data);                // ① 上下文窗口占比
// ...
{tokenUsageEnabled ? (
  <TokenUsageIndicator threadId={isNewThread ? undefined : threadId}
    backendUsage={backendTokenUsage}
    messages={thread.messages}
    pendingMessages={pendingUsageMessages}        // ② 实时增量（见 §6）
    preferences={localSettings.tokenUsage}
    onPreferencesChange={...} />
) : (
  <ContextUsageBadge contextUsage={contextUsage} />
)}
```

> `tokenUsageEnabled` 来自 `useModels()`（模型能力探测）：模型支持时显示完整统计指示器，否则只显示上下文窗口徽标。

---

## 4. 计算核心：`core/messages/usage.ts`

### 4.1 `getUsageMetadata(message)` — 提取

只认 `type === "ai"` 的消息；优先读顶层 `usage_metadata`，回退 `additional_kwargs.usage_metadata`，返回 `{ inputTokens, outputTokens, totalTokens }`（缺失字段按 0 补）。

### 4.2 `accumulateUsage(messages)` — 聚合（关键：按 message.id 去重）

```ts
for (const message of messages) {
  const usage = getUsageMetadata(message);
  if (!usage) continue;
  if (message.id) {
    if (countedMessageIds.has(message.id)) continue;  // 同一条消息只计一次
    countedMessageIds.add(message.id);
  }
  cumulative.inputTokens += usage.inputTokens;
  // ...
}
```

**为什么要去重**：UI 渲染可能把同一条 AI 消息放进多个分组（例如一条消息同时含推理内容 + 最终回答），而 `usage_metadata` 挂在 AI 消息本体上。不去重就会把同一份 token 重复累计。没有任何消息带用量时返回 `null`（而不是全 0 对象），让调用方区分"没有数据"和"用了 0 token"。

### 4.3 `normalizeTokenUsage(value)` — 校验

三个键必须都是**有限、非负**的数字（`Number.isFinite` 且 `>= 0`），任一不满足整体判为 `undefined`。它是**子代理两个用量表面的唯一共享校验**（`core/tasks/lifecycle.ts` 的 `task_running` 事件 和 `core/tasks/subtask-result.ts` 的终态 ToolMessage 元数据）——保持一个函数，防止两处口径漂移（比如一处接受了另一个字段而另一处拒绝）。

### 4.4 `selectHeaderTokenUsage({backendUsage, messages, pendingMessages})` — 头部合并

```ts
if (hasNonZeroUsage(backendUsage)) {                 // 有持久化累计
  const pendingUsage = accumulateUsage(pendingMessages); // 加上本次流的增量
  return pendingUsage ? addUsage(backendUsage, pendingUsage) : backendUsage;
}
return accumulateUsage(messages);                     // 新线程：只算当前可见消息
```

- `hasNonZeroUsage`：`null`/`undefined`/全 0 都视为"没有"，回到消息级兜底；
- `addUsage`：纯分量相加；
- **新线程场景**（后端还没有累计数据）：直接聚合当前可见消息，保证头部至少显示本次用量。

### 4.5 `formatTokenCount(count)` — 格式化

`< 10_000` 用千分位（`1,234`），以上用 `K` 缩写（`12.3K`）。

---

## 5. 线程聚合层：`core/threads/token-usage.ts` + `useThreadTokenUsage`

### 5.1 映射函数

- `threadTokenUsageToTokenUsage(response)`：后端 `ThreadTokenUsageResponse`（`total_input_tokens` / `total_output_tokens` / `total_tokens`）→ UI 的 `TokenUsage`；响应为空返回 `null`。
- `selectContextUsage(response)`：只投影 `context_usage` 块（`token_count` → `tokenCount`、`max_context_tokens`（可空）、`percentage`（可空））；`context_usage` 缺失/为空返回 `null`。

### 5.2 `useThreadTokenUsage` hook（`hooks.ts`）

```ts
export function useThreadTokenUsage(threadId, { enabled = true } = {}) {
  return useQuery<ThreadTokenUsageResponse | null>({
    queryKey: threadTokenUsageQueryKey(threadId),
    queryFn: async () => { ... fetchThreadTokenUsage(threadId) ... },
    retry: false,
    refetchOnWindowFocus: false,
    placeholderData: (previous) =>
      retainThreadTokenUsagePlaceholder(previous, threadId),  // 关键
  });
}
```

`retainThreadTokenUsagePlaceholder` 是线程隔离的关键：

```ts
export function retainThreadTokenUsagePlaceholder(previous, threadId) {
  return previous && previous.thread_id === threadId ? previous : undefined;
}
```

- **同线程 refetch 不闪烁**：run 结束后 `invalidateQueries(threadTokenUsageQueryKey(threadId))` 触发 refetch，期间旧数据继续占位显示；
- **跨线程绝不泄漏**：切到另一个线程时，上一个聊天页的用量不会带进新页面（`thread_id` 不匹配 → 占位数据丢弃，新数据到达前显示空）。

---

## 6. 实时增量：`pendingMessages` 基线机制（`hooks.ts`）

流式时后端累计接口还在请求中（或未刷新），头部要实时跳数字。做法：**发送时给当前消息集拍照（基线），运行中只把"基线之后出现的新消息"当作本次 run 的实时用量**。

```
发送时   pendingUsageBaselineMessageIdsRef = 当前所有消息的 identity 快照
运行时   pendingUsageMessages = getMessagesAfterBaseline(persistedMessages, 基线)
         // 只保留基线之后出现的"新"消息 → 本次 run 的实时用量
头部     selectHeaderTokenUsage(后端累计, pendingUsageMessages)
```

`getMessagesAfterBaseline` 的实现很直白（身份为空的消息永远保留，其余按 identity 过滤）：

```ts
function getMessagesAfterBaseline(messages, baselineMessageIds) {
  return messages.filter((message) => {
    const id = messageIdentity(message);
    return !id || !baselineMessageIds.has(id);
  });
}
```

### 6.1 基线的生命周期（精心处理的边界）

| 时机 | 行为 | 为什么 |
| --- | --- | --- |
| `sendMessage` 发送 | 重置基线 = 当前 `persistedMessages` 的 identity 集合 | 本次 run 只算新消息 |
| `submitPreparedReplay`（regenerate/edit） | 同样重置基线 | 重放也按"新消息"计 |
| **断线重连 / 页面刷新中恢复** | `useEffect` 监听 `thread.isLoading`，若基线为空就重新快照当前消息 | 重连时没有"本地发送"动作，基线为空会把历史消息全算成本次增量（重复计数） |
| `onFinish` / `onError` | 基线重置为最新消息集 | 保证下一次 run 的增量从新基线开始；`onError` 后还会 invalidate 历史与 token 缓存 |
| `stream_replay_gap` 事件 / 切换线程 | 清空基线 | 状态整体复位 |

### 6.2 与渲染节流的关系（#4409）

`useCoalescedStreamMessages` 把**渲染**节流到最多每 80ms（`STREAM_RENDER_COALESCE_MS`）一次，避免 merge/group/render 管线按 token 跑。但 token 统计消费的是**逐 chunk 的原始数组**（`persistedMessages`，经 `messagesRef` 中转），**不受渲染节流影响**——头部数字依然按 SSE chunk 实时更新，只是"上屏的 messages"被节流。

---

## 7. 视图层：三个组件

### 7.1 `TokenUsageIndicator`（`components/workspace/token-usage-indicator.tsx`）

头部胶囊按钮 + 下拉面板：

- 明细：input / output / total；
- 右侧并排显示 context 占比（复用 `formatContextUsagePercentage`，见 `components/workspace/context-usage-format.ts`）；
- 四档视图预设切换：`off` / `summary` / `per_turn` / `debug`，由 `core/messages/usage-model.ts` 的 `getTokenUsageViewPreset(preferences)` 从 `localSettings.tokenUsage`（`{ headerTotal, inlineMode }`）推导：

```
off         → headerTotal=false, inlineMode=off
summary     → headerTotal=true,  inlineMode=off
per_turn    → headerTotal=true,  inlineMode=per_turn
debug       → headerTotal=true,  inlineMode=step_debug
```

### 7.2 `ContextUsageBadge`（`components/workspace/context-usage-badge.tsx`）

`tokenUsageEnabled` 为 false 时接管头部位置。**刻意保持常驻**：`contextUsage` 不可用时渲染仪表盘占位符（不卸载、位置不跳动），数据到达后在原位置显示百分比（`formatContextUsagePercentage` 处理小数/越界/NaN）。

### 7.3 `MessageTokenUsageList` / `MessageTokenUsageDebugList`（`components/workspace/messages/message-token-usage.tsx`）

- **per_turn**：消息组尾部显示本回合汇总（该回合内 AI 消息的用量聚合）；
- **step_debug**：按步骤拆分，步骤数据来自 `core/messages/usage-model.ts` 的 `buildTokenDebugSteps(messages, t)`。

---

## 8. Debug 步骤归属：`core/messages/usage-model.ts`

debug 视图的每步归属来自后端写入的 `additional_kwargs.token_usage_attribution`（后端正本在 `backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py`）。前端只做**宽容解析**：`normalizeTokenUsageAttribution` 忽略未知字段、过滤非法 action（非法 kind 直接丢弃该 action），版本字段保持向前兼容。

步骤构建优先级：

```
1. 有 attribution（token_usage_attribution）
   → 用 actions 生成标签（todo 开始/完成/更新/移除、subagent 派发、搜索、present_files、clarification、tool）
   → actions 为空时按 kind 兜底：final_answer → "最终回答"，thinking → "思考"
   → 多个 action → 合并为一条 stepTotal，sharedAttribution=true
2. 没有 attribution → 解析 tool_calls
   → write_todos → "更新待办"
   → task / web_search / image_search / web_fetch / present_files / ask_clarification 有专用文案
   → 纯文本无工具调用 → finalAnswer；否则 → thinking
3. 一条消息对应多个 action 时合并为 stepTotal（不重复展示）
```

> 注意：精确的 `write_todos` 标签来自后端 attribution 载荷；前端 fallback 故意保持通用文案，避免与后端 `_build_todo_actions` 的算法漂移。

---

## 9. 一个完整的流式时间线

```
用户发送
  → 基线快照（pendingUsageBaselineMessageIdsRef = 现有消息 identity 集合）
  → SSE 流中 AI 消息携带 usage_metadata 到达（useStream 的 thread.messages 实时更新）
  → pendingUsageMessages = 基线后的新消息
  → 头部 = 后端累计（旧数据）+ 本次增量 ── 数字实时跳动
  → run 结束：onFinish 重置基线 + invalidateQueries(threadTokenUsageQueryKey(threadId))
  → useThreadTokenUsage refetch（placeholderData 保证同线程不闪烁）
  → 头部回落为纯后端持久化累计（已包含本次 run）
```

---

## 10. 测试覆盖（改动必跑）

| 测试文件 | 覆盖点 |
| --- | --- |
| `tests/unit/core/messages/usage.test.ts` | `accumulateUsage` 按 message.id 去重、`normalizeTokenUsage` 严格校验、`selectHeaderTokenUsage` 合并优先级、`formatTokenCount` 格式化 |
| `tests/unit/core/threads/token-usage.test.ts` | `retainThreadTokenUsagePlaceholder` 线程隔离（同线程保留 / 跨线程丢弃）、`selectContextUsage` 可空字段投影、`threadTokenUsageToTokenUsage` 映射 |
| `tests/unit/core/threads/api.test.ts` | `fetchThreadTokenUsage` 走共享 auth fetch、不可用时返回 `null` |
| `tests/unit/components/workspace/context-usage-badge.test.ts` | 占位符常驻 / 百分比渲染 |
| `tests/unit/components/workspace/context-usage-format.test.ts` | 百分比格式化（整数/小数/NaN/负值钳制） |

---

## 11. 新人常见任务 → 改哪里

| 想做的事 | 改哪里 |
| --- | --- |
| 改头部数字的合并规则（后端累计 + 实时增量） | `core/messages/usage.ts` 的 `selectHeaderTokenUsage` |
| 改"实时增量"的基线时机 | `hooks.ts` 的 `pendingUsageBaselineMessageIdsRef` 相关位置（sendMessage / onFinish / onError / 重连 effect） |
| 改视图预设（off/summary/per_turn/debug） | `core/messages/usage-model.ts` 的 `getTokenUsageViewPreset` |
| 改 debug 步骤的标签/分组 | `core/messages/usage-model.ts` 的 `buildTokenDebugSteps`（**若涉及精确 todo 标签须同步改后端 middleware**） |
| 改头部 UI/下拉 | `components/workspace/token-usage-indicator.tsx` |
| 改上下文窗口徽标 | `components/workspace/context-usage-badge.tsx` |
| 改后端响应字段映射 | `core/threads/token-usage.ts` + `core/threads/types.ts`（**两端必须对齐**） |
| 改子代理卡片用量 | `core/tasks/lifecycle.ts` / `subtask-result.ts`（共用 `normalizeTokenUsage`） |

---

## 12. 一句话总结

> 前端 token 统计 = **后端持久化累计（Query 缓存，稳） + 流内实时增量（基线拍照，快）**，在 `selectHeaderTokenUsage` 合并；所有去重（按 message.id）、校验（有限非负）、线程隔离（placeholder 校验 thread_id）都是为了回答同一个问题：**这个数字到底该不该变、变多少、切了页面会不会串数据。**
