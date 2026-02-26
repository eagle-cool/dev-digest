---
title: "React Compiler 深度解析：65 道編譯 Pass 如何讓你不再寫 useMemo"
date: 2026-02-26
description: "React Compiler v1.0 終於穩定發布。這篇從原始碼級別拆解它的 65 道編譯 Pass、HIR 中間表示、SSA 轉換、Reactive Scope 推斷，以及它在 Meta 內部實測的真實效能數據。看完你就知道為什麼可以丟掉 useMemo 了——以及什麼時候還不能丟。"
tags: [deep-dive, react, compiler, performance]
---

你有沒有算過，你的 React 專案裡有多少個 `useMemo`、`useCallback`、`React.memo`？

如果答案是「多到數不清」，恭喜你，你跟 Meta 的工程師一樣痛苦。差別在於——他們花了將近十年，寫了一個編譯器來解決這件事。

2025 年 10 月 7 日，React Compiler v1.0 正式發布。這不是一個概念驗證，不是一個 RFC，是一個跑在 Meta 生產環境上、讓 Quest Store 頁面載入快 12%、特定互動快 2.5 倍的真傢伙。

但它到底怎麼做到的？一個 Babel plugin 怎麼可能比你手動優化更聰明？今天我們扒開原始碼，從 65 道編譯 Pass 開始，一路看到底。

---

## 從手動 Memo 地獄說起

React 的 re-render 機制是出了名的「誠實」——state 一變，整棵子樹重新渲染。這在小應用裡不是問題，但當你的元件樹長到幾百個節點時，效能就開始崩了。

於是我們有了三件套：

```jsx
const processedData = useMemo(() => expensiveCalc(data), [data]);
const handleClick = useCallback((id) => onClick(id), [onClick]);
export default React.memo(MyComponent);
```

問題來了：**你怎麼知道該 memo 哪裡？** 答案是——你不知道。你只能用 Profiler 量、用經驗猜、或者乾脆到處都加（然後被 code review 打回來說「premature optimization」）。

更糟的是，`useMemo` 和 `useCallback` 受 Hooks 規則限制，不能放在條件式裡。你想在 early return 後面 memo？做夢。

React Compiler 的設計目標很明確：**讓效能預設就是好的，讓開發者不用再想 memo 的事。** 它不是引入新概念，而是移除概念。

---

## 編譯器的骨架：不是語法糖，是正經的編譯器

很多人以為 React Compiler 就是個「自動幫你加 useMemo 的 codemod」。大錯特錯。

它是一個真正的編譯器，有自己的中間表示（IR）、靜態單賦值形式（SSA）、資料流分析、副作用推斷、和程式碼生成。整個管線跑下來，你的元件會經過 **65 道編譯 Pass**。

原始碼在 `compiler/packages/babel-plugin-react-compiler/src/`，結構長這樣：

```
Entrypoint/     ← Pipeline.ts，65 道 Pass 的指揮中心
HIR/            ← 高階中間表示（控制流圖）
SSA/            ← 靜態單賦值轉換
Inference/      ← 型別與效果推斷
ReactiveScopes/ ← 反應式作用域分析（核心中的核心）
Optimization/   ← 各種優化 Pass
Validation/     ← Rules of React 驗證
Transform/      ← 程式碼轉換
```

接下來我們按編譯管線的順序，一步步看它怎麼把你的 JSX 變成自動 memo 的程式碼。

---

## Phase 1：Lowering — 從 AST 到控制流圖

第一步是 `lower()`，把 Babel 的 AST 轉成 HIR（High-level Intermediate Representation）。

HIR 是一個**控制流圖（CFG）**，沒有巢狀結構。每個基本區塊（basic block）包含一系列指令和一個終端（分支、回傳、跳轉）：

```
bb0:
  $1 = LoadLocal props
  $2 = Destructure { data, onClick } = $1
  $3 = LoadLocal data
  If ($3 !== null) then:bb1 else:bb2

bb1:
  $4 = Call expensiveCalc($3)
  $5 = StoreLocal processedData = $4
  Jump bb3

bb2:
  $5 = StoreLocal processedData = null
  Jump bb3

bb3:
  $6 = LoadLocal processedData
  $7 = JSX <div>{$6}</div>
  Return $7
```

為什麼要攤平成 CFG？因為**巢狀的 AST 沒辦法做精確的資料流分析**。當你把 `if-else` 攤平成 bb1/bb2 兩個分支再合流到 bb3，每個值的定義點和使用點就一目了然。

