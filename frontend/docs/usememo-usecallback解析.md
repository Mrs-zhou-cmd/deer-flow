# React useMemo / useCallback 面试速记卡

> 结合 DeerFlow 前端项目真实代码整理，面试前过一遍即可。

---

## 一、一句话总结（面试回答模板）

> **useMemo 缓存计算值，useCallback 缓存函数引用，本质都是"依赖不变则引用不变"，服务对象是 memo 子组件、useEffect 依赖和 Context 消费者。项目里最典型的三处：Context value 必须 memo（browser-view/context.tsx）、右面板 JSX 派生缓存（chat-box.tsx）、事件处理器链的引用稳定（prompt-input.tsx / input-box.tsx）。但原始类型派生值（welcome.tsx 的 isUltra）就是反例——不过度优化，等有实际重渲染问题时再上。**

---

## 二、本质（面试第一步）

```ts
// useCallback：缓存"函数引用"
useCallback(fn, deps)      // 等价于 useMemo(() => fn, deps)

// useMemo：缓存"计算结果"
useMemo(() => compute(), deps)
```

**核心不是"性能优化"，而是引用稳定性（referential stability）：**

> `deps` 没变时返回**同一个引用**，从而让依赖它的 `memo` 子组件不重渲染、`useEffect` 不重复执行。

---

## 三、项目里 useMemo 的真实用法

### 1. Context value 必须 memo（最高频场景）

`frontend/src/components/workspace/browser-view/context.tsx`：

```tsx
const value = useMemo(
  () => ({ open, latestFrame, pushFrame, openPanel, close }),
  [open, latestFrame, pushFrame, openPanel, close],
);

return (
  <BrowserViewContext.Provider value={value}>
    {children}
  </BrowserViewContext.Provider>
);
```

> **面试点**：这是 `useMemo` 最重要的用途。如果不 memo，每次 Provider 重渲染都会生成**新的对象引用**，所有 `useContext` 的消费者（哪怕和状态无关）全部被迫重渲染。
> 同款写法：`chain-of-thought.tsx`、`prompt-input.tsx`（`attachments` / `controller` context value）。

### 2. 缓存昂贵的派生数据 / JSX

`frontend/src/components/workspace/chats/chat-box.tsx` —— 右面板内容整块 JSX 的构建：

```tsx
const rightPanelContent = useMemo(() => {
  if (renderedRightPanel === "browser") {
    return <BrowserViewPanel threadId={threadId} className="size-full" />;
  }
  // ... 一大段分支 JSX
  return null;
}, [renderedRightPanel, selectedArtifact, threadId, artifacts, setArtifactsOpen]);
```

字符串处理这种"每次渲染都重算浪费"的：

```tsx
const resizableIdBase = useMemo(() => {
  return pathname.replace(/[^a-zA-Z0-9_-]+/g, "-").replace(/^-+|-+$/g, "");
}, [pathname]);
```

纯派生布尔值（`sidecar-panel.tsx` 的 `selectedModel` 查找、`welcome.tsx` 的 `colors` 数组同理）：

```tsx
const artifactPanelOpen = useMemo(() => {
  if (sidecarOpen) return false;
  if (env.NEXT_PUBLIC_STATIC_WEBSITE_ONLY === "true") {
    return artifactsOpen && artifacts?.length > 0;
  }
  return artifactsOpen;
}, [artifactsOpen, artifacts, sidecarOpen]);
```

> **面试点**：`useMemo` 用在**派生值**上——即从 state/props 计算出来的东西，配合纯函数思维。它不替代懒加载（`dynamic()`）等业务层面的加载优化。

---

## 四、项目里 useCallback 的真实用法

### 1. 给子组件 / API 传稳定的事件处理器

`frontend/src/components/workspace/browser-view/context.tsx`：

```tsx
const pushFrame = useCallback((frame: BrowserViewFrame) => {
  setLatestFrame((prev) => { ... });
}, []);   // 只依赖 setState，永远稳定

const openPanel = useCallback(() => setOpen(true), []);
const close = useCallback(() => setOpen(false), []);
```

`frontend/src/components/ai-elements/prompt-input.tsx` —— 一整套处理函数：

```tsx
const clearInput = useCallback(() => setTextInput(""), []);
const add = useCallback((files: File[] | FileList) => { ... }, []);
const remove = useCallback((id: string) => { ... }, []);
const openFileDialog = useCallback(() => { openRef.current?.(); }, []);
```

