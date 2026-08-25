# MessageList 组件流程

> 本文档讲 **`MessageList` 在聊天页（`chats/[thread_id]/page.tsx`）里跑起来之后的完整流程**：
> 数据怎么进来、消息怎么渲染、用户操作怎么走通。
> 以流程为主线，只挑**参与流程的关键变量**顺带解释；组件源码见
> `frontend/src/components/workspace/messages/message-list.tsx`。

## 0. 组件在页面里的角色

```
page.tsx（编排层：拿数据、组回调）
   └─ <MessageList />（展示层：拿 props 渲染 + 触发回调）
```

page.tsx 把三样东西交给 MessageList，之后页面里发生的一切都是这三样在流转：

1. **一份数据**：`thread`（消息数组 + 加载状态）
2. **一组回调**：`onRegenerateMessage` / `onEditAndRegenerateMessage` / `onBranchTurn` / `onSubmitHumanInput`
3. **一组开关**：`canRegenerate` / `canEdit` / `canBranch`（以及历史分页、token 展示相关 props）

### 关键变量速查（按作用流程归类）

> 这些是参与流程的核心状态，后面各节用到时不再重复展开。

| 变量 | 作用流程 | 一句话含义 |
|---|---|---|
| `thread.messages` | 所有流程 | 扁平消息数组，渲染的唯一数据源头 |
| `thread.isLoading` | 3、4、5、6 | 后端是否正在流式生成；流式中禁止一切重放操作 |
| `thread.error` | 3、7 | 当前回合是否出错；出错时跳过耗时结算、解锁人工输入卡片 |
| `isNewThread` | 4、5、6 | 后端是否已建线程；新线程没有可重放的内容，三个开关全关 |
| `isMock` / `NEXT_PUBLIC_STATIC_WEBSITE_ONLY` | 4、5、6、7 | 演示/静态站模式；禁用重放操作和人工输入 |
| `hasMoreHistory` | 2 | 后端是否还有更早的历史页（决定"加载更多"是否出现） |
| `isHistoryLoading` | 2 | 历史请求进行中（按钮转圈、防重复触发） |
| `loadMoreHistory` | 2 | 发起下一页历史请求的回调 |
| `tokenUsageInlineMode` | 1 | 回合底部 token 展示模式：off / per_turn / step_debug |
| `regeneratingMessageId` | 4 | 重新生成进行中的回合 id（按钮转圈、其他按钮禁用） |
| `editingMessageId` | 5 | 编辑重跑进行中的消息 id（该条进入编辑中态） |
| `branchingMessageId` | 6 | 分支请求进行中的回合 id（图标 pulse） |
| `replayActionBusy` | 4、5、6 | 上述三个 id 任一非空即 true —— 互斥锁，同一时刻只允许一个重放操作 |
| `canRegenerate` / `canEdit` / `canBranch` | 4、5、6 | page 层组好的能力开关（共同前置 + 各自附加条件） |
| `hasGoal` / `hasOpenHumanInputCard` / `branchThread.isPending` | 5、6 | canEdit / canBranch 的附加禁点 |
| `pendingHumanInputRequestIds` | 7 | 人工输入提交中（结果未回）的 request id 集合，防重复提交 |
| `answeredResponses` | 7 | 已作答的 request id → 答案映射，卡片据此变"已回答"并禁用 |
| `enableSidecarActions` / `sidecarSurface` | 8 | 选中文本工具条 / 浏览器面板同步的启用条件（主聊天页用默认值） |

---

## 1. 渲染流程：消息 → 分组 → 虚拟列表

这是核心的主流程，每来一批新消息就走一遍：

```
thread.messages（扁平数组）
   │ ① 分组
   ▼
useStableMessageGroups：把消息按"回合"归组
   │     · human 消息 + 紧跟的 assistant 消息算一个回合
   │     · 流式过程中保持分组稳定（不因 token 增长而重组、不闪烁）
   ▼
VirtualMessageList（只渲染可视区域的分组，长对话不卡）
   │ ② 按组类型分发渲染
   ▼
组类型 → 渲染分支：
   ├─ human / assistant      → MessageListItem（气泡 + 操作按钮）
   ├─ assistant:clarification→ HumanInputCard（人工输入卡片）或 MarkdownContent
   ├─ assistant:present-files→ ArtifactFileList（文件列表）
   ├─ assistant:subagent     → SubtaskCard（子任务卡片）
   └─ 其他 tool/思考组        → MessageGroup
```

