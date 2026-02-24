---
title: "React Concurrent 排程器深度解析：Lane 模型、時間切片、與不阻塞 UI 的秘密"
date: 2026-02-24
description: "深入 React 原始碼解析 Concurrent Mode 的三大支柱：Lane bitmask 優先權模型如何取代 expirationTime、Scheduler 的 5ms 時間切片與 MessageChannel 協作排程、以及 startTransition 的中斷恢復機制。"
tags: [deep-dive, react, frontend]
---

你有沒有遇過這種場景：使用者在搜尋框打字，每按一個鍵就觸發一次 filter，清單有上千筆資料，然後——UI 卡了。輸入框的游標不動、鍵盤輸入吃掉、整個畫面像當機了一樣。

你知道原因：JavaScript 是單執行緒，React 的 render 一旦開始就停不下來，瀏覽器沒機會處理使用者輸入。你可能用 `debounce` 來緩解，或是搬出 `useMemo` 來減少計算量。但這些都是繞道——React 團隊想從根本上解決這個問題。

這就是 Concurrent Mode 存在的理由。而它背後的排程系統，是 React 過去五年最大的架構改造。今天我們要拆開引擎蓋，看看裡面到底裝了什麼。

---

## 從同步到可中斷：為什麼 React 需要排程器

React 16 引入 Fiber 架構（我們在[上一篇](https://eagle-cool.github.io/dev-digest/posts/react-fiber-reconciliation-deep-dive/)拆過了），把遞迴 render 改成可中斷的鏈表走訪。但 Fiber 只是**結構上**支持中斷——真正決定「什麼時候中斷、中斷後先做什麼」的，是排程器。

早期的 React 用 `expirationTime`（到期時間）來表達優先權：每個 update 都有一個 timestamp，越接近過期的越優先。聽起來很直覺，但 Andrew Clark 在 [PR #18796](https://github.com/facebook/react/pull/18796) 中指出了三個致命問題：

1. **優先權和批次綁死了。** expirationTime 是連續的數值，React 把「時間相近」的 update 自動 batch 在一起。但有時候你想 batch 兩個不相鄰的優先權，做不到。
2. **無法表達不連續的優先權集合。** 如果你有優先權 A、C 要一起處理但跳過 B，expirationTime 的線性模型搞不定。
3. **CPU 工作被 IO 卡住。** 當一個 Suspense 在等資料（IO-bound），同優先權的 CPU 工作也被擋住了，因為它們共享同一個 expirationTime。

解法？Lane 模型。用 bitmask 取代 timestamp。

## Lane 模型：用 32 個位元表達優先權的藝術

Lane 的核心思想極其優雅：**一個 32-bit 整數，每個 bit 代表一個 lane（車道）**。不同的 update 被分配到不同的 lane，React 用 bitwise 運算來合併、比較、挑選要處理的工作。

來看 `ReactFiberLane.js` 裡的定義（簡化版）：

```javascript
// 每個 Lane 都是一個只有單一 bit 為 1 的整數
const SyncLane           = 0b0000000000000000000000000000010;  // 2
const InputContinuousLane = 0b0000000000000000000000000001000; // 8
const DefaultLane        = 0b0000000000000000000000000100000;  // 32
const TransitionLane1    = 0b0000000000000000000000010000000;  // 128
const TransitionLane2    = 0b0000000000000000000000100000000;  // 256
// ... 一共 16 條 TransitionLane
const IdleLane           = 0b0010000000000000000000000000000;
const OffscreenLane      = 0b0100000000000000000000000000000;
```

為什麼這比 expirationTime 好？因為 bitmask 天然支持**集合運算**：

```javascript
// 合併多個 lane（一個 update 可以屬於多個 lane）
const merged = SyncLane | DefaultLane;

// 檢查某個 lane 是否在集合中
const hasSync = (merged & SyncLane) !== 0; // true

// 從集合中移除某個 lane
const remaining = merged & ~SyncLane; // 只剩 DefaultLane

// 取得最高優先權的 lane（最低位的 bit）
function getHighestPriorityLane(lanes) {
  return lanes & -lanes;  // 位元運算的經典技巧
}
```

最後那個 `lanes & -lanes` 值得停下來看。在二補數中，`-lanes` 會把最低位的 1 以下的 bit 全部反轉。兩者做 AND 之後，只剩最低位的那個 1。因為 React 把越高優先權的 lane 放在越低的 bit，這個操作就能一步取出最高優先權。

### 三層優先權對映

React 的優先權體系其實有三層，這是很多人搞混的地方：

```
Event Priority     →    Lane Priority      →    Scheduler Priority
(事件來源)              (React 內部)              (排程器)
─────────────────────────────────────────────────────────────────
DiscreteEvent      →    SyncLane            →    ImmediatePriority
(click, keydown)
ContinuousEvent    →    InputContinuousLane →    UserBlockingPriority
(mousemove, scroll)
Default            →    DefaultLane         →    NormalPriority
(setState in useEffect)
Transition         →    TransitionLane1~16  →    NormalPriority
(startTransition)
Idle               →    IdleLane            →    IdlePriority
(offscreen, prerender)
```

當你在 `onClick` 裡呼叫 `setState`，React 透過 `requestUpdateLane()` 判斷當前的事件上下文，分配 `SyncLane`——這是最高優先權，會同步執行不中斷。但如果你包在 `startTransition` 裡，React 會分配一條 `TransitionLane`，這個 update 就變成可中斷的。

### Lane 的批次策略

回到 expirationTime 解決不了的問題：怎麼 batch 不連續的優先權？

```javascript
// getNextLanes() 決定這次 render 要處理哪些 lanes
function getNextLanes(root, wipLanes) {
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === 0) return NoLanes;

  // 先找非 idle 的 lanes
  const nonIdlePendingLanes = pendingLanes & ~IdleLanes;

  if (nonIdlePendingLanes !== 0) {
    // 從中挑出「沒被 suspend 擋住」的最高優先權群組
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;
    if (nonIdleUnblockedLanes !== 0) {
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      // 全被 suspend 了，找可以 ping 回來的
      const pingedLanes = nonIdlePendingLanes & pingedLanes;
      if (pingedLanes !== 0) {
        nextLanes = getHighestPriorityLanes(pingedLanes);
      }
    }
  }
  // ...
  return nextLanes;
}
```

關鍵在這裡：`suspendedLanes` 和 `pingedLanes` 也是 bitmask。當某個 Suspense 在等資料，React 只 suspend 那條 lane，其他 lane 的 CPU 工作照跑。expirationTime 做不到這件事——它會把同 expiration 的所有工作一起卡住。

每個 Fiber 節點上還存著 `lanes` 和 `childLanes`：

```javascript
fiber.lanes = SyncLane;       // 這個節點本身有待處理的 update
fiber.childLanes = DefaultLane; // 子樹中有待處理的 update
```

`childLanes` 是一個精妙的優化。當 React 在走訪 Fiber 樹時，如果發現一個節點的 `childLanes` 和當前正在處理的 `renderLanes` 沒有交集（`childLanes & renderLanes === 0`），就可以直接跳過整棵子樹。不用往下走了，因為子樹裡沒有這個優先權的工作。

## 時間切片：5 毫秒的承諾

Lane 模型解決了「什麼工作比較重要」的問題。接下來要解決「怎麼在不阻塞 UI 的前提下完成工作」。

答案是時間切片（Time Slicing）。原理很簡單：**React 每處理完一個 Fiber 節點，就檢查一次有沒有超時。超時了就把控制權還給瀏覽器。**

整個機制的核心就在 `workLoopConcurrent` 和 `workLoopSync` 的差異：

```javascript
// 同步模式：一口氣做完，不管死活
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// 併發模式：做一個就問一次「我該讓了嗎？」
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

差別就一個 `shouldYield()` 呼叫。就這樣。整個 Concurrent Mode 的核心差異就這一行。

### shouldYield 怎麼判斷

`shouldYield()` 來自 Scheduler 套件，它的邏輯是：

```javascript
const frameInterval = 5; // 毫秒

function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    return false; // 還沒到 5ms，繼續做
  }

  // 超過 5ms 了，但如果 browser 沒有等待的用戶輸入，可以再多做一下
  if (navigator.scheduling?.isInputPending?.()) {
    return true;  // 有用戶輸入在排隊，趕快讓
  }

  // 超過 300ms 了就一定要讓（避免餓死瀏覽器）
  if (timeElapsed > 300) {
    return true;
  }

  return false;
}
```

5 毫秒這個數字不是隨便選的。主流螢幕是 60fps，每幀約 16.6ms。React 佔 5ms 做 JS 工作，留下約 11ms 給瀏覽器做 layout、paint、composite。這在大部分裝置上足以維持流暢的畫面更新。

`isInputPending()` 是一個比較新的瀏覽器 API，讓 JS 可以問瀏覽器「現在有沒有使用者輸入事件在等待處理？」。如果有，React 會更積極地讓出控制權。如果沒有，即使超過 5ms，React 會試著多做一點。

### 為什麼選 MessageChannel 而不是 requestIdleCallback

這是面試常考題。React 一開始確實考慮過 `requestIdleCallback`（rIC），但最終選了 `MessageChannel`。原因：

1. **rIC 的執行時機太晚。** 它在瀏覽器「空閒時」才執行，但 React 需要的是「每幀結束後儘快執行」。rIC 可能在整幀結束後才跑，延遲太大。
2. **rIC 的 deadline 不穩定。** 瀏覽器給的空閒時間可能只有 1ms，也可能 50ms，React 需要更可預測的排程。
3. **Safari 長期不支援 rIC。** 這是一個現實考量（直到近期才部分支援）。

`setTimeout(fn, 0)` 也不行——瀏覽器對巢狀 setTimeout 有 4ms 的最低延遲限制（HTML spec 規定），這會嚴重影響排程精度。

`MessageChannel` 完美解決了問題：

```javascript
const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = performWorkUntilDeadline;

