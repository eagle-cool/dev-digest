---
title: "React Hooks 底層全拆解：那條你看不見的鏈表，決定了所有規則"
date: 2026-02-23
description: "深入 React 原始碼的 ReactFiberHooks.js，拆解 Hooks 底層的鏈表資料結構、Dispatcher 切換機制、Effect 循環鏈表，以及為什麼 Dan Abramov 系統性地否決了所有替代方案。"
tags: [deep-dive, react, hooks, internals]
---

「Hooks 不能放在 if 裡面。」

每個 React 開發者都背過這條規則。但你有沒有想過——為什麼？為什麼一個號稱「just JavaScript」的框架，要強制你遵守這種違反直覺的約束？答案不在文件裡那段「確保 Hooks 的呼叫順序一致」的官方說法，而是藏在 React 原始碼裡一條你從來沒看見的鏈表。

今天我們直接打開 `ReactFiberHooks.js`，把這條鏈表從頭到尾拆給你看。

---

## 從 Class 到 Hooks：不是改良，是換血

2019 年 React 16.8 發布 Hooks。Meta 的工程師在維護了五年、數萬個 class component 之後，總結出三個要命的問題：

**第一，邏輯共享是地獄。** 在 Hooks 之前，你要在元件之間共享有狀態的邏輯，只有兩條路：Higher-Order Components (HOC) 和 render props。兩種都會在 React DevTools 裡製造出層層套娃的「wrapper hell」。你打開 DevTools，看到 Provider 包 Consumer 包 withRouter 包 connect 包 withTheme——真正的業務元件埋在第七層。

**第二，生命週期把相關邏輯拆散了。** `componentDidMount` 裡同時塞著 API 呼叫和 event listener 註冊，而對應的清理卻在 `componentWillUnmount`。不相關的程式碼擠在同一個方法裡，相關的程式碼散落在不同方法裡。要追一個功能的完整邏輯，你得在三個生命週期方法之間跳來跳去。

**第三，`this` 讓所有人困惑。** 建構函式裡的 `this.handleClick = this.handleClick.bind(this)` 不只是醜——它是真的讓新手和老手都踩坑。class component 還有另一個問題：它們的 minification 效果差，而且讓 hot reloading 變得不穩定。

Hooks 的目標很明確：讓你把相關的邏輯放在一起，讓邏輯共享不需要改變元件樹結構，然後順便幹掉 `this`。但要達成這些目標，React 團隊選了一個激進的底層設計——用呼叫順序來識別每一個 Hook。

---

## 鏈表的真相：每個 Hook 就是一個節點

打開 React 原始碼的 `packages/react-reconciler/src/ReactFiberHooks.js`，你會看到這個型別定義：

```javascript
type Hook = {
  memoizedState: any,    // 這個 Hook 目前的值
  baseState: any,        // 用於計算優先級的基礎狀態
  baseQueue: Update | null,
  queue: UpdateQueue | null,  // 狀態更新佇列
  next: Hook | null,     // 指向下一個 Hook —— 這就是鏈表
};
```

關鍵在最後一行：`next: Hook | null`。這個欄位把每個 Hook 串成一條單向鏈表，掛在 Fiber 節點的 `memoizedState` 上。

當你的元件長這樣：

```javascript
function MyComponent() {
  const [count, setCount] = useState(0);          // Hook 1
  const [name, setName] = useState('React');       // Hook 2
  useEffect(() => { document.title = name; }, [name]); // Hook 3
  const doubled = useMemo(() => count * 2, [count]);   // Hook 4
  return <div>{doubled}</div>;
}
```

React 內部建立的資料結構是：

```
Fiber.memoizedState
  │
  ▼
Hook1 (useState)          Hook2 (useState)
┌──────────────────┐     ┌──────────────────┐
│ memoizedState: 0 │     │ memoizedState:   │
│ queue: {         │     │   'React'        │
│   dispatch: set… │ ──▶ │ queue: {         │ ──▶ ...
│ }                │     │   dispatch: set… │
│ next: ─────────────┘   │ }                │
└──────────────────┘     │ next: ─────────────┘
                         └──────────────────┘

  ──▶ Hook3 (useEffect)       ──▶ Hook4 (useMemo)
      ┌──────────────────┐        ┌──────────────────┐
      │ memoizedState:   │        │ memoizedState:   │
      │   Effect {...}   │        │   [0, [0]]       │
      │ next: ─────────────┘ ──▶  │ next: null       │
      └──────────────────┘        └──────────────────┘
                                  (鏈表結束)
```

