# 前端核心：`frontend/src/core/api/api-client.ts` 详解

> 本文是对 `frontend/src/core/api/api-client.ts` 的逐段拆解，帮助理解这个前端最核心的通信文件。

## 一句话概括

`api-client.ts` 是**前端和后端 Gateway 之间的唯一通信桥梁**。它包装了 LangGraph 官方 SDK（`@langchain/langgraph-sdk`），在原始 SDK 之上做了一堆 DeerFlow 特有的"补丁"——CSRF 防护、流式断线恢复、静态模式替换、停止按钮的边界情况处理等等。

前端的聊天页面（`useThreadHistory`、提交按钮、SSE 流渲染）**全部**通过它暴露的 `getAPIClient()` 拿 client，不直接碰 SDK。所以它是核心。

## 结构拆解

文件可以分成 6 个部分：

### 1. CSRF 注入（`injectCsrfHeader`）

```ts
function injectCsrfHeader(_url: URL, init: RequestInit): RequestInit { ... }
```

LangGraph SDK 支持 `onRequest` 钩子，每次发请求前调用。这里做的事：**如果是会改状态的请求（POST/PUT/DELETE/PATCH），就自动把 `csrf_token` cookie 的值塞进 `X-CSRF-Token` 请求头**。

为什么要这样：后端 Gateway 有 CSRF 中间件（Double Submit Cookie 模式），缺这个头就返回 403。每次请求现读 cookie（而不是构造 client 时固化），是为了登录/登出/改密码换 cookie 后不用重建 client。

> 注意它和 `fetcher.ts` 里的逻辑是**同一份契约**（共用 `isStateChangingMethod` / `readCsrfCookie`），保证 SDK 路径和手写 fetch 路径行为一致。

### 2. 终端状态短路（`TERMINAL_RUN_STATUSES` + `shouldSkipReconnect`）

```ts
const TERMINAL_RUN_STATUSES = new Set(["success", "error", "timeout", "interrupted"]);
```

这是修复一个**具体 bug** 的代码。场景：

- 用户刷新页面 → 前端重新连接（`joinStream`）到一个已经在内存里的流桥（stream bridge）
- 如果这个 run 已经跑完（success/error/interrupted），后端的内存流桥被回收后，`joinStream` 会**永远阻塞**在空的 condition variable 上
- 结果：`isLoading` 永远为 true，提交按钮永远显示"停止"，刷新后第一条消息永远发不出去

解决：重连前先 `client.runs.get()` 查一次 run 状态，如果是终端状态就直接跳过 `joinStream`（返回空流，走 `onSuccess` 让 `isLoading` 复位）。

注释还解释了为什么 `interrupted` 也算终端：DeerFlow 里只有用户点"停止"（`RunManager.cancel()`）才写 interrupted，而人工介入暂停走的是 `Command(goto=END)`，结束状态是 `success`，所以 interrupted 没有可恢复的流。

### 3. 流式断线恢复（`recoverStreamReplayGaps`）—— 最复杂的一块

后端 Gateway 的 SSE 流在中断后会发一个 **`gap` 事件**（带 `requested_event_id` / `earliest_available_event_id` / `latest_available_event_id`）。前端发现 gap 后：

1. 发一个自定义事件 `{ type: "stream_replay_gap", ...gap }` 给 UI（SDK 不认识 gap 事件会当普通 finish 处理，所以要显式透传）
2. 调 `client.threads.getState()` 拉一次持久化的最新状态，作为 `values` 事件发出去（**让 UI 立即刷新到真实状态**）
3. 用 `resume(runId, latest_available_event_id)` 重新 join，**只从缺口之后的事件继续读**，跳过丢失的中间段

失败预算：`MAX_STREAM_GAP_RECOVERIES = 5`，超了就抛 `StreamReplayGapError`（不静默吞掉）。

