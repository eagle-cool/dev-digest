---
title: "React Fiber 架構深度解析：從遞迴呼叫堆疊到可中斷的鏈表革命"
date: 2026-02-22
description: "深入 React Fiber 的內部實現：為什麼要從 Stack Reconciler 換成 Fiber？FiberNode 的資料結構長什麼樣？雙緩衝、Lane 優先級、可中斷的 work loop 到底怎麼運作？帶你看原始碼級別的核心機制。"
tags: [deep-dive, react, frontend]
---

你有沒有想過一個問題：React 15 和 React 16 的 API 幾乎一模一樣，`setState` 還是 `setState`，`render` 還是 `render`——但 React 團隊花了兩年時間重寫了整個核心引擎。為什麼？

答案藏在一個你每天都在用、但可能從沒搞懂的東西裡：**Reconciliation（協調）**。

React 15 的 reconciler 有一個致命缺陷——它是**同步遞迴**的。一旦開始 diff，就必須跑完整棵樹才能還控制權給瀏覽器。想像你有一個 10,000 行的列表要過濾，React 15 會死死抱著主執行緒不放，動畫卡死、輸入延遲、使用者體驗直接崩盤。React 16 的 Fiber 就是為了解決這個問題而生的——把遞迴改成可中斷的鏈表遍歷，讓 React 學會「暫停」。

今天我們就來拆開 Fiber 的引擎蓋，看看裡面到底裝了什麼。

---

## 從 Stack 到 Fiber：為什麼非改不可

React 15 的 Stack Reconciler，顧名思義，直接利用 JavaScript 的呼叫堆疊來遍歷元件樹。虛擬碼大概長這樣：

```javascript
function reconcile(element, container) {
  const component = instantiateComponent(element);
  const renderedElement = component.render();

  // 遞迴——一旦進去就出不來
  for (const child of renderedElement.children) {
    reconcile(child, container);  // 堆疊越推越深
  }

  commitToDom(component, container);
}
```

問題顯而易見：**遞迴不能暫停**。JavaScript 的呼叫堆疊是引擎管的，你沒辦法跑到一半說「等等，先讓瀏覽器畫個動畫再回來」。堆疊裡的所有狀態（區域變數、回傳位址）都綁在引擎的記憶體裡，你碰不到。

Andrew Clark 在他的 [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture) 文件中寫了一句話，精準到位：

> "Fiber is a reimplementation of the stack, specialized for React components. You can think of a single fiber as a **virtual stack frame**."

Fiber 的核心思路就是：**不用 JavaScript 的呼叫堆疊，自己做一個**。把每個「堆疊幀」變成一個物件（FiberNode），存在記憶體裡，想什麼時候執行就什麼時候執行，想暫停就暫停。

---

## FiberNode：React 的虛擬堆疊幀

每個 React 元件——不管是 `<div>` 還是 `<App />`——在 Fiber 架構下都對應一個 FiberNode。讓我們看看 React 原始碼中 `FiberNode` 的關鍵欄位（來自 `packages/react-reconciler/src/ReactFiber.js`）：

```javascript
function FiberNode(tag, pendingProps, key, mode) {
  // === 身份識別 ===
  this.tag = tag;            // 元件類型：FunctionComponent(0)、ClassComponent(1)、HostComponent(5)...
  this.type = null;          // 對應的函式、類別、或 DOM 標籤字串
  this.key = key;            // 列表 diff 用的 key
  this.stateNode = null;     // 實際的 DOM 節點或 class 實例

  // === 鏈表指標（核心中的核心）===
  this.return = null;        // 父節點（為什麼叫 return？因為處理完要「返回」父節點）
  this.child = null;         // 第一個子節點
  this.sibling = null;       // 下一個兄弟節點

  // === Props 與 State ===
  this.pendingProps = pendingProps;  // 新的 props（render 開始時設定）
  this.memoizedProps = null;        // 上次 render 完成的 props
  this.memoizedState = null;        // 上次 render 完成的 state（function component 的 hooks 鏈表也存這裡）

  // === 副作用標記 ===
  this.flags = NoFlags;             // 位元遮罩：Placement、Update、Deletion...
  this.subtreeFlags = NoFlags;      // 子樹的聚合標記

  // === 排程優先級 ===
  this.lanes = NoLanes;             // 這個 fiber 上的待處理工作優先級
  this.childLanes = NoLanes;        // 子樹中的待處理工作優先級

  // === 雙緩衝 ===
  this.alternate = null;            // 指向另一棵樹中的對應 fiber
}
```

