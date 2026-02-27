---
title: "React Virtual DOM Diff 演算法演進：從 O(n³) 到 O(n) 的工程取捨，以及 No-VDOM 陣營的反擊"
date: 2026-02-27
description: "深入解析 React 的 Virtual DOM diff 演算法如何用三個啟發式假設將樹比對從 O(n³) 降到 O(n)，剖析 reconcileChildFibers 的原始碼實現、key 機制的內部運作，並比較 Svelte、Solid.js 等 no-VDOM 框架的替代方案。"
tags: [deep-dive, react, frontend, performance]
---

「React 用 Virtual DOM 所以很快。」

如果你在面試中聽到這句話，恭喜，你遇到了前端圈最持久的都市傳說之一。連 React 團隊的 Dan Abramov 自己都說過：他希望 React 社群能淘汰掉「Virtual DOM」這個詞。因為 VDOM 從來就不是為了「快」而存在的——它是為了讓你能用宣告式的方式寫 UI，而不用手動操作 DOM。

但這裡有個有趣的工程問題：既然每次 state 更新都要產生一棵新的虛擬樹，React 怎麼知道哪些 DOM 節點需要改？樹和樹之間的比對（diff），在電腦科學裡是一個 O(n³) 的問題。一棵 1000 個節點的樹，理論上要做十億次比較。React 卻把它壓到了 O(n)。

這不是魔法。這是一連串精心設計的「放棄」。

今天我們來拆解這些取捨——從學術上的最優解到 React 的工程妥協，再到 Svelte、Solid.js 這些乾脆說「我不要 VDOM」的框架是怎麼繞過這個問題的。

---

## 樹編輯距離：為什麼是 O(n³)

在正式看 React 之前，先理解它在躲避什麼。

電腦科學中有個經典問題叫「樹編輯距離」（Tree Edit Distance）：給你兩棵樹，算出從 A 變成 B 最少需要多少次操作（插入、刪除、替換節點）。1989 年，Zhang 和 Shasha 發表了經典演算法，時間複雜度是 O(n²m²)，其中 n 和 m 是兩棵樹的大小。對於大小相近的樹，這基本上就是 O(n⁴)。後來的優化版本把它降到了 O(n³)，但這仍然是災難級的——一個中等複雜的 React app 可能有幾千個 DOM 節點，O(n³) 意味著每次 `setState` 都要算幾十億次。

更慘的是，2018 年的一篇 ACM 論文證明了：在一般情況下，樹編輯距離不太可能突破次立方（subcubic）的時間複雜度。換句話說，學術界也沒有銀彈。

React 的解法？不追求最優解。

---

## React 的三個啟發式假設

2013 年，React 團隊的 Christopher Chedeau（vjeux）在一篇經典文章中，把 React 的 diff 策略歸結為三個啟發式假設（heuristics）：

**假設一：跨層級的節點移動極少發生。**

傳統的樹 diff 會嘗試找出「節點 A 從第二層移動到了第五層」這種操作。React 直接放棄了——它只做同層比較。如果一個 `<div>` 在舊樹的第二層，新樹裡沒有了，React 不會去其他層找它，直接銷毀整棵子樹，然後在新位置重建。

這聽起來很浪費？實際上，在真實的 UI 開發中，跨層級移動確實極少發生。你不太會把一個 sidebar 裡的組件搬到 header 裡去。React 用「放棄罕見情況的最優解」換來了「常見情況的極速處理」。

**假設二：不同類型的元素會產生不同的樹。**

如果舊節點是 `<div>`，新節點是 `<span>`——不比了，直接拆掉舊的整棵子樹，建新的。如果舊的是 `<ComponentA>`，新的是 `<ComponentB>`——同樣，直接重建。

這個假設激進嗎？有點。理論上 `<div>` 換成 `<section>` 可能子節點完全一樣。但 React 認為，與其花時間去驗證這個可能性，不如直接重建——因為不同類型的組件大概率有不同的 DOM 結構。

