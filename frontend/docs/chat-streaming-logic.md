# 聊天流式逻辑（Chat Streaming Logic）入门指南

> 适用对象：刚接手 DeerFlow 前端的开发者。
> 本文只讲一件事：**一条用户消息发出后，如何变成屏幕上流式出现的 AI 回复**。
> 相关代码全部在 `frontend/src/`，阅读本文前建议先通读 `frontend/AGENTS.md`。

---

## 1. 一分钟看懂全链路

```
用户输入
  │
  ▼
InputBox (components/workspace/input-box.tsx)
  │  1. 立即显示"乐观消息"(optimistic) → 界面不等待网络
  ▼
sendMessage (core/threads/hooks.ts)
  │  2. 上传附件（如有）
  │  3. thread.submit(...)  →  LangGraph SDK 发起 SSE 流
  ▼
useStream (SDK 的 React hook, 在 useThreadStream 内部)
  │  4. 流式事件回调：onCreated / onUpdateEvent / onCustomEvent / onError / onFinish
  ▼
消息三路合并 (mergeMessages)
  │  5. 历史(useThreadHistory) + 实时流(thread.messages) + 乐观(optimistic) → 去重排序
  ▼
渲染节流 (useCoalescedStreamMessages)
  │  6. 同一 macrotask 内的更新合并，最多每 80ms 通知一次 React
  ▼
MessageList (components/workspace/messages/message-list.tsx)
      7. 渲染消息、工具调用步骤、artifacts、todos
```

核心原则只有一句话：**`core/threads/hooks.ts` 是状态中枢，`core/api/api-client.ts` 负责把流喂给它，`components/workspace/messages/` 只负责展示。**

---

## 2. 关键文件地图

| 文件 | 职责 |
| --- | --- |
| `core/threads/hooks.ts` | **大脑**。`useThreadStream` 是核心 hook：发送、流式回调、乐观消息、三路合并、停止、regenerate。约 1700 行，是本指南的主角 |
| `core/threads/types.ts` | 类型定义：`AgentThreadState`、`RunMessage` |
| `core/threads/api.ts` | 单发 REST 调用（branch、token usage、metadata patch） |
| `core/threads/token-usage.ts` | token 用量统计 hook |
| `core/api/api-client.ts` | **SDK 封装**。单例 `getAPIClient()`；重写 `runs.stream`/`joinStream`/`cancel`：SSE gap 恢复、terminal 状态短路、CSRF 注入、静态 demo 模式 |
| `core/api/stream-mode.ts` | 校验 LangGraph `streamMode` 白名单（`values`/`messages-tuple`/`updates`/`debug`/`tasks`/`checkpoints`/`custom`），非法 mode 直接抛错 |
| `core/api/fetcher.ts` | 普通 REST 的 `fetch` 封装 + CSRF cookie 读取 |
| `core/tasks/lifecycle.ts` / `steps.ts` | 子任务（subagent）`task_started`/`task_running` 事件 → 步骤卡片模型 |
| `core/messages/` | 消息渲染辅助（处理组、run duration、workspace-change 锚点、human-input 卡片） |
| `components/workspace/messages/` | 渲染层：`message-list.tsx`、`message-group.tsx`、`message-list-item.tsx` |

---

## 3. 核心类型（先认识数据长什么样）

### 3.1 线程状态 `AgentThreadState`

```ts
interface AgentThreadState extends Record<string, unknown> {
  title: string;
  messages: Message[];        // 当前 checkpoint 里的消息（LangChain 消息）
  artifacts?: string[];       // 产物文件路径
  todos?: Todo[];             // 待办
  goal?: GoalState | null;    // /goal 目标
}
```

### 3.2 历史消息行 `RunMessage`

```ts
interface RunMessage {
  run_id: string;      // 属于哪次运行
  seq: number;         // 线程全局单调递增序号 —— 排序和分页游标
  content: Message;    // 真正的消息内容
  created_at: string;
}
```

### 3.3 LangChain `Message`

`type: "human" | "ai" | "tool" | "system" | "remove"`，关键字段：`id`、`content`、`tool_calls`、`additional_kwargs`。流式过程中同一个消息会被反复更新（增量 chunk）。

---

## 4. 发送消息：`sendMessage`（hooks.ts）

入口在 `useThreadStream` 返回的 `sendMessage`，被聊天页面调用。

### 4.1 流程

