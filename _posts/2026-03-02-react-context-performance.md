---
title: "React Context 效能陷阱全解：為什麼你的 App 瘋狂 Re-render，以及 Signals 是不是解藥"
date: 2026-03-02
description: "深入 React 原始碼剖析 Context 的 propagateContextChange 機制，解釋為什麼 React.memo 擋不住 Context re-render，並比較 Context splitting、Zustand、Signals 等解決方案的實際效果。"
tags: [deep-dive, react, performance, signals]
---

你有沒有遇過這種情境：使用者在表單裡打了一個字，整個頁面的元件全部閃了一下？

打開 React DevTools Profiler 一看，幾十個不相干的元件全部重新渲染。你困惑地盯著螢幕，心想：「我明明只改了 `name`，為什麼連 `Footer` 都在 re-render？」

然後你往上追，發現罪魁禍首是一個 `<AppContext.Provider>`，裡面塞了整個 app 的狀態。

恭喜你，你踩中了 React 最經典的效能陷阱之一。

今天我們從原始碼的角度，把 Context 的效能問題拆解得清清楚楚。你會明白為什麼 `React.memo` 救不了你、為什麼 Dan Abramov 說「這是設計如此」、以及那些號稱能解決問題的方案——Context splitting、Zustand、Signals——到底各自解決了什麼。

---

## Context 的本質：它不是狀態管理工具

先搞清楚一件事：**Context 是依賴注入（dependency injection）機制，不是狀態管理工具。**

React 官方和 Dan Abramov 都反覆強調過這一點。Context 解決的問題是「把值從 A 點傳到 B 點，不用經過中間的 C、D、E」。它是傳遞管道，不是狀態容器。

但因為 `useState` + `Context` 確實能做到「全域共享狀態」，所以大家就拿它當 Redux 用了。這不是不行，但你得理解它的代價。

代價就是：**Context 的更新是全有或全無的。** 你沒辦法告訴 React「我只想通知 A 元件，不要通知 B 元件」。只要 Provider 的 value 變了，所有消費者一視同仁，全部重新渲染。

為什麼？因為 React 原始碼就是這樣寫的。

---

## 深入原始碼：propagateContextChange 的無差別攻擊

要理解 Context 的效能問題，你得看 React 在 Provider value 改變時到底做了什麼。答案就在 `ReactFiberNewContext.js` 裡的 `propagateContextChange` 函式。

### Provider 怎麼判斷「值變了」

當一個 `<Context.Provider>` 重新渲染時，React 用 `Object.is()` 比較新舊 value：

```javascript
if (Object.is(oldValue, newValue)) {
  // 值沒變，跳過傳播
  return;
}
// 值變了，啟動 propagateContextChange
```

注意是 `Object.is`，不是深比較。這意味著：

```javascript
// 這永遠觸發更新，即使 user 和 theme 的值完全一樣
<MyContext.Provider value={{ user, theme }}>
```

每次 render 都產生一個新物件，`Object.is` 回傳 `false`，遊戲結束。

### propagateContextChange 怎麼運作

一旦判定值改變了，React 開始**走遍 Provider 底下的整棵 fiber 子樹**：

```
propagateContextChange(workInProgress, context, renderLanes)
  │
  ├─ 從 Provider 的 child 開始
  ├─ 走訪每一個 fiber 節點
  │   ├─ 檢查這個 fiber 的 dependencies 鏈表
  │   ├─ 如果鏈表裡有匹配的 context
  │   │   ├─ 更新這個 fiber 的 lanes（標記為需要更新）
  │   │   └─ 呼叫 scheduleContextWorkOnParentPath()
  │   │       └─ 沿著 return 指標一路往上，確保所有祖先都知道子樹有活要幹
  │   └─ 繼續走訪下一個 fiber
  └─ 直到遍歷完整棵子樹
```

用簡化的虛擬碼來看：

