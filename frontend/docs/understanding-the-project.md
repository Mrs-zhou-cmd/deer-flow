# 如何理解 DeerFlow 前端项目

> 适用对象：刚接手 DeerFlow 前端的新开发者。
> 本文回答三个问题：**这个项目是什么？代码怎么组织的？从哪开始读？**
> 想要聊天流式逻辑的深度讲解，请看同目录的 [`chat-streaming-logic.md`](./chat-streaming-logic.md)。

---

## 1. 这是什么项目

**DeerFlow 是一个 LangGraph 驱动的 AI 超级 agent 系统**：后端跑着一个能调用工具、派生子 agent、有持久记忆的 agent，前端是一个聊天界面。

你在**前端**的工作本质上是：**把"一个 AI agent 在干活"这件事，变成用户能看懂、能操作、能干预的界面**。所以你会反复处理：流式输出、工具调用步骤、文件产物（artifacts）、待办（todos）、子任务、MCP 工具开关、自定义 agent……

### 1.1 技术栈（一句话版）

| 层 | 技术 |
| --- | --- |
| 框架 | Next.js 16（App Router）+ React 19 |
| 样式 | Tailwind CSS 4 + Shadcn UI（`components/ui/`）+ MagicUI + React Bits |
| AI 集成 | `@langchain/langgraph-sdk`（与后端 agent 通信）+ Vercel AI Elements |
| 服务端状态 | TanStack Query（React Query） |
| 其他 | CodeMirror（代码/文档预览）、React Flow（图）、Nextra（docs）、GSAP（动效） |
| 测试 | Rstest 单测（`tests/unit/`）+ Playwright E2E（`tests/e2e/`，mock 后端） |

### 1.2 前置知识清单

读代码前，你应该对下面这些"武器"有基本认知（不需要精通）：
1. **Next.js App Router**：`page.tsx` / `layout.tsx` / `route.ts`、Server/Client 组件边界
2. **TanStack Query**：`useQuery` / `useMutation` / `useInfiniteQuery` / `queryClient.invalidateQueries` —— 项目所有服务端数据都走它
3. **LangGraph SDK**：`Client`、`useStream` hook、`streamMode`（`values`/`messages-tuple`/`updates`/`custom`...）
4. **Shadcn 组件**：`components/ui/` 里全是它的生成代码

---

## 2. 服务拓扑 —— 你的代码跑在哪

整个项目由 `make dev` 一次拉起 4 个服务，**浏览器永远只访问一个入口**：

```
浏览器 ──► Nginx :2026（唯一入口）
              ├─ /                → 前端 Next.js :3000
              └─ /api/langgraph/* → Gateway :8001 的 agent runtime
              └─ /api/*           → Gateway :8001 的 REST 路由
```

对前端来说，**"后端"= Gateway 的 HTTP + SSE 接口**。你不需要懂后端实现，但必须懂接口形状（看 `core/threads/types.ts` 和 `core/api/` 就够）。

> ⚠️ 开发时访问 `http://localhost:2026`，**不是 3000**（那是裸前端，没有 nginx 的代理和重写）。

前端与后端的两种通信协议：
- **REST**：历史分页、线程列表、上传、settings、token 用量等一次性请求
- **SSE（Server-Sent Events）**：跑 run 时的流式更新（聊天、任务进度）

---

## 3. 代码分层：一份"洋葱"心智模型

`frontend/src/` 的分层是整个项目最核心的组织思想：

```
┌─────────────────────────────────────────────┐
│  app/       路由壳层（薄）：页面组装、路由、权限   │
├─────────────────────────────────────────────┤
│  components/ 渲染层（厚）：纯 UI，不碰业务逻辑    │
│    ui/          通用基础组件（Shadcn 生成）      │
│    workspace/   聊天工作区组件                  │
│    landing/     落地页组件                     │
│    ai-elements/ AI 相关 UI（prompt-input 等）  │
│    auth/  docs/  ...                         │
├─────────────────────────────────────────────┤
│  core/      业务逻辑层（大脑）：状态、API、领域模型│
│    threads/ agents/ artifacts/ skills/ tasks/ │
│    mcp/ settings/ auth/ config/ i18n/ ...     │
├─────────────────────────────────────────────┤
│  hooks/     共享 React hooks（跨领域复用）       │
│  lib/       纯工具函数（cn、格式化等）           │
└─────────────────────────────────────────────┘
```