**两个特殊时刻**：

- 首次进入且后端还没返回任何消息 → 渲染骨架屏（`MessageListSkeleton`）
- 每个回合底部 → 附加 per-turn token 展示（由 `tokenUsageInlineMode` 决定显示哪种）

**流程入口**：page.tsx 渲染 `<MessageList thread={thread} ... />` → MessageList 每次
`thread.messages` 变化都自动重跑上面这条链。

### 1.1 稳定分组：为什么复杂、怎么做到的

> 上图画了 `useStableMessageGroups`，它背后是 `core/messages/derived-state.ts` 里最绕的
> 一个纯函数 `deriveStableMessageGroups`。它的复杂度是**"流式渲染稳定性"**这个需求
> 带来的，拆成三层就朴素了。

**要解决的问题**：

- 流式回合里每个 chunk 到达都会产生**一批新的消息对象**。如果每帧重新分组，组对象
  引用全变 → React 把组件**卸载重建** → 闪烁、滚动位置丢失、动画重播。
- 更糟的是 assistant 消息会"变身"：先只有文本（该渲染成气泡），工具调用追加进来后
  又该归进 processing 组 → 直接重分组会让文本在气泡和思考面板之间跳来跳去
  （#3868 / #4304）。

**核心思想**一句话：**没变的部分原样返回上一帧的组对象（引用不变），只重新计算真正
变的部分** —— 差分更新，跟 React reconciliation 一个路子。

**第 1 层：消息怎么算"同一个消息"（身份判定）**

流式里同一消息的内容每帧都变，不能比内容，要比身份：

| 函数 | 规则 | 含义 |
|---|---|---|
| `messageStableKey` | tool 消息 → `tool:${tool_call_id}`；普通消息 → `message:${id}`；都没有 → null | 身份 key：工具结果回来时内容变了但 id 不变，身份仍成立 |
| `sameMessageIdentity` | 类型 + 稳定 key 相同 | 是"同一条消息" |
| `sameMessageValue` | 身份相同 + 所有字段 `Object.is` 相等 | 内容也没变 |
| `sameReusableMessage` | value 相同 + run_id 相同 + turn_duration 相同 | 可以整体复用 |

**第 2 层：组复用引擎 `stabilizeGroups`**

```
stabilizeGroups(nextGroups, previousGroups, activeGroupIndex)
   map 每组：
     index === activeGroupIndex   → 用新组  （正在流的那组每帧都变，别复用）
     canReuseMessageGroup(...)    → 用旧组  （id/type/条数相同 + 逐条 sameReusableMessage）
     否则                          → 用新组
```

它就是一个**"尽量返回旧对象"的 map**。`activeGroupIndex` 是唯一例外：流式中最后一组
是正在长大的那个，必须每次用新的。

**第 3 层：两条路径（`deriveStableMessageGroups` 本体）**

```
在流式 且 上次有组？
   ├─ 是 → 路径 A：只重算"当前回合"的尾巴
   │      ① 当前回合起点 = 最后一条可见 human 消息（从后往前扫）
   │      ② 在上一帧组里定位包含它的组
   │      ③ 前缀（更早回合）：逐条比对，完全一致 → 整段复用上一帧；
   │         不一致（历史被更新过）→ stabilizeGroups 兜底
   │      ④ 尾巴（当前回合）：重新分组（isCurrentTurnLoading: true）
   │         + stabilizeGroups，activeGroupIndex = 尾巴最后一组
   │      ⑤ 返回 [...prefix, ...stableTail]
   │
   └─ 否 → 路径 B：全量重新分组 + stabilizeGroups 整体兜底
           （activeGroupIndex = 正在加载 ? 最后一组 : -1）
```

