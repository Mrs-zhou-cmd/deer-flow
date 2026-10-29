# React State 与 Props 面试速记卡

> 结合 DeerFlow 前端项目真实代码整理，面试前过一遍即可。

---

## 一、一句话总结（面试回答模板）

> **Props 是父组件给子组件的只读数据（单向向下），State 是组件内部可变更的数据（setState 唯一入口）。数据向下、事件向上；兄弟共享就状态提升，跨层共享用 Context；能派生的值不存 state，能受控就不非受控。项目里典型例子：PromptInputProvider 状态提升 + 双模式受控/非受控（prompt-input.tsx）、pushFrame 函数式更新（browser-view/context.tsx）、memo 组件依赖稳定 props（chain-of-thought.tsx）。**

---

## 二、一句话本质（面试第一步）

| | State | Props |
|---|---|---|
| 归属 | 组件**内部**私有，可变更 | 父组件**传入**，只读不可变 |
| 变更方式 | `setState`（唯一途径） | 父组件重新渲染时传新值 |
| 生命周期 | 组件卸载即销毁 | 由父组件控制 |
| 方向 | 单向数据流：父 → 子（props），state 不能跨组件直接改 |

> **核心定律：数据向下流动（props 传下去），事件向上流动（回调传上来）。**
> 项目里就是这条铁律的活例子——`browser-view/context.tsx` 里 `pushFrame`（事件向上）由子组件调用，`latestFrame`（数据向下）通过 Context/Props 供下游消费。

---

## 三、项目里 State 的真实用法

### 1. 基础 useState + 类型化

`frontend/src/app/(auth)/login/page.tsx`：

```tsx
const [email, setEmail] = useState("");
const [password, setPassword] = useState("");
const [rememberMe, setRememberMe] = useState(true);
const [isLogin, setIsLogin] = useState(true);
const [setupStatusPhase, setSetupStatusPhase] = useState<
  "checking" | "ready" | "unavailable"
>("checking");   // 用联合类型约束状态机
```

### 2. 惰性初始化（lazy initializer）——面试加分点

初始值需要**计算**时才用函数形式，只在首次渲染执行一次：

`frontend/src/app/(auth)/setup/page.tsx`（读 localStorage 偏好）：

```tsx
const [rememberMe, setRememberMe] = useState(
  () => loadRememberLoginPreference().rememberMe,
);
```

`frontend/src/components/workspace/chats/chat-box.tsx`（首次渲染时读一次，之后 effect 接管）：

```tsx
// Read once: `defaultSize` only applies on mount, the effect below owns every
// later open/close.
const [initialRightPanelSize] = useState(() =>
  rightPanelOpen ? RIGHT_PANEL_DEFAULT_SIZE : "0%",
);
```

> **面试点**：`useState(() => expensiveComputation())` 惰性初始化只跑一次；直接 `useState(expensiveComputation())` 每次渲染都算。

### 3. 函数式更新（functional update）——避免闭包过期

`frontend/src/components/workspace/browser-view/context.tsx`：

```tsx
const pushFrame = useCallback((frame: BrowserViewFrame) => {
  setLatestFrame((prev) => {       // 用 prev，不依赖外部变量
    const sameFrame = prev?.screenshot === frame.screenshot && ...;
    if (sameFrame) return prev;    // 引用没变 → 不触发重渲染（顺手优化）
    if (prev?.screenshot !== frame.screenshot) {
      setOpen(true);               // 状态联动
    }
    return frame;
  });
}, []);                            // 空依赖也安全，因为只用函数式更新
```

`frontend/src/components/ai-elements/prompt-input.tsx` 里移除附件的典型写法：

```tsx
const remove = useCallback((id: string) => {
  setAttachmentFiles((prev) => {
    const found = prev.find((f) => f.id === id);
    if (found?.url) URL.revokeObjectURL(found.url);   // 副作用放更新器里
    return prev.filter((f) => f.id !== id);
  });
}, []);
```

> **面试点**：多个 `setState` 连续调用会被**批处理**（React 18 自动批量），组件只重渲染一次。

### 4. 状态提升（Lifting State Up）

`frontend/src/components/ai-elements/prompt-input.tsx` —— `PromptInputProvider` 把输入框和附件状态**提升**到组件树上层，多个地方共享：

