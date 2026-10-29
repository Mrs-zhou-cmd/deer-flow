# 前端 Markdown 解析与渲染

> 本文档说明 DeerFlow 前端如何将 LLM 输出的 markdown 文本解析并渲染为 UI。
> 适用代码版本：frontend `2.1.0`。

## 总览

前端**不使用 `react-markdown`**，而是以 **`streamdown@2.5.0`**（一个专为 LLM 流式输出设计的 markdown 渲染引擎）为解析/渲染核心，配套：

- `@streamdown/code` —— Shiki 代码高亮（双主题）
- `@streamdown/mermaid /ˈmɜːmeɪd/` —— Mermaid 图渲染
- remark/rehype 生态 —— 语法扩展（GFM、数学、原始 HTML、消毒）

内部实现上，Streamdown 基于 `marked` 将内容切成可记忆化的块，再逐块走 react-markdown / remark / rehype /雷海普/ 流水线；因此"流式输出、分块解析"是其核心能力，本项目也为此做了大量防御性预处理与渲染优化。

```
markdown 文本
   │
   ├─ 预处理 (preprocess.ts) ── 防栈溢出、LaTeX 归一化、剥系统标签
   │
   ├─ <Streamdown> (ai-elements/streamdown.tsx)
   │     ├─ remarkPlugins  : remark-gfm + remark-math
   │     ├─ rehypePlugins  : rehype-raw → rehype-sanitize → rehype-katex
   │     └─ 自定义插件      : @streamdown/code + @streamdown/mermaid
   │
   └─ 组件覆盖 (components.tsx / markdown-content.tsx)
         ├─ a     → markdown-link.tsx（安全链接 + citation 解析）
         ├─ code/pre → 流式期间用廉价 <pre>/<code>，稳定后用 Shiki
         └─ 动画   → 词级淡入 (word fadeIn, 200ms)
```

## 核心文件

| 文件 | 职责 |
| --- | --- |
| `frontend/src/core/streamdown/plugins.ts` | 所有插件预设（安全链、数学、流式列表动画） |
| `frontend/src/core/streamdown/preprocess.ts` | 解析前预处理（防栈溢出、LaTeX 归一化、剥系统标签） |
| `frontend/src/core/streamdown/components.tsx` | `SafeStreamdown` / `SafeMessageResponse` 包装 |
| `frontend/src/core/streamdown/mermaid.ts` | Mermaid 语法的定向归一化 |
| `frontend/src/components/ai-elements/streamdown.tsx` | `ClipboardSafeStreamdown`：剥 `<memory>` 标签 + 渲染崩溃兜底 |
| `frontend/src/components/workspace/messages/markdown-content.tsx` | 消息正文渲染入口（平滑流式展示） |
| `frontend/src/components/workspace/messages/markdown-link.tsx` | 安全链接组件（协议白名单 + citation 解析） |
| `frontend/src/components/ai-elements/code-block.tsx` + `shiki-highlight.ts` | 代码块高亮（Shiki 双主题 + LRU 缓存） |

## 插件链设计（plugins.ts）

### 三套预设

| 预设 | rehypeRaw | 用途 |
| --- | --- | --- |
| `streamdownPlugins` | ✅ 有 | artifact 预览、记忆设置页等"可信富文本"场景 |
| `streamdownPluginsWithoutRawHtml` | ❌ 无 | 消息正文 `MarkdownContent`、reasoning/思考内容 |
| `reasoningPlugins` | ❌ 无 | = withoutRawHtml 的别名，专用于思考链内容 |

关键点：**只要传入了 `rehypePlugins`，Streamdown 就会丢弃自带的默认 rehype 链（`rehype-raw → rehype-sanitize → rehype-harden`）**，所以每个自定义链都必须重新显式加入 `rehypeSanitizeStep`，否则会静默失去消毒能力。

### remark 插件

