---
title: "你真的懂 useEffect 嗎？從原始碼拆解 React Effect 的完整生命週期"
date: 2026-03-01
description: "深入 React 原始碼剖析 useEffect 的內部機制：Effect 環狀鏈表、HookHasEffect 旗標、cleanup 執行順序、dependency array 的 Object.is 比對，以及 Strict Mode 雙重觸發背後的 Offscreen API 佈局。搞懂這些，你才算真正理解 useEffect。"
tags: [deep-dive, react, hooks, frontend]
---

你用 `useEffect` 寫了幾百次了吧？但我問你三個問題：

1. cleanup 函式到底在什麼時候執行？是 re-render 前還是後？
2. dependency array 傳 `[]` 和不傳，底層差在哪？
3. Strict Mode 為什麼要把你的 effect 跑兩次？React 團隊腦子有問題嗎？

如果你有任何一題答不上來，這篇值得你花 15 分鐘讀完。我們不談 API 怎麼用——官方文件寫得夠好了——我們要打開 React 的引擎蓋，看看 `useEffect` 這台引擎到底怎麼運轉的。

---

## 先搞清楚：useEffect 不是 componentDidMount

這是最普遍的誤解，我得先把它釘在恥辱柱上。

Class component 時代的 `componentDidMount` 是**同步**的——它在 DOM 更新後、瀏覽器繪製前執行，等它跑完瀏覽器才會畫面。相當於 `useLayoutEffect` 的行為。

`useEffect` 完全不一樣。它是**非同步**的——React 透過 Scheduler 以 Normal Priority 排程，等瀏覽器畫完畫面後才執行。這個設計意圖很明確：**大部分 side effect 不需要阻塞畫面渲染**。用戶看到 UI 更新比你的 `console.log` 或 API 呼叫先跑完更重要。

時間軸長這樣：

```
Render Phase（虛擬 DOM 計算）
    ↓
Commit Phase
    ├── Before Mutation（getSnapshotBeforeUpdate）
    ├── Mutation（DOM 實際寫入）
    │     └── useLayoutEffect cleanup 執行
    ├── Layout（同步）
    │     └── useLayoutEffect create 執行
    │     └── ref 附加
    ↓
瀏覽器繪製畫面 ← 用戶此時已看到新 UI
    ↓
Passive Effects（非同步排程）
    ├── 第一輪：所有 useEffect cleanup 執行
    └── 第二輪：所有 useEffect create 執行
```

注意 `useEffect` 的 cleanup 和 create 被拆成兩輪。這不是隨便的設計——React 先跑完所有元件的 cleanup，再跑所有元件的 create。原因是**避免一個元件的 cleanup 影響到另一個元件剛建立的資源**。

---

## Effect 在 Fiber 上的資料結構

要懂 `useEffect`，你得先看它在記憶體裡長什麼樣。每個 effect 是這樣的物件：

```typescript
type Effect = {
  tag: HookFlags,          // 位元旗標，決定行為
  create: () => (() => void) | void,  // effect 回呼
  destroy: (() => void) | void,       // cleanup 函式
  deps: Array<mixed> | null,          // dependency array
  next: Effect,            // 指向下一個 effect
};
```

關鍵在 `next`——這是一個**環狀鏈表**（circular linked list）。一個元件裡的所有 effect（不管是 `useEffect` 還是 `useLayoutEffect`）串在同一個環上，掛在 fiber 的 `updateQueue.lastEffect` 上。

```
fiber.updateQueue.lastEffect
    ↓
  Effect₃ → Effect₁ → Effect₂ → Effect₃ (環狀)
    ↑                               ↑
  lastEffect                    lastEffect.next = firstEffect
```

為什麼用環狀鏈表？因為只需要存 `lastEffect` 一個指標，就能同時拿到第一個（`lastEffect.next`）和最後一個。遍歷用 `do...while` 從 `lastEffect.next` 開始，遇到 `lastEffect` 就停。空間效率很高。

### HookHasEffect：最關鍵的旗標

`tag` 欄位用位元旗標控制行為：