**一帧 chunk 走一遍**（承接上面的渲染流程）：

```
上一帧：[human组, 前回合assistant组, 本回合processing组]
新 chunk 到达 → isLoading = true → 走路径 A
   ① 当前回合起点 = 最后一条 visible human → 定位到上一帧 index 0
   ② 前缀 [human组] 逐条比对没变 → 整段复用上一帧对象
   ③ 尾巴重新分当前回合（processing 组里多了一个 token）
   ④ stabilizeGroups：只有尾巴最后一组是 active → 用新的，其余能复用全用旧的
结果：除了正在流的那一组，其它组件零重渲染 —— 滚动、动画、交互全保住
```

**外层 hook 只做记忆**：`useStableMessageGroups` 用 ref 存"上一帧的组 + 上一帧的
isLoading"，喂给纯函数，再把结果存回 ref。函数本身纯，所以
`tests/unit/core/messages/derived-state.test.ts` 全是"给上一帧 + 给新消息 → 断言复用"的用例。

**理解捷径**三句话：

> 1. **身份**：工具消息认 `tool_call_id`，别的认 `id` —— 内容会变，身份不变。
> 2. **复用**：上一帧的组能原样复用就原样返回（引用稳定 = 组件不重挂载）。
> 3. **切分**：流式时只重算"最后一条 human 消息"之后的尾巴，历史整段不动。

---

## 2. 历史加载流程：往上翻 → 拉更早的消息

```
用户往上滚动，接近列表顶部
   │
   ▼
顶部的 LoadMoreHistoryIndicator 被 IntersectionObserver 观察到
（rootMargin 120px，提前触发）
   │
   ▼
节流（1200ms 内最多触发一次）
   │
   ▼
调用 page 传进来的 loadMoreHistory()
   │  （来自 useThreadStream，内部 fetch 更早的历史页）
   ▼
isHistoryLoading = true → 按钮显示"加载中..."
   │
   ▼
更早的消息追加进 thread.messages → 渲染流程重新跑，列表变长
   │
   ▼
后端没有更早数据了 → hasMoreHistory = false → 指示器消失
```

**这条流程的边界**：`hasMoreHistory` 为 false（后端没有更早数据了）或
`isHistoryLoading` 为 true（请求进行中）时，节流函数直接短路，不会再触发请求；
`isHistoryLoading` 同时驱动按钮的"加载中..."态。

---

## 3. 流式生成流程：一次完整回合的生命周期

用户在 InputBox 提交消息后，整个回合在 MessageList 里的流转：

```
用户提交 → sendMessage → 后端开始生成
   │
   ▼
thread.isLoading = true
   │ ① 计时：记录本回合开始时间（turnStartTime）
   │ ② 新消息以 token 粒度流进 thread.messages
   ▼
渲染流程实时重跑：最后一个 assistant 分组处于"流式中"态
   │     · getStreamingMessageLookup 标记哪些消息还在流
   │     · 流式中：隐藏操作按钮、显示思考/工具调用进度
   ▼
生成结束 → thread.isLoading = false
   │ ③ 结算：用"结束时刻 - 开始时刻"算出本回合耗时
   │    （仅当后端还没落盘 turn_duration 时兜底，落盘后以后端为准）
   ▼
页面不可见时 → 触发 onFinish → 发系统通知（标题 + 消息摘要）
```

**几个状态的含义**：

- `thread.isLoading`：后端正在流式生成。它是流程 3 的核心状态，也是流程 4/5/6 的公共禁点。
- `thread.error`：回合出错。出错时跳过耗时结算（避免展示虚假的慢），并会触发人工输入卡片的 pending 清理（见流程 7）。
- `turnStartTime` / `clientDurationsByGroupId`：回合耗时。开始时刻在 `isLoading` 变 true 时记录，结束（变 false）时用"结束 - 开始"算一个客户端兜底耗时；后端一旦落盘权威的 `turn_duration`，客户端值就被覆盖删除。