沒有 key，沒有名稱，沒有 Map——純粹靠位置。第一個呼叫的 Hook 是第一個節點，第二個呼叫的是第二個節點，以此類推。

`memoizedState` 欄位特別有趣，因為它在不同類型的 Hook 裡存的東西完全不同：

| Hook 類型 | `memoizedState` 儲存的內容 |
|---|---|
| `useState` | 狀態值本身 |
| `useEffect` | 一個 `Effect` 物件 |
| `useMemo` | `[快取值, 依賴陣列]` tuple |
| `useCallback` | `[callback, 依賴陣列]` tuple |
| `useRef` | `{ current: value }` 物件 |

同一個欄位，七種語義。這就是為什麼 React 的型別定義裡到處都是 `any`——這不是偷懶，是設計上的取捨。

---

## 首次渲染 vs 重新渲染：兩套完全不同的邏輯

React 不是每次渲染都重新建立鏈表。首次渲染（mount）和後續渲染（update）走的是完全不同的路徑，而控制這一切的是 **Dispatcher 模式**。

### renderWithHooks：一切的入口

每次 React 要渲染一個 function component，都會經過這個函式：

```javascript
function renderWithHooks(current, workInProgress, Component, props, ...) {
  // 設定全域狀態
  currentlyRenderingFiber = workInProgress;
  workInProgress.memoizedState = null;  // 清空，準備重建

  // 關鍵決策：首次渲染還是更新？
  ReactSharedInternals.H =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount     // 首次：建立新鏈表
      : HooksDispatcherOnUpdate;   // 更新：走訪舊鏈表

  // 呼叫你的元件函式——你的 Hooks 在這裡執行
  let children = Component(props);

  // 清理：把 dispatcher 切回「禁止呼叫」模式
  ReactSharedInternals.H = ContextOnlyDispatcher;

  return children;
}
```

注意最後一行：渲染完成後，dispatcher 被切換成 `ContextOnlyDispatcher`。這個 dispatcher 對所有 Hook 呼叫都會拋出錯誤：

```
Error: Invalid hook call. Hooks can only be called inside of
the body of a function component.
```

所以當你在 event handler 或一般函式裡呼叫 `useState()`，不是什麼 lint 規則在擋你，是 React 在 runtime 直接炸給你看。

### 四個 Dispatcher，四種行為

| Dispatcher | 啟用時機 | 行為 |
|---|---|---|
| `ContextOnlyDispatcher` | 渲染週期外 | 所有 Hook 拋出錯誤 |
| `HooksDispatcherOnMount` | 首次渲染 | 呼叫 `mount*` 版本，**建立**鏈表 |
| `HooksDispatcherOnUpdate` | 重新渲染 | 呼叫 `update*` 版本，**走訪**鏈表 |
| `HooksDispatcherOnRerender` | 渲染中觸發的更新 | 重用 work-in-progress hooks |

每個 dispatcher 就是一個物件，把 `useState`、`useEffect` 等名稱映射到不同的實作：

```javascript
const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  useMemo: mountMemo,
  // ...
};

const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  useMemo: updateMemo,
  // ...
};
```

當你寫 `React.useState()`，它實際上解析成 `ReactSharedInternals.H.useState()`。同一行程式碼，在 mount 和 update 時跑的是完全不同的函式。

### mountWorkInProgressHook：建立鏈表

首次渲染時，每個 Hook 呼叫都會觸發這個函式：