配套的 `rememberReconnectRun` / `clearReconnectRun` 用 sessionStorage 记录"当前正在重连的 run"，用来区分是恢复流还是全新提交。

### 4. 停止按钮的 409 吞掉（`isRunNotCancellableError`）

```ts
client.runs.cancel = async (...) => {
  try { ... } catch (error) {
    if (isRunNotCancellableError(error)) { clearReconnectRun(...); return; }
    throw error;
  }
};
```

场景：用户点"停止"的瞬间，后端恰好把 run 标记成 `success`（SSE 流还没排干）。`cancel` 会返回 409。这里把**只有终端状态导致的 409** 吞掉（当作 no-op），因为此时停止本来就是多余操作；但"run 在另一台 worker 上跑着、无法取消"这种 409 **故意不吞**，因为那是真失败，必须让用户看到。

判定靠 `isRunConflictError`：既看 HTTP status 409，又看 message 里是否包含特定短语（`is not cancellable`），AND 语义保证区分不同分支。

### 5. 静态模式（`createStaticClient`）

当 `isStaticWebsiteOnly()`（部署为纯静态站点、无后端）时，不连真后端，而是**伪造一个 client**，所有 `threads.*` / `runs.*` 方法都换成读本地静态演示数据（`loadStaticDemoThreads` 等）。这样同一套 UI 代码在无后端环境也能跑演示。

### 6. 客户端缓存（`getAPIClient`）

```ts
const _clients = new Map<string, LangGraphClient>();
export function getAPIClient(isMock?: boolean): LangGraphClient { ... }
```

按 `"mock"` / `"default"` 缓存 client 单例，重复调用不重建。

## 为什么说它是核心

| 方面 | 说明 |
|---|---|
| **单一入口** | 所有页面代码通过 `getAPIClient()` 拿 client，SDK 被完全隔离在文件内 |
| **修的都是难复现的并发/时序 bug** | 刷新后卡死、gap 恢复、停止瞬间的 409 —— 这些是聊天类应用最容易翻车的地方，全集中在这里处理 |
| **契约与后端严格对应** | 409 的 message 短语、gap 事件结构、CSRF 规则，都是和后端 `thread_runs.py` / `worker.py` 对齐的（注释里都标了出处） |
| **静态/真机双模式** | 同一 client 接口下塞两种实现，UI 层无感知 |

## 阅读建议

如果觉得难懂，按这个顺序看更容易：

1. 先只读 `getAPIClient()` + `createCompatibleClient()` 的骨架（忽略函数体）
2. 再看 `injectCsrfHeader`（最简单）
3. 然后 `handleInactiveRunStream` + `shouldSkipReconnect`（讲的是"刷新后重连终端 run"这一个场景）
4. 最后啃 `recoverStreamReplayGaps`（需要先知道 SSE 的 gap 事件机制）

关键诀窍：这个文件里**每个函数都是为了修一个具体 bug 而存在的**，注释基本都写了触发场景。读的时候先问"这个函数防的是什么坑"，比顺着代码走容易理解得多。

## 相关文件

| 文件 | 关系 |
|---|---|
| `frontend/src/core/api/fetcher.ts` | 手写 `fetch` 路径，与 SDK 路径共享 CSRF 契约（`isStateChangingMethod` / `readCsrfCookie`） |
| `frontend/src/core/api/stream-mode.ts` | `sanitizeRunStreamOptions`：剔除后端不支持的 stream 选项（如 `streamResumable`）并校验 `streamMode` |
| `frontend/src/core/api/index.ts` | 仅 `export * from "./api-client"`，对外统一入口 |
| `frontend/src/core/threads/static-demo.ts` | 静态模式下的演示数据来源 |
| `frontend/src/core/config.ts` | `getLangGraphBaseURL` 决定 SDK 连哪个 API 地址 |
| `backend/app/gateway/routers/thread_runs.py` | 409 conflict 文案（`_cancel_conflict_detail`）的后端来源 |