```javascript
function propagateContextChange(provider, context, renderLanes) {
  let fiber = provider.child;

  while (fiber !== null) {
    // 檢查這個 fiber 是否消費了這個 context
    let dep = fiber.dependencies?.firstContext;
    while (dep !== null) {
      if (dep.context === context) {
        // 找到消費者！標記它需要更新
        fiber.lanes = mergeLanes(fiber.lanes, renderLanes);
        // 通知所有祖先：底下有工作要做
        scheduleContextWorkOnParentPath(fiber, renderLanes);
        break;
      }
      dep = dep.next;
    }
    fiber = getNextFiber(fiber);
  }
}
```

這裡有幾個重點：

**第一，它走訪整棵子樹。** 不管你的 Provider 底下有 10 個還是 1000 個元件，全部都要走一遍去找消費者。

**第二，dependencies 是一個鏈表。** 每當元件呼叫 `useContext(MyContext)`，底層的 `readContext()` 就會把這個 context 加到 fiber 的 dependencies 鏈表裡。一個元件可以訂閱多個 context。

**第三，也是最關鍵的：這個傳播機制會穿透 `React.memo` 和 `shouldComponentUpdate`。**

### React.memo 擋不住 Context

很多人以為「我的元件用了 `React.memo`，應該能避免不必要的 re-render 吧？」

不行。

`propagateContextChange` 直接在 fiber 樹上走訪，它不管你的元件外面包了什麼。它找到消費者就標記 lanes，強制排程更新。`React.memo` 只比較 props，而 context 的更新完全繞過 props 比較機制。