### 2. 有依赖链时，保证下游稳定

`frontend/src/components/workspace/input-box.tsx` —— 典型的回调依赖回调：

```tsx
const cancelDraftSaveTimer = useCallback(() => { ... }, []);
const invalidateDraftSaveTimer = useCallback(() => {
  draftSaveGenerationRef.current += 1;
  cancelDraftSaveTimer();
}, [cancelDraftSaveTimer]);   // 上游稳定 → 下游才稳定
```

`frontend/src/components/workspace/chats/chat-box.tsx` —— 回调依赖多个 memo 值：

```tsx
const handlePanelGroupLayoutChanged = useCallback(
  (layout: Layout) => { ... },
  [activeRightPanel, browserView, resizableIdBase, setArtifactsOpen, sidecar],
);
```

### 3. 放入 useEffect 依赖，避免重复触发

`chat-box.tsx` 把 `selectArtifact`、`deselect` 这些（来自上层 context 的稳定函数）放进 effect 依赖数组，配合 `useCallback` 保证引用稳定，effect 才不会每次渲染都跑：

```tsx
useEffect(() => {
  // ...
}, [threadId, autoSelectFirstArtifact, deselect, selectArtifact, ...]);
```

---

## 五、什么时候**不该**用（反向题必考）

项目里的反面教材 —— `frontend/src/components/workspace/welcome.tsx`：

```tsx
const isUltra = useMemo(() => mode === "ultra", [mode]);
```

这种 `===` 布尔比较**其实没必要 memo**（原始类型每次比较结果一致，`useMemo` 反而增加了 deps 比较的开销）。

**不该用的情况：**

1. **原始类型派生值**（布尔、数字、字符串）——除非传给 `memo` 组件或进 `useEffect` deps，否则没必要。
2. **每次渲染都要用的函数**（内联 `onClick` 成本极低，现代 React 不必包一层）。
3. **过早优化**：`useMemo` / `useCallback` 本身有开销（deps 浅比较 + 闭包创建），滥用反而更慢、更难读。
4. **数据来自 props/state 之外**（如 `Date.now()`、`Math.random()`、`crypto.randomUUID()`）——memo 也没用，每次渲染还是新的。

---

## 六、面试官常追问的进阶点

| 追问 | 标准回答 |
|---|---|
| 为什么 context value 一定要 useMemo？ | 否则 Provider 每次渲染生成新对象，全部消费者重渲染，`memo` 白做 |
| useCallback 依赖数组为空，闭包会过期吗？ | 会。空依赖 = 永远引用首次渲染的闭包。项目里都用 `setXxx(prev => ...)` 函数式更新规避（如 `pushFrame`、`remove`） |
| useMemo 和 useCallback 能替代 React.memo 吗？ | 不能。前者稳定**引用**，后者阻止**重渲染**，两者配合才有效：父组件 useCallback/useMemo 保持引用稳定 + 子组件 `memo()` 才跳过渲染 |
| 为什么不是每次渲染都重新创建？ | 组件函数体每次渲染都执行，默认每次都是新对象/新函数；hooks 只是在 deps 不变时**复用上一次的引用** |
| React 19 / React Compiler 之后还需要吗？ | 编译器会自动 memo，手写逐步变成"反模式"；但现在（尤其依赖 `react-resizable-panels` 等外部库传 ref/回调时）仍需手动 |

---

## 七、文件速查

| 文件 | 用途 |
|---|---|
| `frontend/src/components/workspace/browser-view/context.tsx` | Context value memo + 稳定回调（pushFrame/openPanel/close） |
| `frontend/src/components/workspace/chats/chat-box.tsx` | 派生布尔值、resizableIdBase、右面板 JSX、effect 依赖稳定 |
| `frontend/src/components/ai-elements/prompt-input.tsx` | 一整套稳定事件处理器 + context value |
| `frontend/src/components/workspace/input-box.tsx` | 回调依赖链（cancel → invalidate → schedule） |
| `frontend/src/components/workspace/welcome.tsx` | **反例**：原始类型派生值不该 memo |
| `frontend/src/components/ai-elements/chain-of-thought.tsx` | Context value memo |
| `frontend/src/components/workspace/sidecar/sidecar-panel.tsx` | `selectedModel` 派生查找 |
