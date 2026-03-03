---
title: "React Error Boundary 完全拆解：從 Fiber 原始碼看錯誤捕獲、回退與恢復機制"
date: 2026-03-03
description: "深入 React 原始碼解析 Error Boundary 的完整運作機制：throwException 如何沿 Fiber 樹向上搜尋邊界、getDerivedStateFromError 與 componentDidCatch 的雙階段設計、為什麼只能用 class component，以及 React 19 的錯誤處理改進。"
tags: [deep-dive, react, error-handling, frontend]
---

你的 React 應用在生產環境白屏了。用戶看到一片空白，控制台噴了一堆紅字，然後——整個元件樹直接消失。

如果你寫過 React，大概聽過 Error Boundary。可能還包過一層，覺得「好了，錯誤處理搞定了」。但你有沒有想過：Error Boundary 到底怎麼「攔截」到子元件的錯誤的？為什麼它只能是 class component，都 2026 年了還不給 Hook？React 內部又是怎麼做到「把壞掉的子樹砍掉、換上 fallback UI」這件事的？

今天我們直接翻 React 原始碼，把 Error Boundary 的完整生命週期——從錯誤拋出、Fiber 樹回溯、到恢復渲染——一次拆個乾淨。

---

## 從 React 15 的全面崩壞說起