```
sendMessage(threadId, message, extraContext, options)
  ├─ 1. sendInFlightRef 防并发：同一时刻只允许一次发送（防双击）
  ├─ 2. options.onSent?.() —— 过了守卫才调用（一次性清理，如清空引用）
  ├─ 3. 记录基线（baseline）：
  │      prevHumanMsgCountRef        ← 当前 human 消息数
  │      pendingUsageBaselineMessageIdsRef ← 当前消息身份集合（token 用量"新增"基准）
  │      localTurnOrderBaselineIdentitiesRef ← 本次提交前的身份基线（排序锚点）
  ├─ 4. 构造乐观消息（optimistic）：
  │      { type: "human", id: "opt-human-..." } 立即上屏
  │      有附件时再加一条 { type: "ai", content: "上传中…" }
  ├─ 5. 上传附件（如有）：uploadFiles(threadId, files) → 更新乐观消息的 files 状态
  ├─ 6. thread.submit({ messages: [...] }, { threadId, streamResumable: true, config, context })
  │      —— 这就是发起 SSE 流的地方（LangGraph SDK）
  └─ 7. 成功后 invalidate 侧边栏/搜索缓存
```

### 4.2 几个"为什么"（重要设计）

- **为什么要乐观消息**：网络有延迟，先本地造一条消息上屏，界面"零等待"。
- **为什么记录 human 消息数基线**：流式事件中 `messages-tuple` 可能比 `values` 先到达（AI 步骤先出现、用户消息后出现）。乐观消息必须在服务端 human 消息到达后才清除，否则会闪烁。判断条件：
  `!hasHumanOptimistic || newHumanMsgArrived`
- **`context` 是给 agent 的运行参数**（不是给 UI 的）：
  - `thinking_enabled`：非 flash 模式开启
  - `is_plan_mode`：pro/ultra
  - `subagent_enabled`：ultra
  - `reasoning_effort`：minimal/low/medium/high
- **提交路径特意不带 `streamSubgraphs`**：子任务进度通过根命名空间的 `custom` 事件到达，避免子图帧污染线程视图。

---

## 5. 流式事件：`useStream`（SDK hook）

`useStream` 是 LangGraph SDK 提供的 React hook，在 `useThreadStream` 内部调用。它管理：建流、SSE 消费、`isLoading`、重连（`reconnectOnMount: true`）、错误。

### 5.1 关键配置

```ts
const thread = useStream<AgentThreadState>({
  client: getAPIClient(isMock),   // 单例，见 §6
  assistantId: "lead_agent",
  threadId: onStreamThreadId,
  reconnectOnMount: true,          // 页面刷新/断线后自动重连运行中的 run
  fetchStateHistory: { limit: 1 },
  throttle: true,                  // 同一 macrotask 内的流事件合并为一次 React 通知
  onCreated(meta) { /* 建流成功 */ },
  onUpdateEvent(data) { /* 每个流更新（values / messages-tuple 等） */ },
  onCustomEvent(event) { /* 自定义事件 */ },
  onError(error) { /* 流错误 */ },
  onFinish(state) { /* 流结束 */ },
});
```

### 5.2 各回调的职责

| 回调 | 触发时机 | 做什么 |
| --- | --- | --- |
| `onCreated` | 流建立（拿到 `run_id`） | 记录 `threadIdRef`、往侧边栏/无限列表缓存 upsert 新线程、写 `agent_name` metadata |
| `onUpdateEvent` | 每个 stream chunk | ① 检测 summarization 中间件更新 → 计算"瞬态桥"（见 §7.3）；② 更新侧边栏标题 |
| `onCustomEvent` | 自定义事件 | `stream_replay_gap` → 全面重置 + toast 警告；`task_running`/`task_started` → 子任务步骤；`llm_retry` → toast |
| `onError` | 流出错 | 清空乐观消息、`toast.error`、invalidate 历史与 token 缓存 |
| `onFinish` | 流正常结束 | 调 `onFinish` 回调、记录 token 基线、invalidate 缓存 |

### 5.3 返回值 `thread` 对象

`useStream` 返回：`thread.messages`（实时消息）、`thread.values`（最新状态）、`thread.isLoading`、`thread.stop()` 等。

`useThreadStream` 对它做了一层包装（合并后的 `mergedThread`）：
- `values`：无可见流状态时用空值 `EMPTY_THREAD_VALUES`（避免跨线程泄漏）
- `messages`：**三路合并结果**（见 §7）
- `stop`：包了一层缓存失效逻辑

---

## 6. API 客户端：`core/api/api-client.ts`

`getAPIClient(isMock)` 返回单例 `LangGraphClient`。它做了 5 件关键的事：

### 6.1 CSRF 注入
`onRequest` 钩子：每个写请求（POST/PUT/PATCH/DELETE）从 `csrf_token` cookie 读 token，加 `X-CSRF-Token` 头。登录/换密码后 cookie 轮换也能自动生效。

### 6.2 重写 `runs.stream`
```ts
client.runs.stream = async function* (threadId, assistantId, payload) {
  // 1. sanitizeRunStreamOptions：校验 streamMode 白名单，剥掉 streamResumable
  // 2. 包一层 recoverStreamReplayGaps（见 §6.4）
  // 3. 包一层 handleInactiveRunStream（"not active on this worker" 静默吞掉）
};
```
注意保持 SDK 的**惰性 AsyncIterable** 契约：`for await` 才开始真正建流。