```javascript
const HookHasEffect = 0b0001;  // 需要執行
const HookInsertion  = 0b0010;  // useInsertionEffect
const HookLayout     = 0b0100;  // useLayoutEffect
const HookPassive    = 0b1000;  // useEffect
```

**最重要的是 `HookHasEffect`。** 它決定這個 effect 在 commit phase 要不要真的跑。`HookPassive` 或 `HookLayout` 只是標記類型，`HookHasEffect` 才是開關。

---

## mountEffect vs updateEffect：首次渲染 vs 重新渲染

### 首次渲染：mountEffect

```javascript
function mountEffectImpl(fiberFlags, hookFlags, create, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,  // 永遠包含 HookHasEffect
    create,
    undefined,  // 還沒執行過，沒有 destroy
    nextDeps,
  );
}
```

首次渲染很單純：**永遠加上 `HookHasEffect`**，因為 effect 從來沒跑過，當然要跑。`destroy` 是 `undefined`，因為 create 都還沒執行，哪來的 cleanup。

### 重新渲染：updateEffect

這裡才是精華：

```javascript
function updateEffectImpl(fiberFlags, hookFlags, create, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;  // 拿到前一次的 cleanup
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // deps 沒變：建立 Effect 但「不加」HookHasEffect
        hook.memoizedState = pushEffect(
          hookFlags, create, destroy, nextDeps
        );
        return;  // 提前返回，這個 effect 不會執行
      }
    }
  }

  // deps 變了：加上 HookHasEffect
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create, destroy, nextDeps,
  );
}
```

注意一個容易忽略的事實：**即使 deps 沒變，React 還是會建立新的 Effect 物件並推入環狀鏈表。** 只是不加 `HookHasEffect` 旗標，commit phase 掃到它時直接跳過。為什麼？因為環狀鏈表必須維持完整性——hook 的順序和數量在每次 render 都必須一致（這也是為什麼 hooks 不能放在 `if` 裡面）。

---

## Dependency Array 的比對：Object.is 不是 ===

`areHookInputsEqual` 的實作很直白：

```javascript
function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) {
    return false;  // 沒有前一次的 deps = 一定要跑
  }
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}
```

這裡的 `is()` 是 `Object.is`（或它的 polyfill）。跟 `===` 的差別在兩個邊界情況：

| 比較 | `===` | `Object.is` |
|------|-------|-------------|
| `+0` vs `-0` | `true` | `false` |
| `NaN` vs `NaN` | `false` | `true` |

99% 的情況你不會碰到這兩個 case，但 React 團隊選擇語義更正確的 `Object.is`。

### 三種 deps 模式的底層差異

```javascript
// 模式 1：有 deps
useEffect(fn, [a, b])
// nextDeps = [a, b]，逐一用 Object.is 比對

// 模式 2：空 deps
useEffect(fn, [])
// nextDeps = []，for 迴圈直接跳過（長度為 0），回傳 true
// 結果：永遠不重新執行

// 模式 3：不傳 deps
useEffect(fn)
// deps === undefined → nextDeps = null
// areHookInputsEqual 收到 nextDeps 時，因為 nextDeps 是 null，
// 在 updateEffectImpl 中 nextDeps !== null 這個判斷直接失敗
// 走到下面加上 HookHasEffect → 每次都執行
```

第三種模式的行為是由 `updateEffectImpl` 中 `if (nextDeps !== null)` 這個條件控制的。`undefined` 被轉成 `null`，導致整個 deps 比對被跳過，effect 就每次都跑。

**這是常見踩坑點：忘了傳 deps 導致 effect 在每次 render 都執行，搭配 `setState` 就是無限迴圈。**

---

## Cleanup 的執行時機：比你想的更微妙

很多人以為 cleanup 只在 unmount 時跑。錯。

Cleanup 在**每次 effect 要重新執行前**都會跑。精確的順序是：

1. 元件 re-render（render phase）
2. 新的 Effect 物件建立，帶著前一次的 `destroy` 引用
3. DOM 更新（commit mutation phase）
4. 瀏覽器繪製
5. **前一次 effect 的 cleanup（destroy）執行**——用的是**舊的** closure 值
6. **新的 effect create 執行**——用的是**新的** closure 值
7. 新的 cleanup 函式存回 `effect.destroy`