**并发注意**：流式期间 `canRegenerate/canEdit/canBranch` 全部为 false（`thread.isLoading`），
UI 上操作按钮被禁用，避免对正在流的回合做重放操作。

---

## 4. 重新生成流程

```
用户 hover 某条 assistant 回合 → 出现操作按钮组 → 点"重新生成"
   │
   ▼
前置检查（全过才可点）：
   ├─ 这是最后一个 assistant 回合（enableRegenerateForTurn）
   ├─ 不在流式、没有其他重放操作在进行（!replayActionBusy）
   └─ page 传的 canRegenerate = true
   │
   ▼
setRegeneratingMessageId（按钮转圈、其他按钮禁用）
   │
   ▼
onRegenerateMessage(targetId, 该回合所有 assistant 消息 id)
   │  → page 的 handleRegenerate → regenerateMessage
   │  → SDK 清掉该回合旧内容，开始新一轮流式（回到流程 3）
   ▼
新流结束 → finally 清空 regeneratingMessageId → 按钮恢复

**`replayActionBusy` 是什么**：重新生成 / 编辑 / 分支三个操作共用的互斥锁 ——
只要 `regeneratingMessageId` / `editingMessageId` / `branchingMessageId` 任一非空，
其余两个操作的按钮全部禁用，保证同一时刻只有一个"重放"类请求在飞。
```

---

## 5. 编辑重跑流程

```
用户 hover 最新一条 human 回合 → 点"编辑" → 输入替换文本
   │
   ▼
前置检查：
   ├─ 是"最新可编辑"的 human 回合（getLatestEditableTurn）
   ├─ page 传的 canEdit = true（含 !hasGoal、无未处理人工输入卡片等）
   └─ 没有其他重放操作在进行
   │
   ▼
setEditingMessageId（该条进入编辑中态）
   │
   ▼
onEditAndRegenerateMessage(targetId, replacementText)
   │  → page 的 handleEditAndRegenerate → editAndRegenerateMessage
   │  → 从该条 human 消息起重新生成（含其后续内容）
   ▼
完成 → finally 清空 editingMessageId

**canEdit 为什么比 canRegenerate 严**：编辑会从被编辑的 human 消息起重放后续整个回合，
所以额外要求：没有分支请求在进行（`!branchThread.isPending`）、没有活跃 Goal
（`!hasGoal`，编辑会破坏目标进度）、没有未处理的人工输入卡片
（`!hasOpenHumanInputCard`，否则重放会把卡片请求也冲掉）。
```

---

## 6. 分支流程

```
用户 hover 某条 assistant 回合 → 点"分支"（GitBranchPlus 图标）
   │
   ▼
前置检查：
   ├─ 该回合可分支（branchableAssistantGroupIds：非流式、有 id）
   ├─ page 传的 canBranch = true（含 !branchThread.isPending）
   └─ 不在流式、没有其他重放操作在进行
   │
   ▼
setBranchingMessageId（图标 pulse）
   │
   ▼
onBranchTurn(targetId, 该回合 assistant 消息 id 列表)
   │  → page 的 handleBranchTurn：
   │     branchThread.mutateAsync → 后端创建新线程
   │     成功 → toast "已创建分叉对话" + router.push 跳到新线程页
   │     失败 → toast 错误信息
   ▼
finally 清空 branchingMessageId
```

**`branchThread.isPending` 为什么是禁点**：分支请求本身在 page 层由
`branchThread.mutateAsync` 发起（向后端 POST 创建新线程），请求进行中 UI 必须等它
结束才能再次分支或做其他重放操作，否则会基于不一致的线程状态发请求。

**注意**：新线程页面会重新 mount 一个 MessageList（走流程 1 重新渲染），
当前页面随之卸载。

---

## 7. 人工输入卡片流程

某些回合（如 ask_clarification）会渲染一张输入卡片，等待用户作答：