```ts
const sharedRemarkPlugins = [
  [remarkGfm, { singleTilde: false }],          // 禁用单波浪线删除线（LLM 常误用 ~~~）
  [remarkMath, { singleDollarTextMath: true }], // 支持 $...$ 行内公式
];
```

### rehype 链（完整版，顺序敏感）

```
rehypeRaw
  → rehypeSanitizeStep（自定义 schema）
  → rehypeClobberFragments
  → [rehypeKatex, katexOptions]
```

顺序约束（与 Streamdown 内置链同构）：

- **sanitize 在 raw 之后**：原始 HTML 必须先解析成 hast 节点才能被消毒；之前只是惰性文本。
- **sanitize 在 katex 之前**：KaTeX 输出的是 class/style 密集的可信标记，放在消毒之后才不会被打掉。
- 自定义 `rehypeScopedSlug`（artifact 链）也在 sanitize 之后运行，让生成的 heading id 保留 sanitize 的 `user-content-` 前缀。

### 自定义 sanitize schema

基于 rehype-sanitize 的 GitHub `defaultSchema`，扩展：

- `tel:` href —— 镜像 Streamdown 内置 schema 与 `isSafeHref` 白名单。
- `code` 的 `className` 允许 `math-inline` / `math-display` —— remark-math 靠这两个 class 标记数学区间，rehype-katex 靠它们识别并渲染公式；被 sanitize 剥掉会导致公式不渲染。注意 `hast-util-sanitize` 每个属性只认第一个定义，所以是在原 `^language-.` 允许列表上**就地拓宽**而非追加。
- `code` 的 `metastring` —— 与 Streamdown 内置 schema 对齐。

**刻意不放开的**：`style` 属性/标签、`iframe`、任意 `className`（CSS 注入可做 UI 欺骗）。

### DOM-clobber 防护：rehypeClobberFragments

rehype-sanitize 会给 `id` 加 `user-content-` 前缀（默认 `clobberPrefix`），于是：

- GFM 脚注锚点会变成双重前缀 `user-content-user-content-fn-1`（remark-rehype 输出时已带一次前缀），本插件将其归一化回单前缀，否则脚注全部失效。
- 未加前缀的片段链接 `#foo` 翻译为 `#user-content-foo`；已带前缀的 href 与外部 URL 不动。

## 流式渲染优化（markdown-content.tsx）

消息正文入口 `MarkdownContent` 为流式输出做了多层优化：

### 平滑内容展示

- `useDeferredValue(content)` + 自定义 `useSmoothStreamingContent`：
  - 增量 ≥80 字符时才启用平滑展示；
  - 每 50ms 提交 ≥64 字符；
  - 总时长 300ms；
  - 尊重 `prefers-reduced-motion`（用户偏好减弱动画时直接全量渲染）。
- `parseIncompleteMarkdown={isLoading}`：流式期间允许解析不完整 markdown（Streamdown 原生能力）。
- `rehypeStreamingListItems`：尾部孤立的 `2.` / `-` 会被解析成空列表项，流式期间先隐藏，待子内容到达后再标记动画；中间空项保留在树中，避免有序列表计数重排。

### 代码块的分级渲染

流式期间（`isLoading || isRevealing`）：

- 使用廉价的 `StreamingPre` / `StreamingCode`（纯 `<pre>/<code>` + 语言标头）；
- 等块稳定后才激活 Shiki / Mermaid 等昂贵的"定型"渲染。

## 预处理（preprocess.ts）—— 防御 LLM 输出