```tsx
export function PromptInputProvider({
  initialInput: initialTextInput = "",
  children,
}: PromptInputProviderProps) {
  const [textInput, setTextInput] = useState(initialTextInput);
  const [attachmentFiles, setAttachmentFiles] = useState<...>([]);
  // ... 通过 Context 下发，而不是层层传 props
}
```

> **面试点**：兄弟组件要共享状态 → 提升到最近公共父组件，用 props/Context 下发。项目里"全局态"（线程、设置、artifacts、sidecar、browserView）全部用 Context 承载（`messages/context.tsx`、`browser-view/context.tsx`…）。

### 5. 派生状态（derived state）——面试必考反模式

`frontend/src/components/workspace/chats/chat-box.tsx`：

```tsx
const artifactPanelOpen = useMemo(() => {
  if (sidecarOpen) return false;
  if (env.NEXT_PUBLIC_STATIC_WEBSITE_ONLY === "true") {
    return artifactsOpen && artifacts?.length > 0;
  }
  return artifactsOpen;
}, [artifactsOpen, artifacts, sidecarOpen]);
```

> **面试点**：能从 props/state **算出来**的值就不要额外存一份 state（否则会出现两处不同步的 bug）。"mirror props into state" 是经典反模式，正确做法是渲染时直接派生（如上）。

---

## 四、项目里 Props 的真实用法

### 1. 默认参数（default props）

`frontend/src/components/workspace/chats/chat-box.tsx`：

```tsx
const ChatBox: React.FC<{
  children: React.ReactNode;
  threadId: string;
  browserEnabled?: boolean;
}> = ({ children, threadId, browserEnabled = true }) => { ... }
```

`frontend/src/components/ai-elements/prompt-input.tsx`（多处）：

```tsx
export const PromptInputTextarea = ({
  onChange, onKeyDown, className,
  placeholder = "What would you like to know?",
  ...props
}: PromptInputTextareaProps) => { ... }

export const PromptInputButton = ({
  variant = "ghost", className, size, ...props
}: PromptInputButtonProps) => { ... }
```

> **面试点**：用解构默认值代替已废弃的 `defaultProps`。

### 2. 受控组件（Controlled）vs 非受控（Uncontrolled）——高频追问

`prompt-input.tsx` 的 `PromptInputTextarea` 是**双模式**的教科书：有 Provider 就受控，没有就非受控。

```tsx
const controlledProps = controller
  ? {
      value: controller.textInput.value,                          // 值由父级 state 决定
      onChange: (e) => {
        controller.textInput.setInput(e.currentTarget.value);     // 变化上报父级
        onChange?.(e);
      },
    }
  : { onChange };                                                 // 无 Provider：表单自己管

return (
  <InputGroupTextarea
    {...props}
    {...controlledProps}
  />
);
```

> **面试点**：受控 = `value` + `onChange` 都由 React 接管；非受控 = DOM 自己维护（`ref` 读取）。同一组件的双模式设计是区分"老手"的加分回答。

### 3. 复合组件（Compound Components）——props 分发模式

`prompt-input.tsx` 用"壳 + 子件"组合，子件通过 `ComponentProps<typeof ...>` 继承原生 props：

```tsx
export type PromptInputSubmitProps = ComponentProps<typeof InputGroupButton> & {
  status?: ChatStatus;
};
export const PromptInputSubmit = ({ className, variant = "default", size = "icon-sm", status, children, ...props }: PromptInputSubmitProps) => (
  <InputGroupButton ... {...props}>{children ?? Icon}</InputGroupButton>
);
```

### 4. children 作为渲染函数（Render Props）

`prompt-input.tsx`：

```tsx
export type PromptInputAttachmentsProps = Omit<
  HTMLAttributes<HTMLDivElement>, "children"
> & {
  children: (attachment: PromptInputFilePart & { id: string }) => ReactNode;
};

export function PromptInputAttachments({ children, className, ...props }) {
  return (
    <>
      {attachments.files.map((file) => (
        <Fragment key={file.id}>
          <div className="max-w-60">{children(file)}</div>
        </Fragment>
      ))}
    </>
  );
}
```

> **面试点**：把"怎么渲染每一项"的决定权交给调用方，组件只负责数据与布局——这就是 render props 的价值。

### 5. Props 稳定性 ↔ memo 组件（和前一篇速记卡联动）