```
渲染：检测到 assistant:clarification 组 → 解析出 HumanInputRequest → 渲染卡片
   │
   ▼
用户填表提交
   │
   ▼
① 本地标记 pending（卡片变"发送中..."，防重复提交）
   │
   ▼
② onSubmitHumanInput(request, response)
   │  → page 的 handleSubmitHumanInput → sendMessage
   │  （消息带 hide_from_ui，不会出现在正常消息流里）
   │  → 返回是否发送成功（sent）
   ▼
③ 结果处理：
   ├─ 成功 → 该 request_id 进入 answeredResponses → 卡片变"已回答"并禁用
   ├─ 返回 false → 清除 pending，卡片可重试
   └─ 抛错 → toast 错误 + 清除 pending
   ▼
线程发生错误时 → 自动清空所有 pending（防止隐藏回复永远卡住）

**两个集合的区别**：`answeredResponses` 是"后端已确认作答"的请求（由
`deriveHumanInputThreadState` 从消息推导，卡片据此变"已回答"并禁用）；
`pendingHumanInputRequestIds` 是"前端已提交、结果未回"的请求（卡片转圈防重复）。
线程出错时清空的是 pending —— 因为那条隐藏回复可能根本没送达后端，需要让用户重试。
```

---

## 8. 两个旁路流程（不改变消息数据，但由 MessageList 驱动）

### 8.1 选中文本 → 侧边操作工具条

```
用户在 assistant 回复里选中一段文本（同一条回复内）
   │
   ▼
弹出一个浮动工具条（选中内容附近，空间不够自动翻到下方）
   │
   ▼
两个动作：
   ├─ "添加到对话" → 选中片段作为引用进入主输入框
   └─ "在侧边聊天中提问" → 打开 Sidecar 并带上引用
   │
   ▼
滚动 / 按 Esc → 工具条消失
```

### 8.2 浏览器面板同步

一句话：**当 agent 在回合里用浏览器工具上网时，右侧面板自动同步展示 agent 当前看到的网页截图**。
用户不需要手动操作 —— 只要让 agent 去浏览网页（比如"打开 example.com，看看首页有什么"），
回合里每产生一张新截图，面板就自动打开并更新，像看着 agent 的眼睛一样。

**自动同步流程**：

```
agent 执行 browser_navigate / browser_screenshot 等浏览器工具
   │  （后端在 tool 消息的 additional_kwargs.browser_view 里带上 screenshot/url/title）
   ▼
MessageList 扫描 tool 消息 → 从后往前找最后一条带 browser_view 的 tool 消息
   │
   ▼
pushBrowserFrame({ screenshot, url, title }) 推进共享的 BrowserViewProvider
   │  （context.tsx：截图变化 → 自动 setOpen(true) 打开右侧面板）
   ▼
右侧面板显示：URL 地址栏 + 网页截图 + 标题
   │  （agent 每浏览一步截图实时更新）
```

**几个实现要点**：

- **只推最后一帧**：从后往前扫 tool 消息，命中第一条带 `browser_view` 的就 return，
  不推中间过程，避免面板被中间帧刷屏。
- **`messages` 故意不进 useEffect 依赖**：token 更新会让 effect 重跑，但不希望每次都
  反向扫整个长历史找最后一帧浏览器截图，所以依赖只挂 `pushBrowserFrame` 等稳定引用。
- **新截图才自动开面板**：`BrowserViewProvider.pushFrame` 内部比较前后帧
  （截图 / URL / 标题 / action 全等 → 视为同一帧，直接复用、不重开面板）；
  仅当 `screenshot` 变化时才 `setOpen(true)`，防止面板反复弹出打断用户。

**主聊天页 vs 侧边聊天页**：只有主聊天面驱动共享浏览器面板。侧边聊天（sidecar）渲染的是
另一条 thread 的消息，如果它也 `pushBrowserFrame`，面板会用主 threadId 去解析另一条
thread 的截图 → 404 / 显示错误页面。所以 message-list.tsx 里
`sidecarSurface || !pushBrowserFrame` 时直接 return，不推。

**用户手动操作**（面板打开后）：

- 顶部工具栏的**显示器图标**（`BrowserTrigger`）→ 手动开 / 关浏览器面板；
  面板可见时图标高亮（secondary 样式），再点一次关闭。