步驟 5 是關鍵：cleanup 捕獲的是它被建立時的 props 和 state，不是當前的。這是 JavaScript closure 的基本行為，但在 React effects 的語境下特別容易搞混。

### Commit Phase 的兩輪清掃

在 `flushPassiveEffectsImpl` 中，React 做了一件聰明的事：

```javascript
// 第一輪：跑所有元件的 cleanup
commitPassiveUnmountEffects(root.current);

// 第二輪：跑所有元件的 create
commitPassiveMountEffects(root, root.current);
```

先**全部 cleanup 完**，再**全部 create**。為什麼？

想像你有元件 A 和元件 B，A 的 cleanup 會釋放一個 WebSocket 連線，B 的 create 會建立一個新的。如果交錯執行（A cleanup → A create → B cleanup → B create），B 的 cleanup 可能意外斷掉 A 剛建的連線。分兩輪就沒這個問題。

遍歷順序也有講究：cleanup 是**由上到下**（parent → child），create 是**由下到上**（child → parent）。子元件的 effect 先完成初始化，父元件的 effect 才開始，確保父元件能安全存取子元件的狀態。

---

## Strict Mode 雙重觸發：React 團隊沒有瘋

React 18 開始，開發模式下 `<StrictMode>` 會把新掛載元件的 effect 跑兩次：

```
mount → setup effect → cleanup effect → setup effect（再來一次）
```

第一次看到 API 被呼叫兩次的時候，大部分人的反應是去 Stack Overflow 搜「why useEffect runs twice」然後拿掉 StrictMode。

**這是錯誤的做法。** StrictMode 是在幫你抓 bug。

### 背後的原因：Offscreen API

React 團隊正在開發一個叫 Activity（前身是 Offscreen）的 API。概念是：**元件可以被「隱藏」但不被卸載**，保留 state，等到再次顯示時重新執行 effects。

這個模式在實際場景很常見——tab 切換、路由堆疊、虛擬化列表。目前的做法是卸載再重建，丟失所有 state。Activity API 讓你保留 state 但重跑 effects（因為可能需要重新訂閱、重新連線）。

StrictMode 的雙重觸發就是在**模擬這個行為**：state 保留，effects 重跑。如果你的元件在這個流程下壞了，代表它有 cleanup 沒寫好的問題。

### 它抓什麼 bug？

```javascript
// 壞的：沒有 cleanup
useEffect(() => {
  window.addEventListener('resize', handleResize);
  // StrictMode 下會掛兩個 listener！
}, []);

// 好的：有 cleanup
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

```javascript
// 壞的：fetch 沒有取消機制
useEffect(() => {
  fetch('/api/data').then(res => setData(res));
  // StrictMode 下兩次 fetch，可能 race condition
}, []);

// 好的：用 ignore flag
useEffect(() => {
  let ignore = false;
  fetch('/api/data').then(res => {
    if (!ignore) setData(res);
  });
  return () => { ignore = true; };
}, []);
```

核心原則：**setup 裡取得的每一個資源，cleanup 裡都要完整釋放，而且要能重新取得**。如果你的 effect 符合這個原則，跑兩次跟跑一次結果一模一樣。

### 被保留 vs 被重跑

| 項目 | 行為 |
|------|------|
| `useState`, `useReducer`, `useRef` | **保留**，不會重置 |
| `useEffect`, `useLayoutEffect` | **重跑**（cleanup → create） |
| DOM 渲染結果 | **保留**，不會閃爍 |
| `useMemo`, `useCallback` | **重新計算**（但結果通常相同） |

Production build 完全不受影響，只有 development build 才會雙重觸發。

---

## 實戰陷阱與正確姿勢

### 陷阱 1：Stale Closure

```javascript
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);  // 永遠印 0
    }, 1000);
    return () => clearInterval(id);
  }, []);  // 空 deps = closure 永遠捕獲初始值
}
```

`setInterval` 的回呼捕獲的是第一次 render 時的 `count`（值為 0）。因為 deps 是 `[]`，effect 不會重跑，closure 也不會更新。

**三種解法：**

```javascript
// 1. 把 count 加入 deps（每次 count 變都重建 interval）
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000);
  return () => clearInterval(id);
}, [count]);