```javascript
function mountWorkInProgressHook() {
  const hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // 第一個 Hook：設為鏈表的 head
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 後續 Hook：接到鏈表尾端
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

乾淨俐落。建立新節點，如果是第一個就掛到 Fiber 上，不是就接到尾端。`workInProgressHook` 同時扮演「目前位置」的游標角色。

### updateWorkInProgressHook：走訪鏈表

重新渲染時就複雜了。React 不是重建鏈表，而是**走訪上一次渲染的鏈表**，逐一 clone 節點：

```javascript
function updateWorkInProgressHook() {
  // 從上一次渲染的鏈表取出下一個節點
  let nextCurrentHook;
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current.memoizedState;  // 從 head 開始
  } else {
    nextCurrentHook = currentHook.next;  // 移到下一個
  }

  if (nextCurrentHook === null) {
    // 上一次渲染沒有更多 Hook 了，但這次還在呼叫
    // 這就是那個經典錯誤訊息的來源：
    throw new Error('Rendered more hooks than during the previous render.');
  }

  currentHook = nextCurrentHook;

  // Clone 這個節點到新的鏈表上
  const newHook = {
    memoizedState: currentHook.memoizedState,
    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,
    next: null,
  };

  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
  } else {
    workInProgressHook = workInProgressHook.next = newHook;
  }
  return workInProgressHook;
}
```

**這段程式碼就是「Hooks 規則」的執法者。** 它用兩個游標同步走訪新舊鏈表。如果你在重新渲染時呼叫了比上次更多或更少的 Hooks，游標就會對不上——要嘛 `nextCurrentHook` 提前變成 `null`（比上次多），要嘛到結尾還有剩餘節點（比上次少）。

### 一個有意思的發現：useState 就是 useReducer

看看 `mountState` 和 `updateState` 的原始碼：

```javascript
function mountState(initialState) {
  const hook = mountWorkInProgressHook();
  // ... 初始化 queue 和 dispatch
  return [hook.memoizedState, dispatch];
}