### 6.3 重写 `joinStream`（重连）
```ts
client.runs.joinStream = async function* (threadId, runId, options) {
  // 短路：run 已是 terminal 状态（success/error/timeout/interrupted）
  //  → 直接返回，避免阻塞在已 drain 的流桥上导致 isLoading 永远为 true
};
```

### 6.4 SSE gap 恢复（重难点，新人先了解即可）
后端会在 SSE 里发一个 id-less `gap` 控制帧（表示 replay 历史有缺口）。
SDK 会忽略未知事件，所以这里：

1. 拦截 `entry.event === "gap"`，解析 payload（`earliest_available_event_id` 等）
2. 发一个内部 `custom` 事件 `{ type: "stream_replay_gap" }` 给 hooks 层
3. 重新拉 `client.threads.getState(threadId)`（可靠状态），以 `values` 事件形式吐出
4. 用 `resume(runId, gap.latest_available_event_id)` 从缺口之后继续
5. 最多恢复 5 次（`MAX_STREAM_GAP_RECOVERIES`，含原流最多 6 次），超限抛 `StreamReplayGapError`

### 6.5 `runs.cancel` 兜底
后端 409 "is not cancellable"（run 已终态）时静默吞掉，避免 stop 竞态变成未处理 rejection。

### 6.6 静态 demo 模式
`createStaticClient()`：纯静态站点时用 mock 数据替代所有网络调用（`public/demo/threads/`）。

---

## 7. 消息合并：历史 + 实时 + 乐观（hooks.ts 的核心 memo）

这是整个文件最烧脑的部分，但新人只需要理解**输入输出**，不必深究每个分支。

### 7.1 身份（identity）—— 一切去重的依据

```ts
function messageIdentity(message: Message): string | undefined {
  if (message.tool_call_id) return `tool:${message.tool_call_id}`;
  if (message.id) return `message:${message.id}`;  // human 消息尾缀 __user 会被归一化
  return undefined;  // 没有身份的条目不参与去重
}
```

为什么要有 `__user` 归一化：后端 `DynamicContextMiddleware` 会把提交的 `HumanMessage(id=X)` 变成隐藏的 `SystemMessage(id=X)` + 真正的 `HumanMessage(id=X__user)`，UI 必须把它们当成同一条消息。

### 7.2 三路合并 `mergeMessages(history, live, optimistic)`

```
历史消息（canonical，来自 useThreadHistory）
实时流消息（live，来自 thread.messages，checkpoint 更新）
乐观消息（optimistic，本地临时）

合并规则：
1. 两边各自先按 identity 去重（保留最后一个可见副本）
2. live 中与 history 共享 identity 的 → 原位替换（checkpoint 副本更新，不移动位置）
3. live 中新的消息 → 插到下一个共享锚点之前（保证被保护的前缀消息不被移动）
4. 最后拼上 optimistic；再整体去重
```

配合 `dedupeMessagesByIdentity`：同一 identity 只保留**最后一个可见**副本（隐藏消息视为控制消息）。

### 7.3 上下文压缩（summarization）的"瞬态桥"

后端压缩上下文时会发 `RemoveMessage(ALL)` + 隐藏 summary + 保留尾部。这会瞬间清空 live 消息，但历史页还没刷新 → 中间有个空窗。

解决：`computeSummarizationTransientMessages` 把将要被移除的消息捕获进 `transientHistoryBridgeRef`，在 `resolveTransientHistoryBridge` 里把"被救援"的消息插回原位置附近，等历史页确认后再 `pruneConfirmedTransientMessages` 释放。**这是为 #3825 修的问题，理解概念即可，别轻易动。**

### 7.4 顺序修复（两个 restore 函数）

- `restoreLocalTurnMessageOrder`：本地提交后，`messages-tuple` 可能先发布 AI/tool 步骤、后发布 human 消息 → 把"本轮的可见新步骤"移到新 human 消息之后。
- `restoreReconnectedTurnMessageOrder`：刷新重连后的同款问题（重放缓冲可能丢了 human 消息）。

### 7.5 渲染节流 `useCoalescedStreamMessages`

- `throttle: true`（SDK 布尔级）只合并**同一 macrotask** 的更新。
- 这个 hook 再加一层：流式期间消息快照**最多每 80ms**（`STREAM_RENDER_COALESCE_MS`）更新一次，让 merge/group/render 管线按帧预算跑，而不是按 token 跑（#4409）。
- 实现是"前缘立即 flush + 至多一个尾缘定时器"，不是 debounce——密集流永远不会饿死渲染。
- 流结束时快照清空，下一轮流的前缘立即生效。