注意那三個指標：`child`、`sibling`、`return`。這不是傳統的樹結構（parent + children 陣列），而是一個**鏈表**。這個設計讓 React 可以用迴圈代替遞迴來遍歷整棵樹：

```
        HostRoot
          │
          │ child
          ▼
        <App>
          │
          │ child
          ▼
        <div> ──sibling──▶ <aside>
          │                   │
          │ child              │ child
          ▼                   ▼
        <Header>            <Nav>
          │
          │ child
          ▼
        <h1>
```

每個節點都知道自己的孩子、兄弟、和父親。遍歷演算法是：往下找 `child`，沒有 `child` 就找 `sibling`，沒有 `sibling` 就沿 `return` 回到父節點。全程只需要一個 `workInProgress` 指標，不需要呼叫堆疊。

**這就是 Fiber 能暫停的根本原因**——當前工作位置存在一個變數裡，而不是在呼叫堆疊裡。隨時可以暫停、隨時可以繼續。

---

## 兩階段渲染：Render 與 Commit

Fiber 把工作分成兩個截然不同的階段：

### Render Phase（可中斷）

這個階段做的事情是：遍歷 fiber 樹，呼叫元件函式，比對新舊 props，標記需要更新的節點。**不碰 DOM**。因為沒有副作用，所以可以隨時暫停、甚至丟棄重做。

核心是 work loop，來自 `ReactFiberWorkLoop.js`：

```javascript
// 同步模式——跑到底
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// Concurrent 模式——會喘氣
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

差別就在 `shouldYield()`。每處理一個 fiber 之前檢查一次：5ms 的時間配額用完了嗎？如果用完了，把控制權還給瀏覽器，讓它去處理動畫、使用者輸入、畫面重繪。

`performUnitOfWork` 是每個工作單元的入口：

```javascript
function performUnitOfWork(unitOfWork) {
  // beginWork：處理這個 fiber，回傳第一個子節點（或 null）
  let next = beginWork(unitOfWork);
  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // 沒有子節點了，回溯
    completeUnitOfWork(unitOfWork);
  } else {
    // 繼續往下處理子節點
    workInProgress = next;
  }
}
```

`beginWork` 根據 fiber 的 `tag` 做不同的事：FunctionComponent 就呼叫函式、ClassComponent 就呼叫 `render()`、HostComponent 就比對 props。它回傳第一個需要處理的子 fiber。

`completeUnitOfWork` 負責回溯：

```javascript
function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;
  do {
    completeWork(completedWork);  // 建立 DOM 節點、收集副作用

    if (completedWork.sibling !== null) {
      workInProgress = completedWork.sibling;  // 處理兄弟節點
      return;
    }
    completedWork = completedWork.return;  // 沒有兄弟就回到父節點
    workInProgress = completedWork;
  } while (completedWork !== null);
}
```

用一個具體的例子走一遍。假設元件樹是 `App > div > [Header, Content]`：

```
1. beginWork(HostRoot)    → 回傳 App
2. beginWork(App)         → 呼叫 App()，回傳 div
3. beginWork(div)         → 回傳 Header（第一個子節點）
4. beginWork(Header)      → 呼叫 Header()，回傳 h1
5. beginWork(h1)          → 葉子節點，回傳 null
6. completeWork(h1)       → 建立 DOM <h1>
7. ← h1 沒有 sibling，回溯到 Header
8. completeWork(Header)   → Header 有 sibling → 跳到 Content
9. beginWork(Content)     → 處理 Content 子樹...
10. completeWork(Content)
11. completeWork(div)
12. completeWork(App)
13. completeWork(HostRoot) → Render phase 結束
```

**在 concurrent 模式下，任何兩步之間 `shouldYield()` 都可能回傳 `true`**，React 就會暫停，把 `workInProgress` 存好，等下次排程器安排時間再繼續。

### Commit Phase（不可中斷）

Render phase 結束後，React 手上有一棵標記好所有變更的 fiber 樹。Commit phase 負責把這些變更一口氣刷到 DOM 上：

```javascript
function commitRoot(root, finishedWork) {
  // 階段一：突變前（getSnapshotBeforeUpdate）
  commitBeforeMutationEffects();

  // 階段二：DOM 操作（插入、更新、刪除）
  commitMutationEffects();

  // 關鍵操作：交換樹的指標
  root.current = finishedWork;

  // 階段三：突變後（componentDidMount/Update、useEffect）
  commitLayoutEffects();
}
```

**Commit phase 必須同步執行，不能中斷。** 原因很簡單：如果 DOM 更新做到一半暫停，使用者會看到不一致的畫面。那個 `root.current = finishedWork` 是把「工作中的樹」升級為「當前樹」，這就帶出了下一個重要機制。

---

## 雙緩衝：current 與 workInProgress

React 同時維護**兩棵 fiber 樹**，概念類似於圖形學中的雙緩衝（double buffering）：

```
   current 樹（畫面上的）          workInProgress 樹（正在建構的）
   ┌──────────┐    alternate     ┌──────────┐
   │ Fiber A   │ ◄──────────────► │ Fiber A'  │
   └──────────┘                  └──────────┘
       │ child                        │ child
   ┌──────────┐    alternate     ┌──────────┐
   │ Fiber B   │ ◄──────────────► │ Fiber B'  │
   └──────────┘                  └──────────┘