`frontend/src/components/ai-elements/chain-of-thought.tsx` —— 组件用 `memo` 包裹，靠上层 `useCallback`/`useMemo` 提供稳定引用才生效：

```tsx
export const ChainOfThought = memo(({ className, ...props }) => { ... });
export const ChainOfThoughtStep = memo(({ className, ...props }) => { ... });
```

> **面试点**：`memo` 子组件只有在"父级传的 props 引用不变"时才跳过重渲染。回调 props 必须 `useCallback`、对象/数组 props 必须 `useMemo`，否则 `memo` 形同虚设。项目里 `chain-of-thought.tsx` 同时用 `useMemo` 缓存 Context value + `memo` 包子件，就是完整闭环。

### 6. Props 防抖：避免层层透传（Prop Drilling）

项目里三层以上才需要的数据（线程、设置、sidecar、browserView、artifacts）全部用 **Context**，而不是逐层传 props（见 `browser-view/context.tsx`、`sidecar/context.tsx`、`messages/context.tsx`）。

```tsx
export function useBrowserView(): BrowserViewContextValue {
  const ctx = useContext(BrowserViewContext);
  if (!ctx) throw new Error("useBrowserView must be used within a BrowserViewProvider");
  return ctx;
}
```

> **面试点**：props 适合 1~2 层、明确可见的数据流；跨层共享用 Context；跨页面全局数据用状态管理库或 Server State（项目里是 TanStack Query）。

---

## 五、面试官常追问的进阶点

| 追问 | 标准回答 |
|---|---|
| setState 是同步还是异步？ | 事件处理器/生命周期里异步**批量**更新；`setTimeout`/`Promise` 回调里同步执行（React 18 自动批处理也覆盖这些场景）。读新值要用 `setX(prev => ...)` 或 effect |
| 连续 setState 三次会渲染三次吗？ | 不会。React 18 自动批处理，同一事件里合并为一次渲染 |
| 父组件重渲染，子组件一定重渲染吗？ | 默认会（函数组件每次渲染都执行函数体）。用 `memo` + 稳定 props 可跳过 |
| 为什么不能直接改 state？（`state.x = 1`） | 不变性保证 React 能通过引用比较感知变化；直接改引用不变，不触发重渲染 |
| props 能改吗？ | 不能。想改就"状态提升"到父组件，由父组件改后传新值下来（受控模式） |
| key 的作用？ | 帮助 React 识别列表项/组件身份，用于复用与重挂载判断。项目里列表 map 都带 `key={file.id}` |
| 状态放组件里还是提升？ | 只被自己用 → 本地 state；被兄弟共享 → 提升；跨层 → Context；跨页面 → Server State / 全局 store |

---

## 六、常见坑（面试官爱问"踩过什么坑"）

1. **mirror props into state**：`const [x] = useState(props.x)` 后 props 变了 state 不更新 → 应该派生。
2. **闭包过期**：effect/回调里读旧 state → 用函数式更新或 ref（项目里 `filesRef.current = files` 就是为卸载清理避免 stale closure）。
3. **对象/数组直接放 state 里到处改**：不可变更新（`prev.concat(...)`、`prev.filter(...)`），见 `prompt-input.tsx` 的 `add`/`remove`。
4. **setState 后立刻读值**：还是旧值，要用 effect 或函数式更新。
5. **key 用数组下标**：列表增删会错乱，用稳定 id（项目里 `nanoid()` 生成 id）。

---

## 七、文件速查

| 文件 | 覆盖点 |
|---|---|
| `frontend/src/components/ai-elements/prompt-input.tsx` | 状态提升、受控/非受控双模式、render props、不可变更新、复合组件 |
| `frontend/src/components/workspace/browser-view/context.tsx` | 状态提升到 Context、函数式更新、数据向下事件向上 |
| `frontend/src/components/workspace/chats/chat-box.tsx` | 派生状态（不该存 state）、惰性初始化、默认 props |
| `frontend/src/app/(auth)/login/page.tsx` | 类型化 useState、状态机联合类型 |
| `frontend/src/app/(auth)/setup/page.tsx` | 惰性初始化（读 localStorage） |
| `frontend/src/components/ai-elements/chain-of-thought.tsx` | memo 组件 + props/Context 引用稳定性闭环 |