Dan Abramov 在 GitHub Issue [#15156](https://github.com/facebook/react/issues/15156) 裡明確回應：**「This is working as designed.」**（這是設計如此。）

這不是 bug，這是 feature。

---

## 四大反模式：你可能全中了

理解了原始碼，我們來看最常見的效能雷區。

### 反模式 1：巨型 Context

```javascript
// 整個 app 的狀態塞在一個 context 裡
const AppContext = createContext({
  user: null,
  theme: 'light',
  locale: 'en',
  notifications: [],
  cart: [],
  sidebarOpen: false,
  // ...還有 20 個欄位
});
```

任何一個欄位變動，所有 `useContext(AppContext)` 的元件全部 re-render。你的 `<Footer>` 明明只讀 `theme`，但 `user` 登入時它也得跟著跑一趟。

### 反模式 2：每次 Render 都產生新物件

```javascript
function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  // 每次 render 都是新的物件參考
  return (
    <AppContext.Provider value={{ user, theme, setUser, setTheme }}>
      {children}
    </AppContext.Provider>
  );
}
```

即使 `user` 和 `theme` 都沒變，只要 `AppProvider` 因為**任何原因**重新渲染（比如它的父元件 re-render 了），就會產生新的 value 物件。`Object.is` 比較失敗，所有消費者全部觸發更新。

這是最陰險的一個，因為你的狀態根本沒變，但 re-render 照樣發生。

### 反模式 3：高頻更新放 Context

```javascript
function MouseProvider({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);

  return (
    <MouseContext.Provider value={pos}>
      {children}
    </MouseContext.Provider>
  );
}
```

滑鼠移動每秒觸發 60 次以上，每次都產生新的 `{ x, y }` 物件，每次都觸發所有消費者 re-render。動畫值、捲動位置、拖拽座標——都是同樣的災難。

### 反模式 4：把函式和狀態混在同一個 Context

```javascript
const value = {
  state,                              // 會變
  updateName: (name) => dispatch(...), // 不會變（但因為跟 state 綁一起，一起完蛋）
  updateEmail: (email) => dispatch(...)
};
```

`dispatch` 本身是穩定參考，但因為它跟 `state` 放在同一個物件裡，每次 `state` 變動都連帶產生新物件。那些只需要呼叫 `updateName` 但不讀 `state` 的元件，也被拖下水。

---

## 解法一覽：從簡單到徹底

### 解法 1：useMemo 包裝 value（最低成本）

```javascript
function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  const value = useMemo(
    () => ({ user, theme, setUser, setTheme }),
    [user, theme]
  );

  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}
```

這解決了反模式 2：當 `user` 和 `theme` 沒變時，不會因為父元件 re-render 而產生假陽性。但當 `user` 真的變了，所有消費者還是會 re-render——即使某些元件只讀 `theme`。

**效果：堵住假陽性，沒解決根本問題。**

### 解法 2：Context Splitting（中等成本）

把一個大 Context 拆成多個小 Context：

```javascript
const UserContext = createContext(null);
const ThemeContext = createContext('light');
const APIContext = createContext(null);

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  // API context 永遠不變（dispatch 是穩定參考）
  const api = useMemo(() => ({
    setUser,
    setTheme,
  }), []);

  return (
    <APIContext.Provider value={api}>
      <UserContext.Provider value={user}>
        <ThemeContext.Provider value={theme}>
          {children}
        </ThemeContext.Provider>
      </UserContext.Provider>
    </APIContext.Provider>
  );
}
```

關鍵技巧：**把「會變的資料」和「不會變的操作」分開。** `APIContext` 只放 setter 函式，用 `useMemo` 加空依賴陣列包住（因為 `useState` 的 setter 本身就是穩定參考）。這樣只需要呼叫 `setUser` 的元件不會因為 `user` 改變而 re-render。

實測效果：在一個有 30+ 欄位的表單中，從「打一個字觸發 9 個元件 re-render」降到「只有 2 個元件 re-render」。

**缺點：Provider 金字塔。** 拆得越細，巢狀越深。三、五個還行，十幾個就開始懷疑人生了。

### 解法 3：消費者端的 useMemo（Dan Abramov 推薦）

Dan Abramov 在 Issue #15156 提出的方案：

```javascript
function Button() {
  const { theme } = useContext(AppContext);

  // 即使 Button 因為 context 變化而 re-render，
  // 只要 theme 沒變，底下的 ExpensiveTree 就不會重新渲染
  return useMemo(
    () => <ExpensiveTree className={theme} />,
    [theme]
  );
}
```

`Button` 的函式本體還是會執行（因為 context 變了），但 `useMemo` 讓 React 跳過 `ExpensiveTree` 的 re-render。這招的本質是：**讓 re-render 發生，但讓它變便宜。**

### 解法 4：外部狀態管理（Zustand、Jotai）

如果你的狀態夠複雜，用真正的狀態管理工具可能更乾脆：

```javascript
// Zustand：基於 selector 的訂閱
import { create } from 'zustand';

const useStore = create((set) => ({
  user: null,
  theme: 'light',
  setUser: (user) => set({ user }),
  setTheme: (theme) => set({ theme }),
}));

// 只有 theme 變的時候才 re-render
function ThemeButton() {
  const theme = useStore((state) => state.theme);
  return <button className={theme}>Click</button>;
}

// 只有 user 變的時候才 re-render
function UserBadge() {
  const user = useStore((state) => state.user);
  return <span>{user?.name}</span>;
}
```

Zustand 和 Jotai 的核心差異在於：它們用 `useSyncExternalStore` 搭配 selector 實現**精準訂閱**。只有你 select 的那一小塊狀態變了，元件才會 re-render。

這正是 Context 做不到的事。

**Jotai** 走更遠一步，用原子模型（atomic model）把每個狀態片段變成獨立的 atom。元件只訂閱特定 atom，效能是「手術刀等級」的精準。

---

## Signals：另一個宇宙的解法

到目前為止，所有解法都在 React 的「由上而下、整棵子樹重新渲染」模型裡打轉。Signals 則代表一個根本性的不同思路。

### React 模型 vs Signals 模型

```
React: 在你「建立」狀態的地方觸發 re-render → 往下擴散
Signals: 在你「使用」狀態的地方觸發更新 → 精準命中
```

在 React 裡，`setState` 觸發元件函式重新執行，產生新的 JSX，diff Virtual DOM，然後 patch 真實 DOM。整個元件是更新的最小單位。

在 Signals 裡，signal 自己追蹤誰讀了它。當值改變時，只有真正讀取那個值的計算和 DOM 節點收到通知。不用重新執行元件函式，不用 diff Virtual DOM。

### Preact Signals 的實際示範

```javascript
import { signal, computed } from '@preact/signals';

const count = signal(0);
const double = computed(() => count.value * 2);

function Counter() {
  return (
    <button onClick={() => count.value++}>
      {count} × 2 = {double}
    </button>
  );
}
```

當 `count` 直接放在 JSX 裡時，Preact 可以**直接更新對應的 text node**，連元件函式都不需要重新執行。整個 `@preact/signals` 只有 1.6KB。

### Signals 怎麼解決 Context 問題

把 signals 放進 context，context 就變成純粹的依賴注入，不再是 re-render 的觸發器：

```javascript
// 傳統 React Context：改 count 時，讀 name 的元件也 re-render
const ComponentA = () => {
  const { name, count } = useContext(AppContext);
  return <div>{name}</div>; // count 變了我也得跑一趟
};

// Signal-based Context：context 物件本身永遠不變
const AppContext = createContext({
  name: signal('Alice'),
  count: signal(0),
});

const ComponentA = () => {
  const { name } = useContext(AppContext);
  return <div>{name.value}</div>; // 只有 name.value 變了才更新
};
```

context 物件裡放的是 signal 的參考，不是值本身。參考永遠不變，所以 `Object.is` 永遠回傳 `true`，`propagateContextChange` 永遠不會觸發。而 signal 的 `.value` 存取自帶細粒度追蹤——完美繞過了整個問題。

### TC39 Signals 提案：語言層級的標準化

JavaScript 本身可能會內建 signals。TC39 Signals 提案目前在 Stage 1，由 Angular、Preact、Vue、Solid、Svelte 等框架的核心成員聯合推動：

```javascript
const counter = new Signal.State(0);
const isEven = new Signal.Computed(() => (counter.get() & 1) === 0);

counter.set(5);
isEven.get(); // false
```

如果這個提案最終進入標準，所有框架就能共享同一套反應式原語。你在 Solid 裡建立的 signal，理論上可以在 Vue 元件裡直接使用。

但——React 團隊對此態度不明。React 的哲學一直是「把複雜性留在框架裡，讓元件保持簡單的函式呼叫」。Signals 要求開發者顯式地 `.value` 存取，這跟 React 的心智模型有衝突。

---

## React 自己在做什麼？

### use(Context)：語法糖，不是解藥

React 19 新增了 `use()` API，可以條件式地讀取 context：

```javascript
function Heading({ children }) {
  if (!children) return null;

  // useContext 放在 if 後面會違反 Rules of Hooks
  // use() 不會
  const theme = use(ThemeContext);
  return <h1 style={{ color: theme.color }}>{children}</h1>;
}
```

方便嗎？方便。解決效能問題嗎？完全沒有。所有消費者依然會在 context 值改變時 re-render，跟 `useContext` 行為一模一樣。

### useContextSelector：那個永遠沒來的 RFC

RFC [#119](https://github.com/reactjs/rfcs/pull/119) 提出了 `useContextSelector`：

```javascript
// 理想中的 API：只有 theme 變的時候才 re-render
const theme = useContextSelector(MyContext, (ctx) => ctx.theme);
```

RFC 作者的測試數據：在動畫場景中，**useContext 40ms vs useContextSelector 4ms**——10 倍的差距。

但 React 團隊一直沒有合併這個 RFC。原因包括：

1. **Concurrent Mode 下實現太複雜。** Selector 需要在渲染中途比較新舊值，這跟 concurrent rendering 的時間切片模型有衝突。
2. **React 團隊認為這是架構問題。** josephsavona（React Compiler 核心成員）的原話：「Any reactive system is going to have a hard time with a single immutable blob of state.」——問題不在 Context 的 API，在你把整坨狀態塞進去。
3. **React Compiler 被視為替代方案。** 團隊認為 Compiler 的自動 memoization 可以讓 re-render 變便宜到「不需要避免」。

不過 React 團隊也透露了正在開發的 **Store API**（PR [#33215](https://github.com/facebook/react/pull/33215)），描述為「concurrent-compatible variant of `useSyncExternalStore`」。如果這個 API 落地，可能才是 React 官方對精準訂閱的正式回答。

### React Compiler：讓 Re-render 變便宜

React Compiler（前身 React Forget）的做法不是消除 re-render，而是讓 re-render 更便宜：

```javascript
// 你寫的程式碼
function UserCard() {
  const { user } = useContext(AppContext);
  const formatted = formatDate(user.joinDate);
  return <Card title={user.name} subtitle={formatted} />;
}

// Compiler 自動產生的效果（概念上）
function UserCard() {
  const { user } = useContext(AppContext);
  const formatted = useMemo(() => formatDate(user.joinDate), [user.joinDate]);
  return useMemo(() => <Card title={user.name} subtitle={formatted} />, [user.name, formatted]);
}
```

元件函式還是會執行，但 Compiler 自動 memoize 了子元件的 JSX 輸出。如果 props 沒變，`<Card>` 就不會深入渲染。

**重要澄清：Compiler 不會修復 Context 的根本問題。** 消費者元件的函式本體還是會跑，只是跑完之後可能更快地 bailout。對於簡單元件，這足夠了；對於消費者本身就很重的情況，你還是需要前面提到的解法。

---

## 我的建議：務實的決策框架

別急著上 Signals，也別把所有東西都塞進 Zustand。根據你的實際場景選擇：

**Context + useMemo 就夠了**：如果你的 context 消費者不多（< 20 個）、更新頻率不高（不是每幀都變），把 value 用 `useMemo` 包住，加上適當的 context splitting，就是最佳性價比的解法。

**該用 Zustand/Jotai 了**：如果你有大量元件訂閱同一份狀態、需要精準的 selector-based 訂閱、或者狀態更新頻率很高——用外部狀態管理。Zustand 的 API 極簡，遷移成本低，而且完美支援 concurrent features。

**考慮 Signals**：如果你用 Preact、或者在評估下一個專案的技術棧。Signals 從根本上解決了 re-render 的粒度問題，但它要求你接受一個不同的心智模型。在 React 生態裡，目前還是外掛狀態（Preact Signals 有 `@preact/signals-react`，但不保證長期穩定）。

**最不該做的事**：把 Context 當 Redux 用，塞一個巨型物件進去，然後到處 `useContext`。這會讓你的 app 變成 re-render 地獄，而且 React Compiler 也救不了你。

Context 本身沒有錯。它是一個優秀的依賴注入工具。問題在於我們期待它做的事——精準的狀態訂閱——從來就不在它的設計目標裡。理解這一點，你就能做出對的選擇。

---

## 延伸閱讀

- [How does Context work internally in React?](https://jser.dev/react/2021/07/28/how-does-context-work/) — 從原始碼層級拆解 Context 的 fiber 走訪機制
- [How to write performant React apps with Context](https://www.developerway.com/posts/how-to-write-performant-react-apps-with-context) — Context splitting 的實戰指南，附帶完整的效能比較
- [GitHub Issue #15156: Preventing rerenders with React.memo and useContext](https://github.com/facebook/react/issues/15156) — Dan Abramov 提出的三種 workaround 原始出處
- [RFC #119: Context selectors](https://github.com/reactjs/rfcs/pull/119) — 被擱置的 useContextSelector 提案，留言區有 React 團隊的設計考量
- [The Past and Future of Render Optimization with React Context](https://newsletter.daishikato.com/p/the-past-and-future-of-render-optimization-with-react-context) — Daishi Kato（use-context-selector 作者）的分析
- [Introducing Signals — Preact](https://preactjs.com/blog/introducing-signals/) — Signals 的設計哲學和效能對比
- [TC39 Signals Proposal](https://github.com/tc39/proposal-signals) — JavaScript 語言層級的 Signals 標準化提案