function schedulePerformWorkUntilDeadline() {
  port.postMessage(null);
}
```

`postMessage` 的 callback 會在**當前 macrotask 結束後、下一個 macrotask 開始時**立即執行，沒有人為延遲。這是最接近「把控制權還給瀏覽器一下下，然後馬上拿回來」的方式。

### 中斷後怎麼恢復

React 讓出控制權後，怎麼知道要「繼續做」？靠的是 continuation 模式：

```javascript
function performConcurrentWorkOnRoot(root, didTimeout) {
  // ... 做 render 工作 ...

  if (root.callbackNode === originalCallbackNode) {
    // render 還沒做完，回傳自己作為 continuation
    return performConcurrentWorkOnRoot.bind(null, root);
  }

  return null; // 做完了
}
```

當 `workLoopConcurrent` 因為 `shouldYield()` 中斷時，`performConcurrentWorkOnRoot` 會回傳一個 function。Scheduler 收到回傳的 function 後，不會建立新的 task，而是把同一個 task 的 callback 換成這個 continuation，然後在下一個時間片繼續執行。

這就是為什麼 React 能做到「做一點、讓一點、繼續做」的循環。

## Scheduler 的內部結構：兩個佇列和一個最小堆

React 的 Scheduler 是一個獨立套件 (`scheduler`)，它管理所有待執行的工作。核心結構只有兩個佇列：

```
taskQueue     → 已到期、可以立即執行的任務（sorted by expirationTime）
timerQueue    → 延遲任務，還沒到期（sorted by startTime）
```

兩個佇列都用**最小堆**（min-heap）實現，不是普通的 array sort。這保證插入和取出最高優先權任務都是 O(log n)：

```javascript
// SchedulerMinHeap.js — 這是完整的實現，就這麼短
function push(heap, node) {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);
}