React 16 之前，元件樹裡任何一個 render 錯誤都是災難。一個子元件炸了，整個應用直接白屏。React 團隊在 2017 年的 [Error Handling in React 16](https://legacy.reactjs.org/blog/2017/07/26/error-handling-in-react-16.html) 文章裡明確說了：

> As of React 16, errors that were not caught by any error boundary will result in unmounting of the whole React component tree.

這個設計哲學很直接：**壞掉的 UI 留在畫面上比完全移除更危險**。想像一個支付頁面，金額顯示元件出錯但畫面沒崩——用戶可能看到錯誤的金額然後點了確認。與其冒這個風險，不如直接讓整棵樹消失。

但「全部消失」顯然不是好的用戶體驗。所以 React 16 引入了 Error Boundary：一種特殊的元件，能捕獲子樹的 render 錯誤，並渲染 fallback UI，而不是讓整個應用掛掉。

問題是——它怎麼做到的？

---

## Error Boundary 的六步內部流程

React 的錯誤處理機制分散在三個核心檔案裡：

- **`ReactFiberThrow.js`** — `throwException()`：沿 Fiber 樹向上搜尋 Error Boundary
- **`ReactFiberUnwindWork.js`** — `unwindWork()`：轉換旗標、清理中斷的工作
- **`ReactFiberWorkLoop.js`** — Work Loop 的 try-catch 入口

讓我們跟著一個錯誤，從拋出到恢復，走完完整的六步流程。

### Step 1：Work Loop 捕獲錯誤

React 的渲染引擎（work loop）在處理每個 Fiber 節點時，用 try-catch 包住了 `beginWork()` 和 `completeWork()` 的呼叫：

```javascript
// ReactFiberWorkLoop.js（簡化）
do {
  try {
    if (isSync) {
      workLoopSync();
    } else {
      workLoopConcurrent();
    }
    break;
  } catch (thrownValue) {
    handleThrow(root, thrownValue);
  }
} while (true);
```

當某個子元件在 render 階段拋出錯誤（`throw new Error('boom')`），這個 try-catch 會接住它。注意：React 的 work loop 是一個迴圈，catch 之後不是直接 break——它會處理完錯誤，然後繼續迴圈，從 Error Boundary 重新開始渲染。

### Step 2：`throwException()` 搜尋 Error Boundary

接住錯誤後，React 呼叫 `throwException()`。這個函式做的事情很明確：**沿著 Fiber 樹的 `return` 指針（父節點方向）往上走，找到第一個 Error Boundary**。

```javascript
// ReactFiberThrow.js（簡化）
function throwException(root, returnFiber, sourceFiber, value, rootRenderLanes) {
  // 標記出錯的 fiber 為「未完成」
  sourceFiber.flags |= Incomplete;

  // 沿著 return 指針往上走
  let workInProgress = returnFiber;
  do {
    switch (workInProgress.tag) {
      case ClassComponent: {
        const ctor = workInProgress.type;
        const instance = workInProgress.stateNode;
        if (
          typeof ctor.getDerivedStateFromError === 'function' ||
          typeof instance.componentDidCatch === 'function'
        ) {
          // 找到了！這是一個 Error Boundary
          const errorInfo = createCapturedValueAtFiber(value, sourceFiber);
          const update = createClassErrorUpdate(workInProgress, errorInfo, lane);
          enqueueCapturedUpdate(workInProgress, update);
          workInProgress.flags |= ShouldCapture;
          return;
        }
        break;
      }
      case HostRoot: {
        // 走到根節點了，沒有任何 Boundary 能接——未捕獲的錯誤
        // ...
      }
    }
    workInProgress = workInProgress.return;
  } while (workInProgress !== null);
}
```

關鍵細節：判斷一個 class component 是不是 Error Boundary 的標準，就是檢查它有沒有 `getDerivedStateFromError` 或 `componentDidCatch`。就這麼簡單——不需要繼承特殊基類，不需要呼叫 `super.enableErrorBoundary()`，有這兩個方法就行。

### Step 3：建立 CaptureUpdate

找到 Error Boundary 後，`createClassErrorUpdate()` 建立一個特殊的 update 物件。這裡有個精妙的雙階段設計：

```javascript
// ReactFiberThrow.js（簡化）
function createClassErrorUpdate(fiber, errorInfo, lane) {
  const update = createUpdate(lane);
  update.tag = CaptureUpdate; // 特殊標記：這是錯誤捕獲更新

  // getDerivedStateFromError → 放在 payload（render 階段執行）
  const getDerivedStateFromError = fiber.type.getDerivedStateFromError;
  if (typeof getDerivedStateFromError === 'function') {
    const error = errorInfo.value;
    update.payload = () => getDerivedStateFromError(error);
  }

  // componentDidCatch → 放在 callback（commit 階段執行）
  const inst = fiber.stateNode;
  if (inst !== null && typeof inst.componentDidCatch === 'function') {
    update.callback = function callback() {
      this.componentDidCatch(errorInfo.value, {
        componentStack: errorInfo.stack,
      });
    };
  }

  return update;
}
```

這裡值得停下來仔細看：

- **`update.payload`** 放的是 `getDerivedStateFromError`。它會在 **render 階段**被 `processUpdateQueue()` 處理，是同步且純粹的——只能回傳新的 state。
- **`update.callback`** 放的是 `componentDidCatch`。它會在 **commit 階段**被呼叫，可以執行副作用（例如把錯誤送到 Sentry）。

兩個生命週期方法，兩個不同的時機，兩個不同的用途。這不是 API 設計的怪癖——這是 React 雙階段架構（render + commit）的直接體現。

### Step 4：`Incomplete` 旗標向上傳播

`throwException()` 返回後，`completeUnitOfWork()` 接手。它從出錯的 fiber 開始，沿著 `return` 指針往上走，把每個經過的 fiber 都標記為 `Incomplete`：

```
<ErrorBoundary>           ← 目標（有 ShouldCapture）
  <Layout>                ← Incomplete
    <Sidebar>             ← Incomplete
      <BrokenWidget />    ← Incomplete（錯誤源頭）
```

這些 `Incomplete` 標記告訴 React：「這些 fiber 的工作進度作廢了，別用。」

### Step 5：`unwindWork()` 轉換旗標

當 `completeUnitOfWork()` 走到 Error Boundary 那個 fiber 時，呼叫 `unwindWork()`：

```javascript
// ReactFiberUnwindWork.js（簡化）
function unwindWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case ClassComponent: {
      const flags = workInProgress.flags;
      if (flags & ShouldCapture) {
        // 關鍵轉換：ShouldCapture → DidCapture
        workInProgress.flags = (flags & ~ShouldCapture) | DidCapture;
        return workInProgress; // 返回這個 fiber，讓 work loop 從這裡重新開始
      }
      return null;
    }
  }
}
```

`ShouldCapture` → `DidCapture` 這個旗標轉換看起來微不足道，但它是整個恢復機制的樞紐。`unwindWork()` 返回這個 fiber，讓 work loop 知道：「從這裡重新開始渲染。」

### Step 6：Error Boundary 重新渲染

Work loop 拿到返回的 fiber，對它重新執行 `beginWork()`。在 `finishClassComponent()` 裡，React 檢測到 `DidCapture` 旗標：

```javascript
// ReactFiberBeginWork.js（簡化）
if (didCaptureError) {
  // 強制卸載現有子元件，從頭開始 reconcile
  forceUnmountCurrentAndReconcile(current, workInProgress, nextChildren, renderLanes);
}
```

在這之前，`processUpdateQueue()` 處理了那個 `CaptureUpdate`：呼叫 `getDerivedStateFromError(error)` 拿到新 state（例如 `{ hasError: true }`），然後元件的 `render()` 方法用這個新 state 渲染 fallback UI。

整個流程的 ASCII 圖：

```
Error thrown in <BrokenChild />
        │
        ▼
try-catch in workLoop catches it
        │
        ▼
throwException() walks UP the fiber tree
        │
        ├─ Finds ErrorBoundary with getDerivedStateFromError
        ├─ Creates CaptureUpdate (payload + callback)
        └─ Sets ShouldCapture flag
        │
        ▼
completeUnitOfWork() marks fibers as Incomplete
        │
        ▼
unwindWork() converts ShouldCapture → DidCapture
        │
        ▼
Work loop resumes at ErrorBoundary fiber
        │
        ├─ processUpdateQueue → getDerivedStateFromError(error) → new state
        ├─ render() with { hasError: true } → fallback UI
        └─ forceUnmountCurrentAndReconcile → clean slate for children
        │
        ▼
Commit phase: componentDidCatch(error, info) called
```

---

## 為什麼 Error Boundary 只能是 Class Component

這大概是 React 社群問了七年的問題。都 2026 年了，Hook 什麼都能做，為什麼就是沒有 `useErrorBoundary` 這個 Hook？

答案不是「React 團隊懶」，而是有三個硬性的架構限制。

### 1. Render 階段的同步攔截

`getDerivedStateFromError` 必須在 **render 階段同步執行**——React 需要在同一個 render pass 裡拿到 fallback state 並產生 fallback UI。

Hook 的 state 更新（`useState` 的 setter）是異步的，它排進更新佇列、等下一次 render 才生效。沒有任何 Hook 能在「當前這次 render 正在執行的當下」同步修改 state 並重新產出 JSX。

### 2. 錯誤源的辨識問題

看 `throwException()` 的程式碼——它檢查的是 `workInProgress.tag === ClassComponent`。Class component 有 `stateNode`（class 實例），這個實例在整個生命週期中持久存在。

Function component 沒有實例。React 呼叫你的函式、拿到 JSX、然後就沒了。如果 function component 自己的 render 拋出錯誤，React 在 catch 的時候，需要在同一個 render pass 裡用不同的 state 重新呼叫這個函式——但 hooks 的狀態管理機制（`memoizedState` 鏈表）不支援 `CaptureUpdate` 這種特殊更新類型。

### 3. RFC #126 的結局

社群在 2019 年提出了 [RFC #126](https://github.com/reactjs/rfcs/pull/126)，建議用 HOC 的方式支援 function component error boundary。Dan Abramov 回覆說錯誤處理可能會和更廣泛的控制流特性整合。Andrew Clark 在 2021 年直接關了這個 RFC，原因很務實：既然 `react-error-boundary` 這個第三方庫已經把 DX 包得很好了，React 核心團隊沒有動力去改底層架構。

說白了：不是做不到，是 cost-benefit 不划算。class component 在這個場景下工作得很好，而你只需要寫一次 Error Boundary 元件（或者直接用 `react-error-boundary`）。

---

## 哪些錯誤 Error Boundary 抓不到

這部分很多人踩坑，所以講清楚。

### Event Handler 裡的錯誤

```jsx
function MyComponent() {
  function handleClick() {
    throw new Error('boom'); // Error Boundary 抓不到
  }
  return <button onClick={handleClick}>Click</button>;
}
```

原因：React 在 render 階段呼叫你的元件函式時，外面包了 try-catch。但 event handler 是在 **commit 之後** 由 DOM 事件系統觸發的，跟 React 的 render try-catch 完全沒關係。而且——event handler 出錯時，UI 已經在畫面上了，並沒有「壞掉的 UI 需要替換」的問題。

### 非同步程式碼

```jsx
useEffect(() => {
  setTimeout(() => {
    throw new Error('async boom'); // 抓不到
  }, 1000);
}, []);
```

`setTimeout` 和 Promise rejection 跑在不同的 microtask/macrotask 上。等它們拋出時，React 的 render phase 早就結束了。錯誤會跑到全域的 `unhandledrejection` handler，而不是 React 的 work loop catch。

### Error Boundary 自身的錯誤

如果 Error Boundary 的 `render()` 方法本身就拋出錯誤呢？不好意思，這個錯誤會繼續往上傳播到 **更上層的** Error Boundary。自己不能接自己的錯——不然會無限迴圈。

在原始碼裡，`throwException()` 的搜尋是從 `returnFiber`（父 fiber）開始的，不是從出錯的 fiber 自身開始。所以 Error Boundary 如果是錯誤源頭，搜尋會跳過自己。

### SSR

React 的 server renderer（`renderToString` / `renderToPipeableStream`）用的是完全不同的渲染管線，不走 Fiber work loop。Error Boundary 的整個機制依賴 Fiber 的 `completeUnitOfWork` 和 unwind 流程，在 server 端不存在。React 18+ 的 streaming SSR 有自己的錯誤處理（Suspense boundary 層級），但那是另一回事。

---

## React 19 的錯誤處理改進

Error Boundary 的核心 API（`getDerivedStateFromError` / `componentDidCatch`）在 React 19 **完全沒變**。改變的是圍繞在它外面的錯誤報告機制。

### `createRoot` 的三個錯誤回呼

```javascript
const root = createRoot(document.getElementById('root'), {
  // Error Boundary 接住的錯誤
  onCaughtError(error, errorInfo) {
    Sentry.captureException(error, {
      extra: { componentStack: errorInfo.componentStack },
    });
  },

  // 沒有任何 Error Boundary 接住的錯誤
  onUncaughtError(error, errorInfo) {
    Sentry.captureException(error, { level: 'fatal' });
    showGlobalErrorOverlay(error);
  },

  // React 自動恢復的錯誤（例如 hydration mismatch）
  onRecoverableError(error, errorInfo) {
    console.warn('React recovered from:', error);
  },
});
```

這三個回呼讓你在應用層級統一處理錯誤報告，不需要在每個 Error Boundary 的 `componentDidCatch` 裡寫 Sentry 邏輯。

### 告別三重 console.error

React 19 之前，一個被 Error Boundary 捕獲的錯誤會在控制台出現 **三次**：原始錯誤、自動恢復嘗試失敗的重複錯誤、React 自己印的 `console.error`。React 19 整合成一次清楚的輸出，包含完整的 component stack。

---

## 實戰：錯誤邊界的佈局策略

理解了內部機制，來聊聊怎麼用。

### 別只包一層

最常見的錯誤：在 `<App>` 外面包一層 Error Boundary，然後覺得搞定了。這跟沒包差不多——因為任何錯誤都會讓整個應用 fallback 成一個「出錯了」的畫面。

正確做法是 **按功能區域** 佈置 Error Boundary：

```jsx
function App() {
  return (
    <Layout>
      <ErrorBoundary fallback={<SidebarFallback />}>
        <Sidebar />
      </ErrorBoundary>
      <ErrorBoundary fallback={<MainContentFallback />}>
        <MainContent />
      </ErrorBoundary>
      <ErrorBoundary fallback={<ChatFallback />}>
        <ChatWidget />
      </ErrorBoundary>
    </Layout>
  );
}
```

Sidebar 掛了，主要內容照常顯示。聊天小工具炸了，其他功能不受影響。這才是 Error Boundary 的正確用法——**錯誤隔離**，不是全域保險。

### 跟 Suspense 配對

Error Boundary 和 Suspense 是天生的搭配——一個處理「載入中」，一個處理「載入失敗」：

```jsx
<ErrorBoundary fallback={<DataError />}>
  <Suspense fallback={<DataLoading />}>
    <AsyncDataComponent />
  </Suspense>
</ErrorBoundary>
```

Error Boundary 放在 Suspense **外面**。因為 Suspense 的 fallback 只處理 pending 狀態，如果 Promise reject 了，那個錯誤需要被 Error Boundary 接住。

### 用 `react-error-boundary` 省事

實務上很少人手寫 class component 的 Error Boundary。[react-error-boundary](https://github.com/bvaughn/react-error-boundary) 把所有常見需求都封裝好了：

```jsx
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary';

// 帶重試按鈕的 fallback
<ErrorBoundary
  fallbackRender={({ error, resetErrorBoundary }) => (
    <div>
      <p>出錯了：{error.message}</p>
      <button onClick={resetErrorBoundary}>重試</button>
    </div>
  )}
  onReset={() => {
    // 重置你的應用狀態
    queryClient.clear();
  }}
  resetKeys={[userId]} // userId 變了自動重置
>
  <UserProfile />
</ErrorBoundary>
```

它的殺手級特性是 `useErrorBoundary` Hook——讓你在 function component 裡手動觸發 Error Boundary，解決 event handler 和 async 錯誤抓不到的問題：

```jsx
function DataFetcher() {
  const { showBoundary } = useErrorBoundary();

  async function handleFetch() {
    try {
      const data = await fetchData();
      setData(data);
    } catch (error) {
      showBoundary(error); // 手動觸發最近的 Error Boundary
    }
  }

  return <button onClick={handleFetch}>載入資料</button>;
}
```

`showBoundary` 的原理很簡單：在 state 裡存下 error，觸發 re-render，在 render 階段把 error 拋出去——於是 Error Boundary 的 try-catch 就能接住了。繞了一圈，但管用。

---

## 結論：Error Boundary 是 Fiber 架構的自然產物

Error Boundary 不是什麼魔法——它是 React Fiber 架構 try-catch → 回溯搜尋 → 旗標轉換 → 重新渲染 這整個流程的具體應用。理解了 Fiber 的 work loop，Error Boundary 的行為就全都說得通了。

幾個明確的建議：

1. **按功能區域佈置 Error Boundary**，不要只在根節點包一層。
2. **`getDerivedStateFromError` 負責 UI 回退，`componentDidCatch` 負責錯誤上報**——別搞混。
3. **用 `react-error-boundary` 庫**，除非你有很好的理由自己寫 class component。
4. **Event handler 和 async 的錯誤用 `useErrorBoundary` 的 `showBoundary` 處理**。
5. **React 19 的 `onCaughtError` / `onUncaughtError`** 是統一錯誤報告的好地方，善用它。

別再等「function component 的 Error Boundary Hook」了——RFC 在 2021 年就關了。Class component 在這個場景下不是技術債，而是唯一合理的實現方式。接受它，然後把精力花在真正重要的事情上：決定你的應用在哪裡該有邊界、出錯時該給用戶看什麼。

---

## 延伸閱讀

- [How does ErrorBoundary work internally in React?](https://jser.dev/2023-05-26-how-does-errorboundary-work/) — 最詳細的 Error Boundary 原始碼分析
- [Error Handling in React 16](https://legacy.reactjs.org/blog/2017/07/26/error-handling-in-react-16.html) — Dan Abramov 的原始公告，解釋設計哲學
- [React v19 Release Blog](https://react.dev/blog/2024/12/05/react-19) — React 19 錯誤處理改進說明
- [createRoot API Docs](https://react.dev/reference/react-dom/client/createRoot) — `onCaughtError`、`onUncaughtError`、`onRecoverableError` 官方文件
- [RFC #126: Error Boundaries for Function Components](https://github.com/reactjs/rfcs/pull/126) — 被關閉的 function component Error Boundary RFC
- [react-error-boundary](https://github.com/bvaughn/react-error-boundary) — Brian Vaughn 維護的實用封裝庫
- [ReactFiberThrow.js 原始碼](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberThrow.js) — `throwException()` 和 `createClassErrorUpdate()` 的實際實現