| 函数 | 解决的问题 |
| --- | --- |
| `capMarkdownNesting` | marked 的 tokenizer 对嵌套引用/列表**递归**：~2000 层栈溢出会炸掉整条路由。引用嵌套限 100 层、列表缩进限 200 列（≈100 层嵌套），且**跳过代码围栏/缩进代码**，不破坏代码内容 |
| `normalizeLatexMathDelimiters` | LLM 常输出 `\(...\)` / `\[...\]`，转为 remark-math 认识的 `$...$` / `$$...$$`（跨行有状态转换，跳过代码块） |
| `compactDisplayMathBlocks` | 多行 `$$...$$` 压成单行，避免 Streamdown 的 marked 分块把首尾 `$$` 拆散（`%` 注释存在时不压缩，保留语义） |
| `stripLeakedSystemTags` | 剥掉后端泄漏的 `<memory>`、`<system-reminder>` 等内部标签（保留内部内容），否则 React 报 "unrecognized tag"；**代码围栏内不动**，围栏跟踪是 marker-aware 的（区分 `` ` `` 与 `~`、4 反引号内 3 反引号不误关） |
| `preprocessStreamdownMarkdown` | 定向归一化 mermaid 带标签的虚线箭头 `-. "label" ->` |

## 安全防线

### 链接白名单（markdown-link.tsx）

`isSafeHref` 只允许：

- 协议：`http:` / `https:` / `mailto:` / `tel:`；
- 同文档锚点 `#...`；
- 相对引用（`report.md`、`./`、`../`、`/workspace/...`，解析到 dummy HTTPS base 后走 https 分支）；
- 显式拦截 `javascript:`、`data:text/html`、`vbscript:`、`file:`、协议相对 URL `//evil.com` 及反斜杠变体 `\\evil.com`。

不安全链接渲染为带 `title` 提示的禁用样式 `<span>`（保留可见文本），不成为可点击锚点。

### 链接的三种去向

1. 文本匹配 `citation:...` → `CitationLink`（引用溯源卡片）；
2. `threadId` 存在且 href 以 `/mnt/` 开头 → 解析为 artifact 下载链接（新窗口）；
3. 其余外部链接 → 新窗口 + `noopener noreferrer`。

### 渲染崩溃兜底（ai-elements/streamdown.tsx）

`StreamdownFallbackBoundary`（Error Boundary）：

- marked 嵌套过深抛错时，整条消息回退为纯文本 `<pre>`（`whitespace-pre-wrap`）；
- 内容变化（下一个流式 chunk）时自动重置重试；
- 同时 `stripLeakedSystemTags` 在入口先剥掉内部标签。

## 代码高亮（Shiki）

`shiki-highlight.ts`：

- 动态 `import()` **懒加载** Shiki（不进首屏 bundle）；
- `codeToHtml` 一次渲染 **light/dark 双主题**（`one-light` / `one-dark-pro`，`defaultColor: "light"`），主题切换纯 CSS 变量，避免渲染两棵 HTML 树；
- LRU 缓存：上限 64 条、单块代码上限 100KB（超长直接渲染不缓存），key 含配置版本号；
- 行号通过 Shiki transformer 注入。

`code-block.tsx`：

- `dangerouslySetInnerHTML` 输出 Shiki 产物（仓库中唯一有意的此类用法，仅限 Shiki 输出边界）；
- 加载失败 / 模型输出未知语言时回退到原始 `<pre>/<code>`；
- `CodeBlockCopyButton` 复制原始代码文本（非高亮 HTML）。

## 入口分工

| 场景 | 使用的入口 | 插件链 |
| --- | --- | --- |
| 助手/人类消息正文、reasoning 步骤、子任务卡片 | `MarkdownContent` | withoutRawHtml（无 `rehype-raw`，防 LLM 幻觉 HTML） |
| Artifact markdown 预览 | `Streamdown` + `markdown-preview-plugins.ts` | 完整链 + `rehypeScopedSlug`（标题锚点） |
| 记忆设置页 | `SafeStreamdown` | 完整链（`streamdownPlugins`） |
| 思考/推理内容 | `reasoningPlugins` | withoutRawHtml |

## 与相关文档的关系

- 流式消息的数据流（thread 状态、SSE）见 `frontend/docs/chat-streaming-logic.md`；
- 前端整体目录结构见 `frontend/AGENTS.md`。