**两条铁律**：
1. **UI 组件不直接写业务逻辑** —— 数据获取、状态变更都封装在 `core/` 的 hooks 里，组件只管消费（`const { thread, sendMessage } = useThreadStream(...)`）。
2. **`core/` 不关心样式** —— 它返回数据、状态、动作，不返回 JSX。

这条边界为什么重要：它让"界面改了"不影响"逻辑"，也让核心逻辑可以被单测直接覆盖（`tests/unit/core/...` 没有 DOM 依赖）。

---

## 4. 目录逐个速查

### 4.1 `app/` —— 路由

```
app/
├── (auth)/login, (auth)/setup ...   # 认证相关（括号 = 路由组，不影响 URL）
├── [lang]/                           # 多语言前缀
│   └── workspace/                    # ★ 主要工作区
│       ├── chats/[thread_id]/        #   聊天页（核心中的核心）
│       ├── agents/[...]/             #   自定义 agent 页
│       ├── scheduled-tasks/          #   定时任务
│       └── settings/                 #   设置页
├── api/                              # 前端自己的 API route（如 auth、token 刷新）
├── showcase/                         # 公开只读 demo
├── mock/                             # mock 演示页
├── blog/  docs/                      # 博客与文档站
└── page.tsx                          # 落地页（redirect 逻辑在这）
```

### 4.2 `components/` —— 渲染层

| 子目录 | 内容 |
| --- | --- |
| `ui/` | Shadcn 生成的基础组件（button、dialog...）—— **不要手改** |
| `workspace/` | 聊天主界面：`input-box.tsx`、`messages/`（list、group、item）、`artifacts/`、`tasks/`、侧边栏等 |
| `landing/` | 落地页区块 |
| `ai-elements/` | AI 交互元素：`prompt-input`、`inline-document`、`agent-activity` 等 |
| `auth/`、`docs/` | 对应页面组件 |

### 4.3 `core/` —— 业务逻辑（重点）

| 目录 | 职责 | 常看入口 |
| --- | --- | --- |
| `threads/` | 线程/聊天的一切 | `hooks.ts`（useThreadStream）、`types.ts` |
| `api/` | LangGraph 客户端单例、REST 封装 | `api-client.ts`、`fetcher.ts` |
| `messages/` | 消息渲染辅助 | `utils.ts`（分组、隐藏规则） |
| `agents/` | 自定义 agent 管理 | `hooks.ts` |
| `artifacts/` | 文件产物（代码/文档预览） | `hooks.ts`、`types.ts` |
| `tasks/` | 子任务/步骤卡片 | `hooks.ts`、`steps.ts` |
| `skills/` | skills 系统 | `hooks.ts` |
| `mcp/` | MCP 工具管理 | `hooks.ts` |
| `settings/` | 用户设置（模型、上下文、模式） | `types.ts`、`hooks.ts` |
| `auth/` | 认证状态 | `hooks.ts` |
| `uploads/` | 附件上传 | `index.ts` |
| `sidecar/` | 侧车线程（特殊线程类型） | `thread.ts` |
| `scheduled-tasks/` | 定时任务 | `hooks.ts` |
| `config/` | 后端 URL 配置 | `index.ts`（getBackendBaseURL） |
| `i18n/` | 国际化 | `hooks.ts`（useI18n） |
| `todos/`、`citations/`、`channels/`、`voice-input/`、`input-polish/`... | 各自领域 | — |

**观察规律**：几乎每个领域目录都有 `hooks.ts`（对外 API）+ `types.ts`（领域模型）。读一个领域，先读这两个文件。

### 4.4 其他

- `hooks/`：跨领域共享 hooks（如 `use-scroll`、`use-resize-observer`）
- `lib/`：纯工具（`utils.ts` 的 `cn()` 等）
- `styles/`：全局样式
- `server/`：Next.js 服务端代码（better-auth 会话、API 代理）
- `typings/`：全局类型补充

---

## 5. 核心业务域地图