**假設三：開發者可以用 key 提示哪些子元素是穩定的。**

這是唯一需要開發者配合的部分，也是最常被搞砸的部分（稍後細說）。

有了這三個假設，diff 就從「比較兩棵任意樹」簡化成「逐層、逐節點、線性掃描」——O(n)。

---

## reconcileChildFibers：原始碼裡的真相

說了半天理論，來看 React 到底怎麼寫的。核心邏輯在 `ReactChildFiber.js` 的 `reconcileChildFibers` 函式裡。這個函式處理一個 Fiber 節點的所有子節點的 diff。

React 根據新的 children 的類型分發到不同的處理路徑：

```javascript
function reconcileChildFibers(returnFiber, currentFirstChild, newChild) {
  // 單一元素 — 最常見的路徑
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return reconcileSingleElement(returnFiber, currentFirstChild, newChild);
      case REACT_PORTAL_TYPE:
        return reconcileSinglePortal(returnFiber, currentFirstChild, newChild);
    }
    // 陣列 — 列表渲染
    if (isArray(newChild)) {
      return reconcileChildrenArray(returnFiber, currentFirstChild, newChild);
    }
  }
  // 文字節點
  if (typeof newChild === 'string' || typeof newChild === 'number') {
    return reconcileSingleTextNode(returnFiber, currentFirstChild, newChild);
  }
  // 什麼都沒有 — 刪除所有舊子節點
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

最有趣的是 `reconcileSingleElement`——處理單一子元素的 diff。看它怎麼做：

```javascript
function reconcileSingleElement(returnFiber, currentFirstChild, element) {
  const key = element.key;
  let child = currentFirstChild;

  while (child !== null) {
    if (child.key === key) {
      // key 相同，看型別
      if (child.elementType === element.type) {
        // 型別也相同 → 複用這個 Fiber，刪除其餘兄弟
        deleteRemainingChildren(returnFiber, child.sibling);
        const existing = useFiber(child, element.props);
        existing.return = returnFiber;
        return existing;
      }
      // key 相同但型別不同 → 刪除所有舊的（因為 key 應該是唯一的）
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      // key 不同 → 標記這個舊節點為刪除
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }
  // 沒找到可複用的 → 建立新的 Fiber
  const created = createFiberFromElement(element);
  created.return = returnFiber;
  return created;
}
```

注意這裡的 `child.sibling`——Fiber 架構把子節點組織成鏈表，不是陣列。所以遍歷兄弟節點就是一路 `.sibling` 走下去。這是 Fiber 架構的特徵，但 diff 的核心邏輯（先比 key，再比 type）從 Stack 時代就沒變過。

---

## 陣列 Diff：兩輪掃描的精妙設計

最複雜的 diff 場景是列表——也就是 `reconcileChildrenArray`。這裡的演算法分成兩輪：

### 第一輪：線性比對

從頭開始，逐一比較新舊列表的元素：

```
舊: [A, B, C, D, E]
新: [A, B, F, D, E]

第一輪：
  A === A ✓ 複用
  B === B ✓ 複用
  C !== F ✗ 停止
```

只要碰到 key 或 type 不同的，第一輪就停止。這個設計的直覺是：大部分列表更新都發生在尾部（新增、刪除最後幾個元素），頭部通常是穩定的。

### 第二輪：Map 查找

如果第一輪沒處理完，React 把剩餘的舊節點塞進一個 Map（key → Fiber）：

```javascript
// 簡化的虛擬碼
const existingChildren = new Map();
// 把 C, D, E 放進 Map
existingChildren.set('C', fiberC);
existingChildren.set('D', fiberD);
existingChildren.set('E', fiberE);

// 然後遍歷新列表剩餘的元素
for (const newChild of remainingNewChildren) {
  const existing = existingChildren.get(newChild.key);
  if (existing && existing.type === newChild.type) {
    // 找到了 → 複用，標記位置變化
    reuse(existing);
    existingChildren.delete(newChild.key);
  } else {
    // 找不到 → 建立新的
    createNew(newChild);
  }
}

// Map 裡剩下的都是要刪除的
existingChildren.forEach(child => deleteChild(child));
```

第一輪是 O(min(n,m))，第二輪建 Map 是 O(m)，查找是 O(n)——整體是 **O(n+m)**，也就是線性。

這裡有個重要的細節：React 不會去計算「最小移動次數」。它只判斷節點是否可以複用，以及大致的位置變化方向。它用一個 `lastPlacedIndex` 變數來判斷節點是否需要移動：

```javascript
let lastPlacedIndex = 0;

for (const [index, newChild] of newChildren.entries()) {
  const oldIndex = existingChildren.get(newChild.key)?.index;
  if (oldIndex !== undefined) {
    if (oldIndex >= lastPlacedIndex) {
      // 這個節點在舊列表中的位置 >= 上一個複用節點的位置
      // 不用移動，更新 lastPlacedIndex
      lastPlacedIndex = oldIndex;
    } else {
      // 需要移動（標記為 Placement）
      markForPlacement(newChild);
    }
  }
}
```

這個策略有一個已知的弱點：如果你把列表的最後一個元素移到最前面，React 不會只移動那一個元素——它會認為其他所有元素都需要移動。比如：

```
舊: [A, B, C, D]
新: [D, A, B, C]

React 的判斷：D 不動（lastPlacedIndex = 3），
  A(oldIndex=0) < 3 → 移動
  B(oldIndex=1) < 3 → 移動
  C(oldIndex=2) < 3 → 移動

結果：移動了 3 個節點，而最優解是只移動 D 一個。
```

Vue 3 用了更聰明的演算法（最長遞增子序列，Longest Increasing Subsequence）來最小化 DOM 移動次數。但 React 團隊認為這個 trade-off 可以接受——因為算 LIS 本身也有開銷，而上面這種「把最後一個移到最前面」的操作在真實 UI 中並不常見。

---

## Key 的內部運作：為什麼 index 當 key 是反模式

既然看了原始碼，我們現在可以精確理解 key 為什麼重要。

React 的 diff 對子節點列表的判斷完全依賴 key。沒有 key 的元素，React 會退化到按位置（index）比對——第一個和第一個比，第二個和第二個比。

這在列表沒有重排的時候沒問題。但如果你在列表前面插入一個元素：

```
舊: [B, C, D]        （沒有 key，隱含 index 0, 1, 2）
新: [A, B, C, D]      （隱含 index 0, 1, 2, 3）

React 的比對：
  index 0: B → A  型別相同 → 複用 B 的 Fiber，更新 props 為 A
  index 1: C → B  型別相同 → 複用 C 的 Fiber，更新 props 為 B
  index 2: D → C  型別相同 → 複用 D 的 Fiber，更新 props 為 C
  index 3: 無  → D  新建

結果：三次不必要的 props 更新 + 一次新建
正確做法（有 key）：一次插入就夠了
```

更糟糕的是，如果這些組件有內部 state（比如 `<input>` 的輸入值），按 index 比對會導致 state 錯亂——B 的 state 會跑到 A 身上。這不是效能問題，這是 bug。

所以，用 `index` 當 key 跟沒給 key 一樣（React 預設就是按 index 比對）。你需要的是穩定的、唯一的 key——通常是資料的 ID。

---

## 從 Stack 到 Fiber：Diff 邏輯變了嗎？

在我們的 [Fiber 架構深度解析](/posts/react-fiber-reconciliation-deep-dive/) 中，我們詳細拆解了 React 從 Stack Reconciler 到 Fiber Reconciler 的轉變。但這裡要強調一個常被忽略的事實：**diff 的核心邏輯幾乎沒變**。

Stack 時代的 diff 規則：
1. 不同類型 → 重建子樹
2. 同類型 DOM 元素 → 比對屬性
3. 同類型組件 → 遞迴更新
4. 列表用 key 比對

Fiber 時代？完全一樣。

真正變的是 diff 的**執行方式**：

- **Stack**：遞迴遍歷，一旦開始就不能中斷。一棵大樹的 diff 可能阻塞主線程好幾百毫秒。
- **Fiber**：把樹攤平成鏈表，diff 變成一個可以暫停、恢復的迴圈。每處理一個節點就可以檢查一次「有沒有更高優先級的工作」。

```
Stack 時代（遞迴，不可中斷）：
reconcile(root)
  → reconcile(child1)
    → reconcile(grandchild1)
    → reconcile(grandchild2)
  → reconcile(child2)    ← 到這裡才處理 child2，中間不能停

Fiber 時代（迴圈，可中斷）：
while (nextUnitOfWork !== null) {
  nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  if (shouldYield()) break; // 可以在任何節點之間暫停
}
```

所以 Fiber 不是改了 diff 演算法，而是改了 diff 的排程方式。演算法是一樣的 O(n)，但執行體驗完全不同。

---

## No-VDOM 陣營：他們怎麼繞過這一切

到這裡你可能會想：既然 VDOM diff 有這麼多 trade-off，有沒有辦法完全跳過 diff？

有。而且不只一種。

### Svelte：編譯時期解決一切

Svelte 的創造者 Rich Harris 在 2018 年寫了一篇著名的文章〈Virtual DOM is pure overhead〉（虛擬 DOM 是純粹的開銷），直接向 VDOM 開戰。他的核心論點是：

> VDOM diff 不是免費的。你建立了一整棵虛擬樹，然後花 O(n) 遍歷它來找出哪些東西改了——但**你的編譯器其實在編譯時就知道哪些東西可能會改**。

Svelte 的做法是在編譯時分析你的組件，生成精準的 DOM 更新指令：

```javascript
// 你寫的 Svelte
<script>
  let count = 0;
</script>
<p>{count}</p>
<button on:click={() => count++}>+1</button>

// Svelte 編譯後（簡化）
function update(changed) {
  if (changed.count) {
    set_data(t0, count);  // 直接更新那個 text node，不 diff
  }
}
```

沒有虛擬樹。沒有 diff。編譯器在建置時期就知道 `count` 變了只會影響那個 `<p>` 裡的文字節點，所以直接生成 `set_data` 呼叫。O(1) 的更新。

### Solid.js：執行時的細粒度響應式

Solid.js 走了另一條路——它不靠編譯器做靜態分析，而是用細粒度的響應式系統（Fine-Grained Reactivity）。每個 signal（類似 state）都知道自己被哪些 DOM 節點訂閱了：

```javascript
// Solid.js
const [count, setCount] = createSignal(0);

// 這個 effect 在建立時就綁定了 count → textNode 的關係
createEffect(() => {
  textNode.data = count();
});
```

當 `count` 變了，signal 系統直接通知綁定的 DOM 節點更新。不需要 diff 整棵樹，甚至不需要 re-render 整個組件。Ryan Carniato（Solid.js 創造者）稱之為「組件只執行一次」——組件函式跑一次建立訂閱關係，之後所有更新都是 signal 直接驅動。

### Vue 3：VDOM + 編譯時優化的混合路線

Vue 3 沒有放棄 VDOM，但用編譯器做了大量優化來減少 diff 的工作量：

- **靜態提升（Static Hoisting）**：純靜態的節點在編譯時被提取出來，diff 時直接跳過。
- **PatchFlags**：編譯器給每個動態節點標記「哪些東西是動態的」——只有 class？只有 text？還是 props？diff 時只檢查有標記的部分。
- **Block Tree**：把組件切成「動態區塊」，diff 時只遍歷動態節點，不碰靜態的。

```html
<!-- Vue template -->
<div>
  <p>這段文字永遠不變</p>
  <p>{{ dynamicText }}</p>
</div>

<!-- 編譯後，React 風格的 diff 會比較兩個 <p>，
     但 Vue 的 Block Tree 只追蹤第二個 <p> -->
```

這讓 Vue 3 在保留 VDOM 的程式設計模型的同時，把 diff 的實際工作量壓到接近最小。

---

## 效能真相：Benchmark 告訴我們什麼

說了這麼多理論，實際跑起來呢？js-framework-benchmark 是前端框架效能的標準測試場。2025 年的結果大致是這樣的排名趨勢：

1. **Solid.js / Svelte** — 接近原生 DOM 操作的速度
2. **Vue 3 (Vapor mode)** — 幾乎追上 Solid
3. **Inferno** — VDOM 陣營的效能王者（用了極度優化的 keyed diff）
4. **Vue 3** — 穩定的好成績
5. **React** — 中等偏上，在大量列表操作時略顯吃力

但——**benchmark 不是全部**。

React 在 benchmark 中不是最快的，但它處理複雜 UI 狀態的能力（Concurrent Features、Suspense、Transitions）是 benchmark 測不出來的。當你的 app 有幾十個組件同時更新、使用者在打字的同時背景在載入資料、而你需要確保輸入不卡頓——這時候 Fiber 的排程能力比 diff 快個幾毫秒重要得多。

Svelte 和 Solid 的更新確實更快，但它們的程式設計模型也有各自的限制。Svelte 的編譯魔法讓 debug 更難（你看到的不是你寫的）。Solid 的「組件只執行一次」讓習慣了 React 心智模型的開發者容易踩坑。

---

## 結論：VDOM 不是為了快，是為了「不用想」

回到文章開頭那個迷思。Virtual DOM 從來不是一個效能優化技術——它是一個**抽象層**。它讓你能寫 `return <div>{items.map(...)}</div>` 而不用管「上一次 render 有幾個 item、這次新增了哪些、哪些要刪除」。Diff 是這個抽象層的實現代價，React 用三個啟發式假設把它壓到了可接受的範圍。

如果你用 React，你該知道的是：
- **給列表加 key，而且不要用 index**。這是你能給 diff 演算法最大的幫助。
- **避免不必要的組件類型變化**。條件渲染時，盡量保持同一個位置的組件類型穩定。
- **不要過度擔心 diff 效能**。在 99% 的場景中，你的效能瓶頸是不必要的 re-render，不是 diff 本身。

如果你在評估新專案的技術選型，VDOM vs no-VDOM 不應該是決定因素。選 React 是因為生態系和 Concurrent Features。選 Svelte 是因為簡潔和 bundle size。選 Solid 是因為你想要最極致的更新效能。diff 演算法？那是框架作者該操心的事，不是你的。

---

## 延伸閱讀

- [React's diff algorithm — vjeux (2013)](https://calendar.perfplanet.com/2013/diff/) — React diff 的原始設計文件，直接從作者口中了解三大假設
- [Virtual DOM is pure overhead — Rich Harris](https://svelte.dev/blog/virtual-dom-is-pure-overhead) — Svelte 創造者對 VDOM 的正面挑戰，論點清晰值得一讀
- [React Reconciliation 官方文件](https://react.dev/learn/preserving-and-resetting-state) — React 官方對 diff 行為的說明，特別是 key 和組件位置的影響
- [ReactChildFiber.js 原始碼](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactChildFiber.js) — 本文分析的核心原始碼，想看完整實現的人直接啃這個
- [Thinking Granular: How is SolidJS so Performant? — Ryan Carniato](https://dev.to/ryansolid/thinking-granular-how-is-solidjs-so-performant-4g37) — Solid.js 作者親自解釋細粒度響應式為什麼能跳過 diff
- [Vue 3 Rendering Mechanism](https://vuejs.org/guide/extras/rendering-mechanism) — Vue 的編譯時優化策略，看 VDOM 陣營怎麼用 PatchFlags 反擊