HIR 保留了高階資訊——它知道 `if` 跟三元運算子是不同的東西、`for` 跟 `while` 是不同的迴圈。這是刻意的設計：React Compiler 需要理解 JavaScript 的執行語義，而不是把所有控制流都抹平成 goto。

入口在 `HIR/BuildHIR.ts`，`lowerStatement()` 用一個巨大的 switch 處理所有 JavaScript 語句類型：`IfStatement`、`ForStatement`、`TryStatement`...每一種都有對應的 CFG 建構邏輯。

---

## Phase 2：SSA 轉換 — 讓每個值只被賦值一次

CFG 建好之後，下一步是轉成 **SSA（Static Single Assignment）形式**。SSA 的規則很簡單：**每個變數只能被賦值一次**。

```javascript
// 原始碼
let x = 5;
if (condition) { x = 10; }
console.log(x);

// SSA 形式
x₁ = 5;
if (condition) { x₂ = 10; }
x₃ = φ(x₁, x₂);  // phi 函式：在合流點合併兩個定義
console.log(x₃);
```

那個 `φ`（phi）函式是 SSA 的精髓——在控制流合併的地方，它告訴你「這個值可能來自哪些分支」。

React Compiler 用的是 **Braun 演算法（2013）**，出自論文 "Simple and Efficient Construction of Static Single Assignment Form"。相比經典的 Cytron 演算法，Braun 的好處是不需要先算支配樹（dominator tree），可以直接從 AST 建構 SSA。

SSA 對編譯器來說是超級武器。因為每個值只有一個定義點，你可以輕鬆追蹤：這個值從哪來？它依賴哪些輸入？它會不會在渲染過程中被修改？

這正是 React Compiler 需要回答的核心問題。

---

## Phase 3：反應式分析 — 找出哪些值會變

SSA 轉換完之後，編譯器要回答一個關鍵問題：**在兩次渲染之間，哪些值可能改變？**

這就是 `inferReactivePlaces()` 的工作。它標記所有**反應式值（reactive values）**：

- **Props** — 父元件可以傳不同的值
- **State** — `useState` 的值會變
- **Context** — `useContext` 的值會變
- 從以上值衍生的任何計算結果

非反應式的值（常數、模組層級變數、React 內建函式）不會改變，不需要 memo。

接下來是 `inferReactiveScopeVariables()`，它把相關的指令分組成**反應式作用域（Reactive Scope）**：

```
ReactiveScope #1 [deps: data]
  processedData = expensiveCalc(data)

ReactiveScope #2 [deps: onClick]
  handleClick = (item) => onClick(item.id)

ReactiveScope #3 [deps: processedData, handleClick]
  jsx = <div>{processedData.map(...)}</div>
```

每個 Reactive Scope 有自己的依賴清單。只有當依賴改變時，這個作用域裡的程式碼才需要重新執行。

這比你手動寫 `useMemo` 更細緻——編譯器可以把一個元件裡的 JSX 拆成多個獨立的 scope，每個 scope 各自追蹤依賴。你手動做？光想 dependency array 就要想破頭。

---

## Phase 4：副作用追蹤 — 誰動了我的資料

React Compiler 不只追蹤值的流動，還追蹤**副作用（effects）**。它定義了五種效果類型：

| 效果 | 意義 |
|------|------|
| **Read** | 讀取一個值 |
| **Store** | 儲存到本地變數 |
| **Capture** | 在閉包中捕獲引用 |
| **Mutate** | 修改一個值 |
| **Freeze** | 凍結一個值（標記為不可變） |

`inferMutationAliasingEffects()` 和 `inferMutationAliasingRanges()` 這兩個 Pass 會追蹤每個值的**突變範圍（mutation range）**——從它被建立到最後一次被修改之間的區間。

為什麼這很重要？因為 memo 的前提是**值不會在快取之後被偷偷修改**。如果編譯器發現一個物件在被傳給子元件之後還被 mutate，它就不能安全地 memo 這個值。

```javascript
// 編譯器會偵測到這裡的問題
const items = [];
items.push(computedValue);  // mutation
return <List data={items} />;
// items 在同一個 render 中被 mutate 然後 freeze（傳給 JSX）
// 編譯器會把 mutation 和 freeze 放在同一個 scope
```

這也是為什麼 Rules of React 要求你不能在 render 中做副作用——編譯器需要假設你的元件是**冪等的（idempotent）**：相同輸入，相同輸出，沒有副作用。

