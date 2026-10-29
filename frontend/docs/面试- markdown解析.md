# 面试问答：Markdown 解析经验

> 场景：面试官问 "你有做过 markdown 解析吗？"
> 素材来源：DeerFlow 前端 `frontend/src/core/streamdown/` 渲染链路（技术细节见 `frontend/docs/markdown-rendering.md`）。

## 回答策略

**不要只说"我引入过 react-markdown"**，而是把"用过 markdown 库"升级成：

> 在 **不可信输入 + 流式渲染 + 必须稳定** 的约束下，从预处理、sanitize、渲染管线到性能兜底，完整设计过一条 markdown 渲染链路。

---

## 一、30 秒电梯版（保底回答）

> 做过。我在一个 AI 聊天应用的前端里负责整个 markdown 渲染链路。它不是简单的"引入 react-markdown 渲染一下"，而是围绕 **流式输出** 这个场景做了三件事：
>
> - **安全**：LLM 输出不可信，做了 sanitize 白名单和链接协议校验；
> - **稳定性**：深度嵌套的 markdown 会栈溢出，做了预处理兜底；
> - **性能**：流式期间用轻量渲染，内容稳定后才加载 Shiki 高亮。
>
> 另外还处理了数学公式归一化和渲染崩溃回退。

---

## 二、2 分钟详细版（STAR 结构）

### S —— 背景

项目是 LangGraph 的 AI 聊天应用，**模型是逐字/逐词流式返回 markdown 的**，前端要实时渲染，且模型输出**完全不可信**（可能带幻觉 HTML、恶意链接、病态嵌套）。

### T —— 挑战

1. **不完整 markdown**：流式过程中"半截代码块""孤立的列表符号"随时出现，普通渲染器会闪烁或渲染错误；
2. **XSS 风险**：模型可能输出 `<script>`、`javascript:` 链接、伪造 UI 的 HTML；
3. **崩溃风险**：marked 的 tokenizer 对嵌套引用/列表是递归解析的，深度到几千层直接栈溢出，能把整个聊天路由炸成错误页（真实踩过，issue #3393）；
4. **性能**：每个 chunk 都全量 re-parse + 高亮，代码块上 Shiki 每次都跑会卡。

### A —— 方案（按层拆解）

**① 预处理层**（手动管线，进渲染器之前先过一遍）：

- `capMarkdownNesting`：把引用嵌套限到 100 层、列表缩进限到 200 列，**代码围栏内不处理**（那是合法内容）；
- `normalizeLatexMathDelimiters`：LLM 常输出 `\(...\)`、`\[...\]`，转成 remark-math 认识的 `$...$` / `$$...$$`；多行 `$$` 块压成单行，避免分块器把起止符拆散；
- `stripLeakedSystemTags`：剥掉后端泄漏的 `<memory>` 等内部标签。

**② 渲染管线**（remark + rehype 链，顺序敏感）：

- `remark-gfm`（关掉 single-tilde，LLM 爱误用 `~~`）+ `remark-math`；
- rehype 链是 `rehype-raw → rehype-sanitize（自定义 schema）→ rehype-katex`。**关键细节**：streamdown 只要传入自定义 rehypePlugins 就会丢掉内置 sanitize，必须自己补上；sanitize 必须在 katex **之前**，否则 KaTeX 输出的 class 会被剥掉，公式渲染不出来；
- 消息正文刻意**去掉 `rehype-raw`**，让模型幻觉的 HTML 标签保持纯文本，这是第二道防线。

**③ 安全组件**：

- `isSafeHref` 协议白名单：只放 `http/https/mailto/tel` 和相对路径，拦 `javascript:`、`data:`、协议相对 URL；
- 对抗 DOM-clobber：sanitize 会给 `id` 加 `user-content-` 前缀，做了个 rehype 插件把双重前缀归一化、把脚注/锚点链接对齐。

**④ 流式渲染**：

- `useDeferredValue` + 定时提交的平滑展示（50ms 一批），尊重 `prefers-reduced-motion`；
- **分级渲染代码块**：流式期间用廉价的 `<pre>/<code>`，块稳定后才动态 `import()` Shiki，一次 `codeToHtml` 出 light/dark 双主题（CSS 变量切主题，不渲染两棵树），带 LRU 缓存；
- 渲染崩溃用 Error Boundary 兜底：整条消息回退为纯文本而不是炸掉路由。

### R —— 结果

渲染不再崩、公式正常显示、无 XSS 注入面；首屏不打包 Shiki/CodeMirror；流式体验平滑。整个链路配套了完整的单元测试（`preprocess.test.ts`、`mermaid.test.ts` 等）。

---

## 三、加分点：主动抛出权衡

讲完方案后补一句，会显得你真的思考过：

> 有几个明显的 trade-off：
> - `$` 行内公式开启后，模型输出里的美元金额会被误判成公式起点，所以 sanitize schema 和 math 归一化都围绕"**宁可显示成普通文本、不可误渲染/不可执行**"的优先级设计；
> - 平滑展示和流式分块解析本质上是"**每帧重渲染 vs 缓存**"的取舍，用 `useDeferredValue` + 稳定后降级渲染来平衡。

---

## 四、常见追问与应答要点

| 追问 | 应答要点 |
| --- | --- |
| 为什么不用 react-markdown？ | 它每次内容变化全量重 parse，流式场景开销大；streamdown 基于 marked 分块 + 支持 `parseIncompleteMarkdown`，能处理半截语法 |
| sanitize 为什么不用默认的？ | 默认 schema 会剥掉 math class（`math-inline` / `math-display`）导致公式不渲染，所以基于 GitHub 默认扩展；但刻意不放 `style` / `iframe` |
| 深度嵌套为什么崩？ | marked 的 blockquote/list tokenizer 是递归的，2000 层栈溢出；所以预处理先 cap，而不是指望渲染器扛 |
| dangerouslySetInnerHTML 怎么防？ | 全仓库只有 Shiki 输出这一个边界用，且 Shiki 产物是我们信任的（代码转义过），模型内容本身永远不进 innerHTML |
| 流式性能怎么量化？ | 首屏资产里没有 Shiki/CodeMirror chunk（动态 import 生效）；高亮有 LRU 缓存；流式期间渲染的是廉价 DOM |

---

## 五、可选延伸：秀一手

> 另外我们还做了 mermaid 语法的定向归一化 —— 模型输出带标签的虚线箭头 `-. "label" ->` 是 mermaid 不认识的写法，我写了个行级转换器在进渲染器前修掉，而且是围栏感知的，不会动代码块里的内容。

---

## 六、技术支撑文件（回答时可按需引用）

| 文件 | 用途 |
| --- | --- |
| `frontend/docs/markdown-rendering.md` | 渲染链路的完整技术文档 |
| `frontend/src/core/streamdown/preprocess.ts` | 预处理：防栈溢出、LaTeX 归一化、剥系统标签 |
| `frontend/src/core/streamdown/plugins.ts` | 插件预设：sanitize schema、math、rehype 链 |
| `frontend/src/core/streamdown/safe-children.ts` | 预处理接入层（hooks） |
| `frontend/src/components/workspace/messages/markdown-content.tsx` | 流式渲染入口 |
| `frontend/src/components/ai-elements/streamdown.tsx` | 崩溃兜底 + 剥内部标签 |
| `frontend/src/components/ai-elements/code-block.tsx` / `shiki-highlight.ts` | Shiki 分级渲染 |
| `frontend/src/components/workspace/messages/markdown-link.tsx` | 链接协议白名单 |