function pop(heap) {
  if (heap.length === 0) return null;
  const first = heap[0];
  const last = heap.pop();
  if (last !== first) {
    heap[0] = last;
    siftDown(heap, last, 0);
  }
  return first;
}

function siftUp(heap, node, i) {
  let index = i;
  while (index > 0) {
    const parentIndex = (index - 1) >>> 1;
    const parent = heap[parentIndex];
    if (compare(parent, node) > 0) {
      heap[parentIndex] = node;
      heap[index] = parent;
      index = parentIndex;
    } else return;
  }
}
```

`scheduleCallback` 是排程的入口：

```javascript
function scheduleCallback(priorityLevel, callback, options) {
  const currentTime = getCurrentTime();
  const startTime = options?.delay ? currentTime + options.delay : currentTime;

  // 根據優先權決定 timeout（越高優先，timeout 越短）
  let timeout;
  switch (priorityLevel) {
    case ImmediatePriority:  timeout = -1;       break; // 已過期！
    case UserBlockingPriority: timeout = 250;     break;
    case NormalPriority:     timeout = 5000;      break;
    case LowPriority:       timeout = 10000;     break;
    case IdlePriority:      timeout = 1073741823; break; // maxSigned31BitInt
  }

  const expirationTime = startTime + timeout;
  const newTask = { callback, priorityLevel, startTime, expirationTime, sortIndex: -1 };

  if (startTime > currentTime) {
    // 延遲任務，放進 timerQueue
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
  } else {
    // 立即任務，放進 taskQueue
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    schedulePerformWorkUntilDeadline(); // 用 MessageChannel 觸發
  }

  return newTask;
}
```

注意 `ImmediatePriority` 的 timeout 是 `-1`，意味著它的 expirationTime 永遠在過去——已經過期了，必須馬上做。而 `IdlePriority` 的 timeout 接近 12.4 天（`maxSigned31BitInt` 毫秒），基本上可以無限延後。

### 飢餓問題

一個高優先權的 update 不斷插隊，低優先權的工作永遠做不完——這就是飢餓（starvation）。React 怎麼處理？

每個 task 都有 `expirationTime`。當 Scheduler 從 taskQueue 取出任務時，如果任務的 expirationTime 已經過了（`currentTime >= task.expirationTime`），這個任務就算「餓了」，會被提升到同步執行，不再可中斷。

在 Lane 模型中也有對應的機制。`markStarvedLanesAsExpired()` 會檢查每條 lane 等了多久：

```javascript
function markStarvedLanesAsExpired(root, currentTime) {
  const pendingLanes = root.pendingLanes;
  const expirationTimes = root.expirationTimes;

  let lanes = pendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;
    const expirationTime = expirationTimes[index];

    if (expirationTime === NoTimestamp) {
      // 還沒設到期時間，設一個
      expirationTimes[index] = computeExpirationTime(lane, currentTime);
    } else if (expirationTime <= currentTime) {
      // 過期了！標記為已到期
      root.expiredLanes |= lane;
    }

    lanes &= ~lane;
  }
}
```

一旦一條 lane 被標記為 `expiredLanes`，下次 `getNextLanes()` 就會優先處理它，而且會用同步模式執行。飢餓保護確保了低優先權的工作最終一定會被執行。

## startTransition：開發者的排程介面

以上這些機制對開發者來說都是透明的——直到 `startTransition` 出現。這是 React 第一次讓開發者**直接參與排程決策**。

```javascript
function SearchResults({ query }) {
  const [isPending, startTransition] = useTransition();
  const [filterText, setFilterText] = useState('');

  function handleChange(e) {
    // 立即更新輸入框（高優先權）
    setFilterText(e.target.value);

    // 延遲更新搜尋結果（低優先權）
    startTransition(() => {
      setSearchQuery(e.target.value);
    });
  }
  // ...
}
```

底層發生了什麼？

1. `startTransition` 呼叫時，React 把全域的 `ReactCurrentBatchConfig.transition` 設為一個非 null 物件。
2. 在 transition callback 裡的 `setState` 呼叫 `requestUpdateLane()` 時，檢測到 transition 上下文，分配一條 `TransitionLane`。
3. React 有 16 條 TransitionLane（bit 7 到 bit 22），`requestTransitionLane()` 會循環分配：

```javascript
function requestTransitionLane() {
  // 循環使用 16 條 TransitionLane
  const lane = nextTransitionLane; // 從 TransitionLane1 開始
  nextTransitionLane <<= 1;
  if ((nextTransitionLane & TransitionLanes) === 0) {
    nextTransitionLane = TransitionLane1; // 回到第一條
  }
  return lane;
}
```

為什麼要 16 條？因為多個 `startTransition` 可能同時進行。不同 transition 分配到不同的 lane，React 就能獨立追蹤、獨立取消它們。用完 16 條就循環回來——實務上 16 個同時進行的 transition 已經夠用了。

### 中斷的真正機制

當使用者快速打字，每次按鍵都觸發一個新的 transition。React 怎麼「中斷」前一個正在做的 transition render？

```
按鍵 'a': 分配 TransitionLane1 → 開始 render → workLoopConcurrent 開始處理
  ↓