function updateState(initialState) {
  return updateReducer(basicStateReducer, initialState);
}
```

`updateState` 直接呼叫 `updateReducer`。`useState` 在更新階段就是一個使用了最簡單 reducer 的 `useReducer`：

```javascript
function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action;
}
```

這就是為什麼 `setState(prev => prev + 1)` 能用——reducer 檢查 action 是不是函式，是的話就把目前狀態傳進去。所以當你在爭論「該用 useState 還是 useReducer」時，底層其實是同一個東西。

---

## 條件式 Hook 的崩壞過程

現在你知道了鏈表的結構，讓我們具體看看條件式 Hook 怎麼炸的：

```javascript
function BadComponent({ showName }) {
  const [surname, setSurname] = useState('Doe');    // 位置 1
  if (showName) {
    const [name, setName] = useState('John');       // 位置 2（有條件的！）
  }
  const [age, setAge] = useState(25);               // 位置 3 或 2
  useEffect(() => { /* ... */ });                   // 位置 4 或 3
}
```

**第一次渲染**（showName = true），鏈表長這樣：

```
Hook1(surname:'Doe') → Hook2(name:'John') → Hook3(age:25) → Hook4(effect) → null
```

**第二次渲染**（showName = false），React 開始走訪舊鏈表：

1. `useState('Doe')` → 游標指向 Hook1 → 拿到 `'Doe'` ✓
2. `useState(25)` → 游標指向 Hook2 → 拿到 **`'John'`** ✗ **（應該是 age 的狀態，拿到了 name 的！）**
3. `useEffect(...)` → 游標指向 Hook3 → 拿到 **`25`** ✗ **（用 age 的值當 effect 物件，完全錯亂）**
4. 鏈表還剩 Hook4，但沒有更多 Hook 呼叫了

整條鏈表的狀態映射向前位移了一格。更恐怖的是，位置 2 的 age 狀態靜靜地拿到了 `'John'` 這個字串值——不會報錯，只會行為異常。這種 bug 可以潛伏很久才被發現。

最終 React 會偵測到「呼叫的 Hook 數量比上次少」然後拋出錯誤，但在那之前，你的狀態已經被污染了。

---

## Dan Abramov 為什麼否決所有替代方案

在「[Why Do React Hooks Rely on Call Order?](https://overreacted.io/why-do-hooks-rely-on-call-order/)」這篇文章裡，Dan Abramov 系統性地審視並否決了每一個提案：

**方案一：用字串 key 識別。** `useState('count', 0)` 看起來很合理，但問題是 key 碰撞。當兩個獨立開發的 custom hook 恰好用了同一個 key 字串，它們會靜默地共享狀態。更慘的是，如果一個 library 的作者在 hook 裡新增了一個狀態變數，可能會跟使用者的 key 撞車。

**方案二：用 Symbol 避免碰撞。** Symbol 是唯一的，理論上不會撞。但是：

```javascript
function Form() {
  const name = useFormInput();     // 內部用 Symbol('input')
  const surname = useFormInput();  // 同一個 Symbol！狀態撞車！
}
```

同一個 custom hook 呼叫兩次，裡面的 Symbol 是同一個。

**方案三：只允許一個 useState。** 回到 `this.state` 的模式。但這就意味著你無法把有狀態的邏輯抽成 custom hook——因為每個元件只有一塊狀態。這直接否定了 Hooks 存在的意義。

**最強的論點是可組合性。** 在呼叫順序的設計下，custom hook 可以自由組合。你不需要檢查整條依賴鏈有沒有 key 碰撞。多個 custom hook 可以各自呼叫其他 custom hook，不需要任何協調。這種無摩擦的組合能力就是 Hooks 真正的突破——而它的代價，就是「不能放在 if 裡面」這條規則。

---

## 第三條鏈表：Effect 的循環鏈表

除了 Hook 鏈表，React 還維護一條獨立的 **Effect 循環鏈表**，掛在 `fiber.updateQueue.lastEffect` 上：

```javascript
const effect = {
  tag: HookFlags,        // Passive | Layout | HookHasEffect
  create: () => cleanup, // 你傳給 useEffect 的函式
  inst: { destroy: undefined }, // 清理函式
  deps: Array | null,    // 依賴陣列
  next: null             // 指向下一個 Effect（循環！）
};
```

「循環」是重點——`lastEffect.next` 指回第一個 Effect。這讓 React 可以從任何位置開始遍歷整條鏈，而且不需要額外儲存 head 指標。

Effect 的生命週期是這樣的：

1. **渲染階段**：每個 `useEffect` 呼叫都透過 `pushEffect` 建立一個 Effect 節點，加入循環鏈表
2. **Commit 階段**：React 非同步排程 `flushPassiveEffects`
3. **Unmount pass**：遍歷鏈表，執行帶有 `HookHasEffect` flag 的 Effect 的清理函式（上一次的 `destroy`）
4. **Mount pass**：執行 `create` 函式，回傳值成為新的 `destroy`（下一次的清理函式）
5. **重新渲染**：`updateEffect` 用淺比較檢查依賴陣列。如果沒變，建立 Effect 但**不設 `HookHasEffect` flag**——commit 階段會跳過它

還有一條循環鏈表是 **Update Queue**，掛在每個 `useState` Hook 的 `queue.pending` 上。當你連續呼叫多次 `setState`，每個更新都會被串進這條循環鏈表，等到下次渲染時批次處理。循環設計讓插入操作是 O(1)——在 `pending`（最後一個更新）和 `pending.next`（第一個更新）之間插入，不需要遍歷。

三條鏈表，三種用途：Hook 鏈表管狀態映射，Effect 鏈表管副作用執行，Update 鏈表管狀態更新佇列。

---

## Stale Closure：Hooks 帶來的新 Bug 品種

Class component 從來沒有 stale closure 問題——`this.state` 永遠指向最新的值。但 Hooks 把狀態放進了函式閉包，而每次渲染都是一個新的閉包作用域。當一個 callback 持續引用舊渲染的閉包值，就產生了「stale closure」。

最經典的案例：

```javascript
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);  // 永遠印 0！
    }, 1000);
    return () => clearInterval(id);
  }, []);  // 空依賴 = 只跑一次 = 閉包永遠捕獲 count=0
}
```

解法有三個等級：

**等級一：修正依賴陣列。** 把 `count` 加進 deps，讓 effect 隨著 count 變化重新執行。缺點是每次 count 變化都會清除並重建 interval。

**等級二：函式式更新。** 如果你要更新狀態，用 `setState(prev => prev + 1)` 而不是 `setState(count + 1)`。函式式更新從 React 內部取得最新狀態，繞過閉包。

**等級三：useRef 當逃生口。** 把最新值存進 ref，在 callback 裡讀 `ref.current`：

```javascript
const countRef = useRef(count);
countRef.current = count;  // 每次渲染都更新