```

- **current 樹**：代表目前畫面上的內容。`fiberRoot.current` 指向它的根節點。
- **workInProgress 樹**：render phase 正在建構的新版本。每個 fiber 的 `alternate` 欄位指向另一棵樹中的對應節點。

當 commit phase 完成，`root.current = finishedWork` 一行程式碼就把 workInProgress 樹變成新的 current 樹。下次更新時，老的 current 樹的 fiber 節點會被**回收再利用**成新的 workInProgress 節點（透過 `createWorkInProgress` 函式），避免大量記憶體分配。

這個設計很聰明：如果 render phase 被中斷或丟棄，current 樹完全不受影響——畫面保持穩定。只有 commit phase 成功完成，才會切換指標。

---

## 排程器與 Lane 優先級模型

Fiber 讓 React 能暫停工作，但「什麼時候暫停、什麼工作先做」——這是排程器（Scheduler）的責任。

### 為什麼不用 requestIdleCallback？

React 最初考慮過 `requestIdleCallback`，但最後放棄了：

1. **頻率太低**：Chrome 的 `requestIdleCallback` 每秒最多觸發約 20 次，遠遠不夠 UI 更新用。
2. **依賴硬體**：它綁定螢幕更新率和 Vsync 週期。
3. **Safari 不支援**：直到現在。

React 自己的排程器用 **MessageChannel** 來排程工作：

```javascript
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

schedulePerformWorkUntilDeadline = () => {
  port.postMessage(null);
};
```

為什麼用 MessageChannel 而不是 `setTimeout`？因為 `setTimeout` 在巢狀呼叫超過 5 層後會被瀏覽器強制加上 4ms 延遲。MessageChannel 沒有這個限制，能讓 React 精確控制每個時間切片（預設 5ms）。

### Lane 模型：用位元遮罩管理優先級

React 18 之前用 `ExpirationTime`（過期時間）來管理更新優先級，但這個模型有根本缺陷——它把「優先級排序」和「批次處理」耦合在一個數值裡。Lane 模型（[PR #18796](https://github.com/facebook/react/pull/18796)）用 31 位元的位元遮罩解決了這個問題：

```javascript
const SyncLane            = 0b0000000000000000000000000000001;  // 最高優先級
const InputContinuousLane = 0b0000000000000000000000000000100;  // 連續輸入（拖曳、滾動）
const DefaultLane         = 0b0000000000000000000000000010000;  // 一般 setState
const TransitionLane1     = 0b0000000000000000000000001000000;  // startTransition
const TransitionLane2     = 0b0000000000000000000000010000000;  // startTransition
// ...總共 16 個 TransitionLane
const IdleLane            = 0b0100000000000000000000000000000;  // 閒置時才處理
```

位元遮罩的好處是操作極快且表達力強：

```javascript
// 檢查某個 lane 是否在集合中
function includesSomeLane(set, subset) { return (set & subset) !== 0; }

// 合併 lanes
function mergeLanes(a, b) { return a | b; }

// 移除 lanes
function removeLanes(set, subset) { return set & ~subset; }
```

當你呼叫 `startTransition(() => setState(...))`，React 會把這個更新分配到 TransitionLane。排程器在決定渲染時會判斷：

```javascript
const shouldTimeSlice =
  !includesBlockingLane(root, lanes) &&
  !includesExpiredLane(root, lanes);