### 7.6 合并产物

```ts
const mergedMessages = useMemo(() => {
  // 1. 瞬态桥救援
  // 2. mergeMessages(有效历史, 节流后的实时消息, 可见乐观消息)
  // 3. 本地/重连顺序修复
}, [ ... ]);
```

`mergedThread.messages = mergedMessages` —— 这就是 MessageList 拿到的最终数据。

---

## 8. 历史消息：`useThreadHistory`

### 8.1 分页模型

- 接口：`GET /api/threads/{id}/messages/page?before_seq=<cursor>`
- 用 TanStack Query `useInfiniteQuery`；每页若干行 `RunMessage`
- `seq` 是线程全局单调递增的序号：**排序键 + 分页游标 + 去重键**（`reconcileThreadHistoryRows` 用 Map<seq, row> 合并新旧页）
- 页序是"从新到旧"加载（`next_before_seq` 往前翻），最终 `flattenThreadHistoryPages` 反转成时间正序

### 8.2 与其他数据的关系

- `useThreadHistory` 只在 `isMock` 为 false 时启用
- 历史与实时流重叠的部分以**实时流为准**（`mergeMessages` 里 live 覆盖 canonical）
- 历史页在长时间运行中会后台刷新，`reconcileThreadHistoryRows` 保证已加载的旧行不被挤掉位置

---

## 9. 停止与 regenerate

### 9.1 停止 `stopThread`

```
stopThreadAndInvalidateCaches(queryClient, () => thread.stop(), threadId)
  ├─ thread.stop()：SDK 的流停止路径
  └─ 立即 invalidate 全部相关缓存 + 1.5s 后（STOP_THREAD_FINALIZATION_REFETCH_DELAY_MS）
     再补一次 refetch —— 因为 SDK stop 可能走 abort + fire-and-forget cancel，
     后端标题定稿可能晚到
```

### 9.2 Regenerate / Edit-and-rerun

走 `submitPreparedReplay`（共用骨架）：

```
POST /api/threads/{id}/runs/regenerate/prepare
  → 返回 input + checkpoint + metadata（要重放的输入、检查点、元数据）
  → 构造 PendingPreparedReplayMask：targetRunId + supersededMessageIds
  → 立即隐藏被取代的消息（pendingSupersededRunIds / pendingSupersededMessageIds）
  → thread.submit(prepared.input, { checkpoint, metadata, ... })
  → 乐观替换消息
```

**只允许"最新一轮"可编辑**（`getLatestEditableTurn`：空闲且最后可见轮次以终态 assistant 消息结束）。

---

## 10. 新人常见任务 → 改哪里

| 想做的事 | 改哪里 |
| --- | --- |
| 改消息气泡样式 | `components/workspace/messages/message-list-item.tsx` |
| 改消息分组逻辑 | `components/workspace/messages/message-group.tsx`（含工具调用步骤） |
| 改输入框/发送按钮 | `components/workspace/input-box.tsx` |
| 改乐观消息行为 | `hooks.ts` 的 `sendMessage` |
| 改流式回调行为（如新增自定义事件） | `hooks.ts` 的 `onCustomEvent` |
| 新增一个 streamMode | `core/api/stream-mode.ts`（**同时必须改后端**，两端对齐） |
| 改 SSE gap 恢复 | `core/api/api-client.ts`（**风险高**，先搞懂再动） |
| 改历史分页 | `hooks.ts` 的 `useThreadHistory` |
| 改子任务卡片 | `components/workspace/tasks/` + `core/tasks/` |

## 11. 调试技巧

1. **看流里到底有什么**：在 `onUpdateEvent` 里临时 `console.log(data)`，能看到每个 SSE chunk 的原始形状（`values` vs `messages-tuple`）。
2. **区分三个数据源**：Chrome DevTools → Network 过滤 `stream`（SSE）、`messages/page`（历史）、`upload`（附件）。
3. **`pnpm test` 看既有行为**：`tests/unit/core/threads/` 下有 hooks 的纯逻辑测试（`mergeMessages`、`restore*Order`、`decideCoalesce` 等），改逻辑前先看测试怎么写的——**项目强制 TDD**。
4. **E2E**：`tests/e2e/chat.spec.ts` 用 `page.route()` mock 全部后端 API，验证真实页面交互（发送、流式响应、停止）。
5. 提交前必跑：`pnpm check`（lint + typecheck）、`pnpm test`。

## 12. 一句话总结

> 前端聊天 = **乐观 UI（快） + SSE 流（真） + 历史分页（稳）** 三份数据，按 `identity` 去重、按 `seq`/锚点排序，合并成一份 `messages` 喂给渲染层；中间所有"顺序怪、闪烁、丢失"的问题，都是在这条合并管线上解决的。