useEffect(() => {
  const id = setInterval(() => {
    console.log(countRef.current);  // 永遠是最新值
  }, 1000);
  return () => clearInterval(id);
}, []);  // 安全：ref 是 mutable 的
```

React 19.2+ 新增的 `useEffectEvent` 提供了更優雅的解法，它讓你宣告一個「永遠讀取最新 props/state」的事件函式，不需要加進依賴陣列。但截至 2026 年初，這個 API 仍在逐步推廣中。

---

## 效能的真話：你可能不需要 useMemo

每個 `useMemo` 和 `useCallback` 都會：

1. 在鏈表中建立一個 Hook 節點（記憶體分配）
2. 儲存 `[值, 依賴陣列]` tuple
3. **每次渲染**都對依賴陣列做淺比較
4. 阻止被快取的值被 GC 回收

對於便宜的計算，memoization 的開銷可能超過計算本身。而且最常見的 `useCallback` 用法根本沒有效果：

```javascript
// 沒用：Parent 重新渲染 → Child 還是重新渲染
function Parent() {
  const handleClick = useCallback(() => doSomething(), []);
  return <Child onClick={handleClick} />;
}
```

除非 `Child` 被 `React.memo` 包裝，否則 `useCallback` 提供零收益——Child 因為 parent re-render 而 re-render，跟 props 有沒有變無關。

但 2025 年 10 月 React Compiler v1.0 正式發布後，遊戲規則變了。Compiler 在編譯階段自動分析元件的資料流，在有意義的地方插入 memoization。在 Meta 的生產環境（Quest Store、Instagram）中，Compiler 帶來了高達 2.5 倍的互動速度提升。Next.js 16 已內建穩定支援。

這意味著大多數手動的 `useMemo` / `useCallback` 不再必要。Compiler 能覆蓋包括條件路徑在內的 memoization 場景——這是手動 memoization 做不到的。

---

## 你現在該知道的事

React Hooks 的底層不神秘，就是一條掛在 Fiber 上的單向鏈表加上一套 dispatcher 切換機制。首次渲染建立鏈表，後續渲染走訪鏈表。呼叫順序就是索引——這個設計帶來了無摩擦的組合能力，代價是那幾條你已經背熟的規則。

幾個實際建議：

- **讓 ESLint `react-hooks/exhaustive-deps` 規則幫你。** 它比你更懂依賴陣列。如果你經常 suppress 它的警告，問題通常不在規則，而是在你的程式碼結構。
- **優先用函式式更新避免 stale closure。** `setState(prev => prev + 1)` 比修正依賴陣列簡單得多。
- **別再手動 useMemo/useCallback 了。** 如果你的專案能用 React Compiler，讓它幫你做。如果不能，只在有 `React.memo` 搭配時或真正昂貴的計算上用。
- **Custom hook 是組合的單位，不是狀態共享的工具。** 每個元件呼叫 `useCounter()` 拿到的是獨立的計數器。要共享狀態，用 Context 或外部 store。

下次有人問你「為什麼 Hooks 不能放在 if 裡面」，你可以回答：「因為底層是鏈表，靠位置索引，條件呼叫會讓游標對不上。」一句話，比文件上寫的清楚十倍。

---

## 延伸閱讀

- [ReactFiberHooks.js — React 原始碼](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js) — 本文所有原始碼引用的出處，直接看最準
- [Why Do React Hooks Rely on Call Order?](https://overreacted.io/why-do-hooks-rely-on-call-order/) — Dan Abramov 親自解釋為什麼否決所有替代方案
- [Algebraic Effects for the Rest of Us](https://overreacted.io/algebraic-effects-for-the-rest-of-us/) — Hooks 背後的理論基礎：代數效應
- [React Compiler v1.0 發布公告](https://react.dev/blog/2025/10/07/react-compiler-1) — 自動 memoization 的時代來了
- [React Hooks 的生命週期深入解析](https://jser.dev/react/2022/01/19/lifecycle-of-effect-hook/) — Effect 循環鏈表的詳細圖解
- [The Useless useCallback](https://tkdodo.eu/blog/the-useless-use-callback) — TkDodo 解釋為什麼多數 useCallback 沒用