按鍵 'b': SyncLane update（輸入框）+ 新的 TransitionLane2 update
  ↓
shouldYield() → true（因為有高優先權的 SyncLane）
  ↓
React 放棄 TransitionLane1 的半成品 render
  ↓
先執行 SyncLane update（更新輸入框）
  ↓
再開始 TransitionLane2 的 render（搜 'ab' 的結果）
```

關鍵程式碼在 `ensureRootIsScheduled()`：

```javascript
function ensureRootIsScheduled(root) {
  const nextLanes = getNextLanes(root, workInProgressRootRenderLanes);

  // 如果新的最高優先權和正在做的不同
  if (existingCallbackPriority !== newCallbackPriority) {
    // 取消正在進行的排程任務
    cancelCallback(existingCallbackNode);
  }

  // 根據新的優先權重新排程
  const newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root)
  );

  root.callbackNode = newCallbackNode;
  root.callbackPriority = newCallbackPriority;
}
```

當高優先權的 update 進來，`getNextLanes()` 會回傳新的優先 lane，如果和正在處理的不同，React 就取消舊的 scheduler task，建立新的。正在進行的 concurrent render 自然就被丟棄了——`workInProgress` 樹只做了一半，不會被 commit。

`isPending` 的實現也很巧妙。當你呼叫 `startTransition` 時，React 其實執行了**兩個** update：

```javascript
function startTransition(setPending, callback) {
  // 第一個 update：高優先權（SyncLane），把 isPending 設為 true
  setPending(true);

  // 切換到 transition 上下文
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = {};

  // 第二個 update：低優先權（TransitionLane），執行 callback + setPending(false)
  setPending(false);
  callback();

  ReactCurrentBatchConfig.transition = prevTransition;
}
```

`setPending(true)` 在高優先權 lane，會馬上 render 出來。`setPending(false)` 在 transition lane，要等 transition 完成才會 render。所以 `isPending` 在 transition 期間會是 `true`——這就是你拿來顯示 loading spinner 的狀態。

## 這些對你日常開發到底有什麼影響

理論很美好，但什麼時候 Concurrent Mode 真的有差？

**真正受益的場景：**

- **大量 re-render 的清單或表格**：搜尋 filter、排序、展開收合樹狀結構。用 `startTransition` 把 filter 邏輯包起來，輸入框永遠不會卡。
- **複雜的圖表或視覺化**：D3 或 canvas 重繪搭配 `useDeferredValue`，讓互動保持流暢。
- **頁面切換**：SPA 的路由切換時，如果新頁面很重（大量資料 fetch + render），transition 讓舊頁面在切換期間保持可互動。

**不會有感的場景：**

- 元件本身就很輕量，render 在 1ms 內完成。時間切片的 overhead 反而是浪費。
- 已經在用 `React.memo`、`useMemo` 做好精準 memoization 的 app。
- 伺服器端渲染——SSR 是同步的，Concurrent 特性只在 client 有用。

**常見誤解澄清：**

1. **「Concurrent Mode 讓 React 多線程了」**——沒有。JavaScript 還是單線程。React 只是在一個線程裡做到「做一點、讓一點」的協作排程。
2. **「所有 state update 都是併發的」**——錯。只有用 `startTransition` 或 `useDeferredValue` 標記的 update 才走併發路徑。`onClick` 裡的 `setState` 預設是 `SyncLane`，同步執行。
3. **「時間切片有很大的效能開銷」**——其實很小。`shouldYield()` 只是讀一個 timestamp 做比較，nanosecond 級別。真正的開銷在中斷後丟棄半成品 render 重來——但這只發生在有高優先權 update 插隊時。

---

## 結論：排程是一場取捨

React 的 Concurrent 排程器是一個精心設計的系統：Lane 用 bitmask 解決了優先權的表達問題，Scheduler 用 5ms 時間切片和 MessageChannel 實現了不阻塞 UI 的協作排程，`startTransition` 把排程決策權交給了開發者。

但它不是免費的午餐。中斷和恢復有成本，多條 lane 的排程邏輯增加了 React 內部的複雜度，而且它只解決 CPU-bound 的渲染問題——如果你的效能瓶頸在網路請求或 DOM layout，Concurrent Mode 幫不了你。

我的建議很具體：

1. **預設不用刻意啟用 concurrent 特性。** React 18+ 的 `createRoot` 已經開啟 concurrent 能力，但 `setState` 預設還是同步的。
2. **當你遇到明顯的 UI 卡頓，才用 `startTransition`。** 先 profile，確認瓶頸是 render 太重，然後用 transition 包起來。
3. **不要把所有 update 都包在 `startTransition` 裡。** 使用者直接互動（打字、點擊）的反饋應該是即時的，只有「結果」可以延遲。
4. **理解 Lane 模型有助於 debug。** React DevTools 的 Profiler 會顯示 lane 資訊，知道你的 update 在哪條 lane，就能理解為什麼某些 render 被延遲了。

React 的排程器不會讓你的程式碼變快——它讓你的程式碼**看起來**更快。這不是自欺欺人，這是 UX 工程的核心：感知效能比實際效能更重要。排程器做的事，就是確保使用者「感覺」這個 app 很流暢——即使你的 render 其實很重。

---

## 延伸閱讀

- [React Source: ReactFiberLane.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberLane.js) — Lane 模型的完整定義，所有 bitmask 常數和操作函式
- [Andrew Clark's Lane PR #18796](https://github.com/facebook/react/pull/18796) — 從 expirationTime 遷移到 Lane 的原始 PR，提交訊息解釋了動機
- [React Source: Scheduler.js](https://github.com/facebook/react/blob/main/packages/scheduler/src/forks/Scheduler.js) — Scheduler 的完整實現，包含 min-heap 和 MessageChannel 排程
- [Building a Custom React Renderer (Sophie Alpert)](https://www.youtube.com/watch?v=CGpMlWVcHok) — 理解 React reconciler 的內部運作
- [React 18 Working Group: Discussion on Concurrent Features](https://github.com/reactwg/react-18/discussions) — React 團隊對 concurrent 特性的設計討論
- [Inside React's Scheduler (Maxim Koretskyi)](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react) — 深入分析 React Fiber 和排程器的經典文章