---

## Phase 5：驗證 — Rules of React 不只是建議

編譯管線裡有一整票驗證 Pass，對應 React 的規則：

```
validateNoSetStateInRender()        ← render 裡不能 setState
validateNoRefAccessInRender()       ← render 裡不能讀 ref.current
validateNoDerivedComputationsInEffects()  ← effect 裡不該做衍生計算
validateNoJSXInTryStatement()       ← try 裡不能有 JSX
validateLocalsNotReassignedAfterRender() ← render 後不能重新賦值
```

如果你的程式碼違反了這些規則，編譯器不會報錯——它會**靜默跳過**這個元件，讓它以原始方式執行。這是刻意的設計決策：寧可不優化，也不改變語義。

但「靜默失敗」也是 React Compiler 最被批評的地方。你以為編譯器在幫你優化，結果它早就 bail out 了，你完全不知道。

好消息是，你可以開啟 ESLint 規則來偵測 bail-out：

```javascript
// eslint 設定
'react-hooks/todo': 'error',
'react-hooks/unsupported-syntax': 'error',
```

---

## Phase 6：程式碼生成 — 快取的魔法

經過前面所有分析，`codegenFunction()` 終於要把結果寫回 JavaScript 了。

編譯器從 `react/compiler-runtime` 引入一個 `_c()` 函式（底層是 `_useMemoCache`），它回傳一個固定長度的陣列，每個 slot 初始化為一個特殊的 sentinel 值：

```javascript
Symbol.for("react.memo_cache_sentinel")
```

這個 sentinel 保證跟任何合法的 props、state、計算結果都不 `===`，所以第一次渲染一定走「cache miss」路徑。

然後每個 Reactive Scope 生成一組 cache-check 程式碼：

```javascript
import { c as _c } from "react/compiler-runtime";

function ProductCard(t0) {
  const $ = _c(6);           // 6 個快取 slot
  const { product, onClick } = t0;

  let t1;
  if ($[0] !== product.name) {       // slot 0: 依賴值
    t1 = <h2>{product.name}</h2>;    // cache miss → 重新計算
    $[0] = product.name;             // 儲存新的依賴值
    $[1] = t1;                       // 儲存計算結果
  } else {
    t1 = $[1];                       // cache hit → 取快取
  }

  let t2;
  if ($[2] !== onClick) {
    t2 = (id) => onClick(id);
    $[2] = onClick;
    $[3] = t2;
  } else {
    t2 = $[3];
  }

  let t3;
  if ($[4] !== t1 || $[5] !== t2) {  // 多重依賴
    t3 = <div>{t1}<button onClick={t2}>Buy</button></div>;
    $[4] = t1;
    // 注意：結果存在下一個可用的 slot
  } else {
    t3 = $[6];                       // 外層 JSX 也被快取
  }

  return t3;
}
```

模式很清晰：**每個 memo 區塊用兩個以上連續的 slot，偶數 slot 存依賴值，奇數 slot 存計算結果**。用 `!==` 嚴格比較依賴值，miss 就重算，hit 就取快取。

這比 `useMemo` 強在哪裡？

1. **粒度更細** — 不是 memo 整個元件（`React.memo`），而是 memo 個別 JSX 片段
2. **可以跨條件** — 編譯器可以在 `if` 分支或 early return 之後設 memo，Hooks 做不到
3. **自動追蹤依賴** — 不用手寫 dependency array，不怕漏、不怕多

---

## 實戰：Meta 的效能數據與社群實測

### Meta 內部數據

| 產品 | 指標 | 改善 |
|------|------|------|
| Quest Store | 初始載入 | 最高 12% 更快 |
| Quest Store | 特定互動 | 2.5 倍更快 |
| Instagram | 所有頁面平均 | 3% 改善 |

### 社群實測

Sanity Studio 報告渲染效能改善 20-30%。Wakelet 報告 LCP 改善 10%、INP 改善 15%（純 React 元件改善 30%）。

但最有參考價值的是 DeveloperWay 的實測。他們在一個生產應用上測試，361/363 個元件成功編譯。在 9 個已知的 re-render 問題中：

- **2 個** 被完全修復
- **5 個** 部分改善
- **2 個** 完全沒改善

那 2 個沒改善的案例很有教育意義：一個是 gallery card list 搭配 React Query，blocking time 從 130ms 降到 90ms（不錯），但**手動優化**（把物件解構成 primitive 值）直接降到 **0ms**。

