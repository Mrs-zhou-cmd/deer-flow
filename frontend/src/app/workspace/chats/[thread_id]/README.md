# page.tsx 阅读笔记

> 本文档是理解 `frontend/src/app/workspace/chats/[thread_id]/page.tsx` 的入门笔记。
> 核心结论一句话：**这个组件不是业务逻辑的所在地，而是一层"编排层（orchestration）"** ——
> 它把数据 hooks 和操作回调收集起来，再分发给底下的展示组件。

## 0. 路由背景

```
frontend/src/app/workspace/chats/
├── [thread_id]/
│   ├── layout.tsx
│   └── page.tsx        ← 本文档的主角
```

- `[thread_id]` 是 Next.js App Router 的**动态路由段**，匹配 URL 中对应位置的任意一段值。
- `/workspace/chats/abc-123` → `thread_id = "abc-123"`；`/workspace/chats/new` → `thread_id = "new"`。
- 参数值通过 `useParams()` 读取，见 `frontend/src/components/workspace/chats/use-thread-chat.ts`：

  ```ts
  const { thread_id: threadIdFromPath } = useParams<{ thread_id: string }>();
  ```

- 特殊值 `"new"` 表示"新建对话"占位：前端先生成临时 `uuid()`，等后端创建线程后，
  由 `onStart` 用原生 `history.replaceState` 把 URL 替换成真实 thread id。

## 1. 组件定位

这个 `page.tsx` 只做三件事：

1. **从 hooks 拿数据**（文件前 ~60 行的所有 `use*` 调用）
2. **把数据整理成回调**（`handle*` 函数，文件中间部分）
3. **把数据和回调塞进 JSX**（最后的 return 部分）

它几乎不含业务逻辑 —— 消息怎么流、历史怎么加载、编辑重跑怎么发，
全在 `@/core/threads/hooks` 与 `@/components/workspace/*` 里。

## 2. 阅读主线：三条线

### 线 A：数据从哪来（顶部 hooks 区）

按依赖关系排序，核心链路是：

```ts
// ① URL 参数 → threadId（动态段）
const { threadId, setThreadId, isNewThread, setIsNewThread, isMock } = useThreadChat();

// ② 线程本身：所有聊天数据 + 操作入口（最重要的 hook）
const { thread, sendMessage, regenerateMessage, editAndRegenerateMessage,
        isUploading, isHistoryLoading, hasMoreHistory, loadMoreHistory } = useThreadStream({
  threadId: isNewThread ? undefined : threadId,   // 关键：新线程时传 undefined，避免拉历史
  ...
});

// ③ 辅助数据：token 用量、线程元信息、分支、goal
const threadTokenUsage = useThreadTokenUsage(isNewThread || isMock ? undefined : threadId, ...);
const threadMetadata  = useThreadMetadata(threadId, ...);
const branchThread    = useBranchThread();
const { activeGoal, hasGoal, setLocalGoal } = useActiveGoal(threadId, thread.values.goal);
```

### 线 B：状态流转（两个布尔量）

这个组件最微妙的地方，注释已写清：

- `isNewThread`：**后端是否已创建该线程**（决定要不要 fetch 历史，见 issue #2746）
- `isWelcomeMode`：**视觉上是否居中欢迎布局**（决定动画和排版）

流转关系：

```
/new 页面
  │  用户点提交 → onSend: setIsWelcomeMode(false)     ← UI 立刻动画到聊天布局
  │  后端创建成功 → onStart: setThreadId(真实id); setIsNewThread(false)
  │                history.replaceState(...)          ← 原生 API 换 URL，防止组件重挂载
  ▼
普通聊天页（isNewThread=false, isWelcomeMode=false）
```

注意 `onSend` **不能**翻转 `isNewThread` —— LangGraph SDK 一旦拿到 thread id 就会
急切地 fetch `/history`，假定线程在后端已存在（issue #2746）。

### 线 C：用户操作去哪（中间的回调区）

| 回调 | 触发点 | 去向 |
|---|---|---|
| `handleSubmit` | InputBox 提交 | `sendMessage` |
| `handleRegenerate` | 消息上的"重新生成" | `regenerateMessage` |
| `handleEditAndRegenerate` | 消息上的"编辑重跑" | `editAndRegenerateMessage` |
| `handleBranchTurn` | "从此处分支" | `branchThread` + `router.push` 到新线程 |
| `handleSubmitHumanInput` | 人工输入卡片 | `sendMessage`（带 `hide_from_ui`） |
| `handleStop` | 停止生成 | `thread.stop()` |

## 3. JSX 结构（外层 → 内层）

```
ThreadContext.Provider          ① 把 thread 共享给消息列表等后代
└─ SidecarProvider              ② 侧边栏对话的上下文
   └─ ChatBox                   ③ 整体布局容器（侧边栏等）
      └─ div（flex 容器）
         ├─ header              顶栏：ThreadTitle + 右侧工具按钮组
         │   （ScheduledTasks / TokenUsage / Sidecar / Browser / Export / Artifact）
         └─ main
            ├─ 消息区：MessageList（滚动、历史加载、重生成/编辑/分支入口）
            └─ 底部输入区：InputBox + （GoalStatus / TodoList 卡片）
```

判断技巧：组件名（`ThreadTitle`、`TokenUsageIndicator`、`MessageList`、`InputBox`…）
全是**展示型**的，只接收 props，不关心数据从哪来。
**想改 UI 去这些子组件里改，不要在这个 page 里堆 JSX。**

## 4. 实操建议（循序渐进）

1. **练习 1（只改 page.tsx，不动逻辑）**：在 header 按钮组里加一个自定义按钮
   （如 `onClick={() => toast("hello")}`），熟悉"在正确的位置加 JSX"。
2. **练习 2（只改状态）**：改 `isWelcomeMode` 的初始值或动画 class
   （`-translate-y-[calc(50vh-48px)]` 那几行），观察欢迎态 ↔ 聊天态的切换。
3. **练习 3（追进 hook）**：读 `useThreadStream` 源码（`@/core/threads/hooks`），
   找到 `sendMessage` 实现，理解它如何处理 `onSend` / `onStart` 回调 —— 数据流的关键一步。

## 5. 调试小抄

- 确认当前 URL 参数 → 看 `useThreadChat` 里的 `useParams()`
- 确认后端有没有这个线程 → 看 `isNewThread`
- 确认消息为什么没渲染 → 看 `thread.messages` 和 `isHiddenFromUIMessage`
- 确认为什么页面跳到 `/chats/new` → 看中间那个 `router.replace` 的 effect
  （条件苛刻：无消息 + 无元数据 + 历史加载完，才会跳）

## 6. 相关文件

| 文件 | 作用 |
|---|---|
| `src/app/workspace/chats/[thread_id]/layout.tsx` | 该路由的布局 |
| `src/components/workspace/chats/use-thread-chat.ts` | URL 参数 → threadId / isNewThread / isMock |
| `src/core/threads/hooks` | `useThreadStream` 等核心数据 hook |
| `src/core/threads/utils.ts` | `pathOfThread` 路径生成（agent / 普通线程分流） |
| `src/app/workspace/agents/[agent_name]/chats/[thread_id]/page.tsx` | 同构的 agent 聊天页（双层动态段） |