// 2. 用 functional update 避免讀取 count
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);  // 不需要讀 count
  }, 1000);
  return () => clearInterval(id);
}, []);

// 3. 用 ref 存最新值
const countRef = useRef(count);
countRef.current = count;
useEffect(() => {
  const id = setInterval(() => console.log(countRef.current), 1000);
  return () => clearInterval(id);
}, []);
```

### 陷阱 2：物件作為 dependency

```javascript
function Chat({ roomId }) {
  const options = { serverUrl: 'localhost', roomId };

  useEffect(() => {
    const conn = createConnection(options);
    conn.connect();
    return () => conn.disconnect();
  }, [options]);  // 每次 render 都是新物件 → 每次都重連
}
```

`Object.is({}, {})` 是 `false`。每次 render 都建立新的 `options` 物件，reference 不同，effect 就重跑。

**正解：把物件建立移到 effect 裡面，或展開成原始值：**

```javascript
useEffect(() => {
  const conn = createConnection({ serverUrl: 'localhost', roomId });
  conn.connect();
  return () => conn.disconnect();
}, [roomId]);  // roomId 是 string，Object.is 比對沒問題
```

### 陷阱 3：useEffectEvent 的救贖

React 19 帶來了 `useEffectEvent`，優雅解決了「effect 要讀最新值但不想加到 deps」的問題：

```javascript
function Page({ url }) {
  const { shoppingCart } = useContext(CartContext);

  const onVisit = useEffectEvent((visitedUrl) => {
    logVisit(visitedUrl, shoppingCart.length);
    // 永遠讀到最新的 shoppingCart，不需要加到 deps
  });

  useEffect(() => {
    onVisit(url);
  }, [url]);  // 只有 url 變化才重跑
}
```

`useEffectEvent` 建立的函式有穩定的 reference，但內部永遠讀最新的 props/state。ESLint plugin 也知道不需要把它放進 dependency array。這是 React 團隊承認「有些東西不應該是 reactive 的」之後給出的正式答案。

---

## 你現在該做什麼

1. **不要拿掉 StrictMode。** 如果你的 effect 在雙重觸發下壞了，問題在你的 effect，不在 StrictMode。修好 cleanup。

2. **每個 effect 都要有對應的 cleanup。** 訂閱？退訂。連線？斷開。Timer？清除。event listener？移除。沒有例外。

3. **dependency array 不是 performance optimization。** 它是語義契約——告訴 React「這個 effect 依賴哪些值」。不要用 `eslint-disable` 壓掉警告，那是在自找 bug。

4. **物件和陣列不要直接當 deps。** 展開成原始值，或把建立邏輯移進 effect 裡面。

5. **升級 React 19 後試試 `useEffectEvent`。** 它是 stale closure 問題的正規解法，比 ref workaround 乾淨太多。

React 的 effect 系統看起來簡單（就是一個函式加一個 array 嘛），但底層的環狀鏈表、位元旗標、兩輪清掃、非同步排程——每一個設計決策都有它的理由。搞懂這些，你用 `useEffect` 的時候就不是在背規則，而是真的理解它為什麼這樣運作。

---

## 延伸閱讀

- [useEffect — React 官方文件](https://react.dev/reference/react/useEffect) — 用法和最佳實踐的權威來源
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) — 官方文件中最值得讀的一頁，很多 effect 其實不該存在
- [Adding Reusable State to StrictMode (reactwg/react-18#19)](https://github.com/reactwg/react-18/discussions/19) — React 團隊解釋 Strict Mode 雙重觸發的原始討論
- [How does useEffect work internally (jser.dev)](https://jser.dev/2023-07-08-how-does-useeffect-work/) — 從原始碼角度分析 commit phase 的 effect 處理
- [Under the hood of React's hooks system (The Guild)](https://the-guild.dev/blog/react-hooks-system) — Hook 鏈表和 dispatcher 機制的深度解析
- [ReactFiberHooks.js 原始碼](https://github.com/facebook/react/blob/v18.3.1/packages/react-reconciler/src/ReactFiberHooks.new.js) — 文中提到的 mountEffect / updateEffect 實作所在