原因是 React Query 每次 query key 改變時會回傳新的物件引用。編譯器只做 `===` 比較，物件引用一變就全部 cache miss。手動把物件解構成 primitive（`name`、`id`、`price`），每個值獨立比較，問題就消失了。

**教訓：編譯器不是萬能的。它在元件邊界內做得很好，但跨元件的資料流——特別是第三方 library 回傳的物件——還是需要你動腦。**

---

## Bail-out 陷阱：編譯器偷偷放棄的時候

React Compiler 最陰險的行為就是**靜默 bail-out**。以下是常見的觸發場景：

### 1. 修改解構的 prop

```javascript
// ❌ 編譯器直接放棄
function MyComponent({ value }) {
  value = value ?? fallback;  // 修改了 prop
}

// ✅ 用新變數
function MyComponent({ value: rawValue }) {
  const value = rawValue ?? fallback;
}
```

### 2. try/catch 裡的條件邏輯

```javascript
// ❌ 條件式在 try 裡面 → bail out
try {
  const res = await fetch(url);
  if (res.ok) { setData(await res.json()); }
} catch (e) {
  setError(e);
}
```

編譯器目前無法處理 try/catch 裡面的三元運算子、optional chaining、nullish coalescing。

### 3. "use no memo" 逃生口

如果編譯器搞壞了你的元件，你可以加一個指令暫時跳過：

```javascript
function ProblematicComponent() {
  "use no memo";
  // 這個元件不會被編譯
}
```

這是暫時的 escape hatch，未來應該會被移除。

---

## 你現在該做什麼

**如果你的專案在 React 17/18/19 上：** React Compiler 已經支援這三個版本。但最佳體驗在 React 19。

**如果你想試：** 從 `babel-plugin-react-compiler` 開始。Next.js 15.3.1+ 有原生整合，Vite 透過 `vite-plugin-react` 也支援，Expo SDK 54+ 甚至預設開啟。

**安裝之後第一件事：** 開啟 ESLint 的 `react-hooks/todo` 和 `react-hooks/unsupported-syntax` 規則，看看有多少元件被 bail out。如果數字很高，先修 Rules of React 的違規，再開編譯器。

**不要急著刪掉所有 useMemo/useCallback。** 編譯器在大多數情況下會自動忽略手動 memo（`dropManualMemoization` pass），但在某些邊界案例——特別是 effect 依賴和第三方 library 介面——手動控制還是有價值的。

**最重要的一點：** React Compiler 假設你的程式碼遵守 Rules of React。如果你在 render 裡讀 ref、setState、或做副作用，編譯器不會幫你，反而可能讓你以為一切正常（因為它靜默 bail out 了）。所以，與其擔心「編譯器會不會改壞我的程式碼」，不如先問自己：「我的程式碼有沒有遵守 React 的規則？」

十年磨一劍，65 道 Pass，Meta 內部數萬個元件的實戰驗證。React Compiler 不完美，但它解決了一個真實的問題——讓「寫正確的 React 程式碼」和「寫高效能的 React 程式碼」不再是兩件事。

---

## 延伸閱讀

- [React Compiler v1.0 官方公告](https://react.dev/blog/2025/10/07/react-compiler-1) — 發布公告，含 Meta 內部效能數據
- [React Compiler 官方文件](https://react.dev/learn/react-compiler) — 安裝、設定、除錯的完整指南
- [Understanding React Compiler — Tony Alicea](https://tonyalicea.dev/blog/understanding-react-compiler/) — 非常清晰的編譯流程圖解
- [React Compiler 原始碼 Pipeline.ts](https://github.com/facebook/react/blob/main/compiler/packages/babel-plugin-react-compiler/src/Entrypoint/Pipeline.ts) — 65 道 Pass 的指揮中心
- [SSA 與 Reactivity — Recompiled.dev](https://www.recompiled.dev/blog/ssa/) — 編譯器理論如何應用在 React 上
- [React Compiler 在真實程式碼上的表現 — DeveloperWay](https://www.developerway.com/posts/how-react-compiler-performs-on-real-code) — 最誠實的社群實測報告
- [React Compiler 的靜默失敗與修復方法](https://acusti.ca/blog/2025/12/16/react-compiler-silent-failures-and-how-to-fix-them/) — bail-out 場景完整彙整
- [DESIGN_GOALS.md](https://github.com/facebook/react/blob/main/compiler/docs/DESIGN_GOALS.md) — 編譯器的設計目標與非目標