let exitStatus = shouldTimeSlice
  ? renderRootConcurrent(root, lanes)  // 可中斷
  : renderRootSync(root, lanes);        // 同步執行
```

SyncLane 和 InputContinuousLane 的更新**不會被時間切片**，它們永遠同步跑完。只有 TransitionLane 以下的優先級才會用 concurrent 模式。這就是為什麼普通的 `setState` 在事件處理器中仍然是同步批次處理的——它走的是 SyncLane。

---

## 你該知道的常見誤解

**「Fiber 自動讓所有更新都變成 concurrent」**——錯。React 16 雖然用了 Fiber 架構，但所有更新仍然同步執行。要真正啟用 concurrent 特性，你需要 React 18 的 `createRoot` API，然後用 `startTransition` 或 `useDeferredValue` 明確標記哪些更新可以被中斷。

**「Babel 編譯出 Fiber 物件」**——錯。Babel 把 JSX 編譯成 `React.createElement()`（或現代的 JSX transform），產出的是 **React Element**（普通的 JS 物件）。Fiber 節點是**執行期**由 reconciler 建立的，不是編譯期的產物。

**「Virtual DOM diff 讓 React 很快」**——這話要小心。手動精確操作 DOM 永遠比任何 diff 演算法快。Fiber 的價值不在於「快」，而在於讓你用**宣告式的方式**寫 UI，同時透過可中斷的、有優先級的 reconciliation 維持**可接受的效能**。這是一個工程上的 trade-off，不是魔法。

**「Render phase 只執行一次元件函式」**——在 concurrent 模式下，React 可能會多次呼叫你的 render 函式（如果工作被中斷並重新開始）。這就是為什麼 React 嚴格禁止在 render 中放副作用，也是為什麼 Strict Mode 會故意 double-invoke——幫你抓出不純的 render。

---

## 理解 Fiber 之後，你該做什麼

1. **善用 `startTransition`**：它不是可選的語法糖——它是告訴排程器「這個更新不急，可以被中斷」的正式管道。大量資料的搜尋過濾、頁面切換，都該包在 `startTransition` 裡。

2. **保持 render 純淨**：Fiber 的可中斷性建立在一個前提上——render phase 沒有副作用。如果你在 render 中發 API 請求或修改全域變數，concurrent mode 會讓你哭出來。

3. **別怕 re-render**：Fiber 的整個設計就是讓 re-render 成本夠低。與其花時間過度 `useMemo` 每一個計算，不如確保你的元件樹結構合理、props 穩定。React Compiler（Forget）未來會自動處理大部分 memoization。

4. **key 的重要性被低估了**：Reconciliation 的 list diff 嚴重依賴 `key`。錯誤的 key（比如用 index）會導致不必要的卸載和重建，Fiber 再怎麼優化也救不了你。

Fiber 不是什麼高深的概念——它就是一個把遞迴改成迴圈的工程決策，加上一套優先級排程系統。但正是這個看似簡單的改變，讓 React 從一個「同步渲染引擎」進化成了「可中斷、有優先級的 UI 排程框架」。理解這個底層，你才能真正用好 `startTransition`、`Suspense`、`useDeferredValue` 這些建立在 Fiber 之上的 API。

---

## 延伸閱讀

- [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture) — Andrew Clark 撰寫的 Fiber 架構設計文件，理解 Fiber 的起點
- [React 16: A Look Inside the Rewrite (Meta Engineering)](https://engineering.fb.com/2017/09/26/web/react-16-a-look-inside-an-api-compatible-rewrite-of-our-frontend-ui-library/) — Facebook 工程團隊對 Fiber 重寫過程的第一手記錄
- [Inside Fiber: In-depth overview of the new reconciliation algorithm](https://blog.ag-grid.com/inside-fiber-an-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/) — 非常詳盡的兩階段渲染流程解析
- [What are Lanes in React source code?](https://jser.dev/react/2022/03/26/lanes-in-react) — 對 Lane 模型的完整原始碼級分析
- [React Source: ReactFiber.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiber.js) — FiberNode 建構子的原始碼
- [React Source: ReactFiberWorkLoop.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.js) — work loop 和 render/commit 階段的實現
- [Initial Lanes Implementation PR #18796](https://github.com/facebook/react/pull/18796) — Andrew Clark 提交的 Lanes PR，包含完整的設計理由