```
                         ┌─────────────┐
                         │   auth      │  登录/注册/会话（better-auth）
                         └──────┬──────┘
                                ▼
        ┌───────────────────────────────────────────┐
        │             聊天工作区 (workspace)           │
        │                                            │
        │  threads/ ← 线程 + 流式（见流式文档）          │
        │  agents/  ← 选哪个 agent 干活                 │
        │  tasks/   ← 子任务进度卡                      │
        │  artifacts/ ← 生成的代码/文档产物              │
        │  skills/  ← 技能开关                          │
        │  mcp/     ← 外部工具开关                      │
        └───────────────────────────────────────────┘
        │
        ▼
   settings/（模型、模式 flash/pro/ultra、上下文）
        │
        ▼
   scheduled-tasks/（定时后台运行，非交互）
```

每个域的**页面入口**（在 `app/`）→ **组件**（在 `components/`）→ **逻辑**（在 `core/`）→ **后端**（REST/SSE）。新人读任何功能都按这条链路走。

---

## 6. 五条关键数据流（主线）

| # | 数据流 | 核心位置 | 说明 |
| --- | --- | --- | --- |
| 1 | **聊天流** | `core/threads/hooks.ts` | 详见 [`chat-streaming-logic.md`](./chat-streaming-logic.md) |
| 2 | **侧边栏线程列表** | `useInfiniteThreads`（hooks.ts）+ 乐观 upsert | `threads.search` 无限滚动；发送/建流时往缓存 upsert，流结束 invalidate |
| 3 | **历史分页** | `useThreadHistory`（hooks.ts） | `GET /api/threads/{id}/messages/page`，seq 游标向前翻 |
| 4 | **认证** | `server/better-auth/` + `core/auth/` | 前端页面服务端校验会话 → 未登录 redirect login；API 靠 cookie + CSRF |
| 5 | **设置/配置** | `core/settings/` | 模式（flash/pro/ultra）→ 映射成 run 的 `context` 参数 |

---

## 7. 技术选型背后的关键决策（为什么这么设计）

理解这些"为什么"，比背目录更重要：

1. **为什么用 LangGraph SDK 而不是自己写 fetch 轮询？**
   SDK 提供了 `useStream` hook：自动管理 SSE 消费、`isLoading`、页面刷新重连（`reconnectOnMount`）、`streamResumable`。项目在其上再包装一层（`api-client.ts`）补足了 SDK 的短板：SSE gap 恢复、terminal 状态短路、CSRF。**这是"站在巨人的肩膀上 + 打补丁"的模式。**

2. **为什么服务端状态全走 TanStack Query？**
   统一了缓存、失效、分页（`useInfiniteQuery`）、乐观更新（`setQueriesData`）。你会在 `hooks.ts` 里看到大量 `invalidateQueries` / `setQueriesData` —— 这是前端保持列表、详情、历史、token 用量多路缓存一致的手段。

3. **为什么有乐观消息 + 三路合并（历史/实时/乐观）？**
   因为 SSE 是"增量到达"的，且事件顺序不完全可靠（`messages-tuple` 可能先于 `values`）。为了既流畅（乐观）又正确（服务端为准）又稳定（历史兜底），设计了三份数据源合并。**这是项目里最难也最有价值的部分。**

4. **为什么默认 Server Component？**
   性能 + 首屏。只有交互组件才 `"use client"`（如 input-box、消息列表的交互部分）。

5. **为什么有 `showcase/` 和 `mock/`？**
   前者是公开只读 demo（allowlist 校验），后者是纯前端演示；还有 `core/static-mode.ts` 的纯静态模式（`createStaticClient` 用本地数据替代网络）。

6. **为什么大量使用 ref 而不是 state？**
   `hooks.ts` 里 `xxRef` 满天飞：因为 SSE 回调（onUpdateEvent 等）需要**读取最新值但不触发重渲染**，且避免回调闭包捕获旧值。**看到 ref 不要慌，那是在"绕过 React 渲染周期读即时值"。**

7. **为什么文档强调 TDD？**
   合并/去重/排序这类逻辑极其微妙（文件里全是针对 #3779、#3825、#4304、#4399、#4409 的修复注释），没有测试就是裸奔。**改 `core/` 逻辑前，先看对应单测。**

---

## 8. 代码约定（提交前必看）

- 组件默认 Server Component；需要交互才加 `"use client"`（在文件顶部）
- 路径别名 `@/*` → `src/*`（如 `@/core/threads/hooks`）
- Import 顺序有强制规范（内置 → 外部 → 内部 → 父级 → 同级）
- **生成代码别手改**：`components/ui/`（Shadcn）、`components/ai-elements/` 中来自 registry 的部分
- 提交前必须过：`pnpm check`（eslint + tsc）和 `pnpm test`
- 改了架构/命令/文档相关的东西，同步更新 `frontend/AGENTS.md` 或 README