- **Live 模式**：点面板里的 "Live" 按钮进入直播控制后，用户可以直接在截图里
  点击 / 滚动 / 敲键盘（`useBrowserStream` 把输入事件实时转发给后端浏览器），
  等于自己接管了 agent 的浏览器；再点一次退出直播。
- **非 Live 导航**：在 URL 地址栏输入网址回车 → `navigateBrowser(threadId, url)`
  让后端导航，并把新截图 `pushFrame` 回面板。
- 左上角的返回 / 前进按钮仅 Live 模式可用。

**相关代码位置**：

| 文件 | 角色 |
|---|---|
| `components/workspace/messages/message-list.tsx` | 同步驱动点：useEffect 反向扫 tool 消息 → `pushBrowserFrame`；`sidecarSurface` 时不推 |
| `components/workspace/browser-view/context.tsx` | `BrowserViewProvider`：`pushFrame` 存最新帧、新截图自动开面板、open/close 状态 |
| `components/workspace/browser-view/browser-view-panel.tsx` | 右侧面板 UI：截图展示、URL 地址栏、返回/前进、Live 模式、点击/滚动坐标转发 |
| `components/workspace/browser-view/browser-trigger.tsx` | 顶部栏显示器按钮，手动开 / 关面板 |
| `components/workspace/browser-view/use-browser-stream.ts` | Live 模式：实时画面 + 输入事件转发 |
| `components/workspace/browser-view/api.ts` | 非 Live 导航 `navigateBrowser` |
| `components/workspace/messages/message-group.tsx` | 链式思考里渲染单个 `browser_*` 工具步时也会 `pushFrame` + `openPanel` |

---

## 9. 流程总览图

```
                     ┌────────────────────────────────────────┐
                     │            page.tsx（编排层）           │
                     │  thread · 回调 · 开关  →  props 下发      │
                     └───────────────────┬────────────────────┘
                                         ▼
              ┌──────────────────────────────────────────────┐
              │               MessageList                    │
              │  渲染流程 ← 消息分组 → 虚拟列表 → 按类型渲染     │
              │  ┌──────────────┬────────────────┐           │
              │  │ 数据流入      │ 用户操作流出     │           │
              │  │ 历史加载 ②    │ 重新生成 ④     │           │
              │  │ 流式回合 ③    │ 编辑重跑 ⑤     │           │
              │  │              │ 分支 ⑥         │           │
              │  │              │ 人工输入 ⑦     │           │
              │  └──────────────┴────────────────┘           │
              └──────────────────────────────────────────────┘
```

数据永远单向流动：**hooks → page → MessageList → 用户操作 → 回调 → hooks → 新数据回流**。

## 10. 相关文件

| 文件 | 在流程中的角色 |
|---|---|
| `components/workspace/messages/message-list.tsx` | 流程实现：分组、虚拟列表、操作按钮、指示器 |
| `components/workspace/messages/virtual-message-list.tsx` | 渲染流程中的虚拟滚动层 |
| `components/workspace/messages/message-list-item.tsx` | 单条消息气泡 + 编辑入口 |
| `components/workspace/messages/message-group.tsx` | 工具/思考类分组渲染 |
| `components/workspace/messages/human-input-card.tsx` | 人工输入卡片 |
| `components/workspace/messages/subtask-card.tsx` | 子任务卡片 |
| `core/threads/hooks` | 数据源头：`useThreadStream`（消息、历史、发送） |
| `core/messages/derived-state.ts` | 分组 / token 状态的推导逻辑 |
| `components/workspace/browser-view/context.tsx` | 8.2 浏览器面板：`BrowserViewProvider`（pushFrame / 自动开面板 / open 状态） |
| `components/workspace/browser-view/browser-view-panel.tsx` | 8.2 浏览器面板：右侧面板 UI（截图、地址栏、Live 直播控制） |
| `components/workspace/browser-view/browser-trigger.tsx` | 8.2 浏览器面板：顶部栏显示器按钮 |
| `components/workspace/browser-view/use-browser-stream.ts` | 8.2 浏览器面板：Live 模式实时画面 + 输入转发 |