---

## 9. 新人第一天：开发工作流

```bash
# 1. 起环境（仓库根目录，会一起拉起 backend + frontend + nginx）
make dev
#    浏览器打开 http://localhost:2026

# 2. 或只跑前端（配合已运行的 backend）
cd frontend
pnpm install
pnpm dev        # localhost:3000（裸前端，无 nginx 代理）

# 3. 日常循环
pnpm check      # lint + typecheck（改完必跑）
pnpm test       # 单测
pnpm test:e2e   # E2E（会先 build 再跑，比较慢，只在必要时跑）
```

> 测试用 `@rstest`（不是 jest/vitest）：`tests/unit/` 镜像 `src/` 目录结构；E2E 用 Playwright + `page.route()` mock 后端。

---

## 10. 学习路径：按顺序读这些代码

**第 1 步（半天）**：把环境跑起来，发一条消息，打开 Network 看三个请求：SSE（`stream`）、`messages/page`（历史）、可能的 `upload`。**先建立"看到的东西 → 网络请求"的对应感。**

**第 2 步（1 天）**：读透一份核心文档 + 一个核心文件：
1. `docs/chat-streaming-logic.md`（本项目配套文档）
2. `core/threads/hooks.ts` —— 不要求全懂，但要找到：`useThreadStream`、`sendMessage`、`mergeMessages`、`useThreadHistory` 的位置

**第 3 步（1 天）**：挑一个领域读透：推荐从 **`core/artifacts/`** 或 **`core/tasks/`** 开始（比 threads 小，模式相同）。读法：`types.ts` → `hooks.ts` → 找它在 `components/workspace/` 的使用处。

**第 4 步（持续）**：做小任务练手（按难度排序）：
1. 改落地页某个 section（`components/landing/`）
2. 改消息气泡样式（`components/workspace/messages/message-list-item.tsx`）
3. 给设置页加一个设置项（`core/settings/` + `app/[lang]/workspace/settings/`）
4. 改侧边栏线程列表的展示（`useInfiniteThreads` 消费端）
5. （进阶）给 `onCustomEvent` 加一个自定义事件的处理

**第 5 步（进阶）**：理解 SSE 层——读 `core/api/api-client.ts` 的 gap 恢复；理解合并管线的每个函数和对应 issue 注释（#3779 #3825 #4304 #4399 #4409）。

---

## 11. 常见疑问 FAQ

**Q：后端接口在哪定义？我要不要看后端代码？**
A：不用看后端代码。前端视角的"接口文档"就是 `core/threads/types.ts` + `core/api/`。接口变了，类型会跟着变（typecheck 会告诉你）。

**Q：为什么有的组件在 `app/` 里 import，有的在 `components/`？**
A：`app/` 只放"页面组装"逻辑（读参、权限、布局），真正的 UI 都在 `components/`。页面越薄越好。

**Q：`core/` 里的 hooks 和 `hooks/` 里的 hooks 什么区别？**
A：`core/` 是**领域 hook**（绑定某个业务域，如 threads）；`hooks/` 是**通用 hook**（不绑定业务，如滚动、尺寸）。

**Q：我看到很多 `opt-` 前缀的 id 是什么？**
A：乐观消息（`opt-human-...`、`opt-ai-...`），本地临时生成，服务端消息到达后会被清除。渲染时也会过滤它们（不进入"已渲染账本"）。

**Q：跑 `pnpm dev` 没数据？**
A：先确认后端起了（`make dev` 或 `cd backend && make dev`），并访问 `:2026` 而不是 `:3000`（nginx 代理）。也可以开 `showcase` 或 mock 模式体验纯前端。

**Q：改 `components/ui/` 会怎样？**
A：它来自 Shadcn registry，升级会被覆盖。要改基础组件请走 registry 配置或提 issue。

---

## 12. 一句话总结

> **DeerFlow 前端 = 一个把"AI agent 干活过程"可视化、可交互的聊天产品**。理解它只需抓住一条主线：`app/` 定义页面 → `components/` 负责长得好看 → `core/` 负责数据正确 → SSE/REST 连接后端；再把聊天流式（合并管线）这个最深的点单独啃透，就完成了 80% 的入门。
