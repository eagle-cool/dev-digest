---
title: "Next.js RSC 完整資料流：從 Server Component Render 到 Client Hydration 的每一步"
date: 2026-03-05
description: "深入解析 React Server Components 在 Next.js 中的完整資料流：Flight 協議的 wire format、RSC Payload 的行格式、self.__next_f 的串流機制、Client Component 的 module reference 編碼，以及 Suspense 邊界如何實現漸進式 hydration。"
tags: [deep-dive, nextjs, react, frontend]
---

你在 Next.js App Router 裡寫了一個 Server Component，裡面 `await` 了一個資料庫查詢，然後把結果傳給一個帶 `'use client'` 的 Client Component。

問題來了：Server Component 在 Node.js 裡跑完之後，它的「渲染結果」到底是什麼？不是 HTML——那只是其中一半。不是 JSON——太粗糙了。React 為了這件事發明了一套完整的 wire protocol，叫 **React Flight**。

今天我們就從 Server Component 的第一行程式碼開始，追蹤資料流經過的每一個節點，直到它在瀏覽器裡變成可互動的 UI。如果你讀完上一篇對 App Router 路由解析的拆解，今天就是那棵 LoaderTree 被「渲染」之後的故事。

---

## 雙軌輸出：HTML 和 RSC Payload

傳統 SSR 的故事很簡單：Server 把元件跑一遍 → 吐 HTML → 瀏覽器收到 HTML → React hydrate 接手。但 React Server Components 改變了這個架構——Server 現在同時輸出**兩份東西**：

1. **HTML**：給瀏覽器立即顯示的靜態骨架，讓使用者不用等 JavaScript 載入就能看到內容
2. **RSC Payload**：一份序列化的 Virtual DOM 樹，讓 React 在 client 端知道「這棵樹長什麼樣、哪些地方需要 hydrate」

為什麼需要兩份？因為 HTML 沒有辦法表達 React 的元件邊界。瀏覽器收到 `<div><h1>Hello</h1></div>` 的時候，它不知道這個 `<div>` 是來自 `<Layout>` 還是 `<Page>`，也不知道哪個 `<button>` 是 Client Component 需要掛事件的。

RSC Payload 就是那張「地圖」——它告訴 React：「`<div>` 的第二個 child 是一個 Client Component，它的 JavaScript 在 `static/chunks/app/page.js` 裡，props 是 `{count: 0}`。」

```
Server Component
      │
      ▼
 ┌────────────┐
 │  React 渲染  │
 │  Pipeline   │
 └────┬───┬────┘
      │   │
      ▼   ▼
    HTML   RSC Payload
      │   │
      ▼   ▼
   瀏覽器收到兩份資料
      │
      ▼
  React 用 RSC Payload
  對照 HTML 做 hydration
```

這個「雙軌」架構是理解 RSC 的關鍵。忘掉「SSR 就是吐 HTML」的舊思維——現代 Next.js 吐的是 HTML + 一份完整的元件樹描述。

---

## Flight 協議：React 發明的 Wire Format

React Flight 是 RSC 的序列化協議，原始碼分散在 React 倉庫的幾個核心檔案裡：

- `ReactFlightServer.js`：Server 端序列化引擎
- `ReactFlightClient.js`：Client 端反序列化引擎
- `ReactFlightReplyServer.js`：Client → Server 的回傳解碼（Server Actions 用）
- `ReactFlightReplyClient.js`：Server → Client 的回傳編碼

Flight 的設計目標很明確：**逐行串流、可增量解析**。不像 JSON 必須等整個物件結束才能 parse，Flight payload 是 line-delimited 的——每收到一行就能處理一行。

### 行格式解析

每一行的格式是：

```
ROW_ID:ROW_TAG[ROW_DATA]\n
```

- **ROW_ID**：唯一識別符（數字或十六進位），用來讓後續行引用先前定義的資料
- **ROW_TAG**：型別標記，告訴 parser 怎麼解讀這行資料
- **ROW_DATA**：JSON 編碼的實際內容

### 三種核心 Row Type

**HL（Hints）— 資源提示**

```
1:HL["/_next/static/media/c9a5bc6a7c948fb0-s.p.woff2","font",{"crossOrigin":"","type":"font/woff2"}]
```

HL 行會被轉換成 `<link rel="preload">` 標籤。Next.js 用它來提前載入字型、CSS、關鍵 JavaScript。這些 hint 會排在 payload 的最前面，因為瀏覽器越早知道要載入什麼資源越好。

**I（Imports）— 模組引用**

```
5:I["(app-pages-browser)/./node_modules/next/dist/client/components/app-router.js",["app-pages-internals","static/chunks/app-pages-internals.js"],"default"]
```

I 行定義了 Client Component 的模組位置。三個欄位分別是：
- 模組路徑（bundler 的模組 ID）
- chunk 陣列（要載入哪些 JavaScript 檔案）
- 匯出名稱（`"default"` 或具名匯出）

這是 `'use client'` 的底層實現——bundler 在編譯時期把 Client Component 的 `import` 替換成 module reference，Server 端只保留一個「指向 client bundle 的指標」，而不是把 Client Component 的程式碼送出去。

**數字行 — DOM 與元件樹定義**

```
7:["$","main",null,{"className":"page_main__GlU4n","children":[["$","h1",null,{"children":"Hello"}],["$","@5",null,{"count":0}]]}]
```

這行描述了一棵 Virtual DOM 子樹。`"$"` 表示 React element，後面依序是：
- 元素型別（`"main"`、`"h1"` 是 HTML 原生元素）
- key
- props

注意 `"@5"` 這個寫法——它引用了 Row ID 5 定義的 I 行（也就是前面的 `app-router.js`）。這就是 Client Component 在 RSC Payload 裡的表示方式：**不是程式碼，而是一個指向 bundle 的引用加上序列化的 props**。

### 特殊標記

Flight 協議還有幾個特殊前綴，出現在 props 值裡面：

```javascript
"$"  → React element（遞迴結構）
"@"  → 引用已定義的 Row（module reference 或 lazy node）
"$S" → Symbol.for("react.suspense")
"$L" → Lazy node（Promise，尚未 resolve）
"$RE" → React Element 的 $$typeof symbol
```

這些標記讓 Flight 能夠表達 JSON 原本不支援的型別——Symbol、Promise、React element reference——全部壓縮成字串前綴。

---

## Server 端渲染管線：從元件到 Payload

當一個 request 進來，Next.js 的渲染管線大致分四個階段：

### 階段一：解析 Server Component 樹

```
renderJSXToClientJSX(jsx)
```

React 從根元件開始遞迴執行。這個函式的邏輯（簡化版）是：

```javascript
async function renderJSXToClientJSX(jsx) {
  if (typeof jsx.type === "string") {
    // HTML 原生元素 — 保留，遞迴處理 children
    return {
      ...jsx,
      props: await renderJSXToClientJSX(jsx.props),
    };
  } else if (typeof jsx.type === "function") {
    // Server Component — 呼叫它！
    const Component = jsx.type;
    const returnedJsx = await Component(jsx.props);
    return renderJSXToClientJSX(returnedJsx);
  }
  // Client Component — 替換成 module reference
  // ...
}
```

關鍵在第二個分支：Server Component 被**直接呼叫**，它的回傳值繼續遞迴處理，直到整棵樹只剩下 HTML 原生元素和 Client Component 的 module reference。

Server Component 的程式碼在這裡就「消失」了——它執行完就只剩下產出的 JSX 結構。這就是為什麼 Server Component 的程式碼不會送到瀏覽器：**它們在 Server 端已經被「執行並展開」了**。

### 階段二：序列化成 Flight 格式

展開後的樹被送進 `ReactFlightServer` 的序列化引擎。這個引擎做幾件事：

1. **Module Reference 收集**：遇到 Client Component 就註冊一個 I 行，並用 `$` + ID 替代原本的元件函式
2. **Chunk 優先排序**：先送 HL（資源提示），再送 I（模組定義），最後送 DOM 結構。因為瀏覽器需要先知道要載入什麼資源
3. **Promise 處理**：遇到 async 元件回傳的 Promise，先發一個 `$L`（Lazy node）佔位，等 Promise resolve 後再發 resolve 的那行

```
渲染順序：
HL 行 → 字型、CSS（最先送出）
 ↓
I 行 → Client Component module references
 ↓
DOM 行 → 元件樹結構（引用前面定義的 I 行）
 ↓
Lazy resolve 行 → Suspense 內的非同步內容
```

### 階段三：HTML 生成

同時，React 也用 `renderToReadableStream`（或對應的 SSR API）把同一棵樹渲染成 HTML。這份 HTML 會立即送到瀏覽器——使用者可以在 JavaScript 還沒載入的時候就看到頁面內容。

### 階段四：串流輸出

HTML 和 RSC Payload 會被**交錯串流**到瀏覽器。Next.js 使用 `Transfer-Encoding: chunked`，所以這兩份資料不需要等全部準備好才送出。

---

## self.__next_f：Next.js 的 Payload 傳遞機制

RSC Payload 送到瀏覽器後，必須有個方式讓 React 讀到它。Next.js 的做法是把 payload 嵌在 `<script>` 標籤裡：

```html
<script>self.__next_f.push([1,"1:HL[\"/_next/static/media/font.woff2\",\"font\",...]\n"])</script>
<script>self.__next_f.push([1,"5:I[\"./app/page.js\",...]\n7:[\"$\",\"main\",...]\n"])</script>
```

`self.__next_f` 是一個陣列，`push` 方法在 `next/src/client/app-index.tsx` 裡被覆寫了。每次 push，覆寫的函式就會把資料丟給 RSC 的 client-side parser。

push 的陣列第一個元素是 payload type：

| Type | 名稱 | 用途 |
|------|------|------|
| 0 | BOOTSTRAP | 初始化資料 |
| 1 | DATA | 主要 RSC Payload（最常見）|
| 2 | FORM_STATE | Server Actions 的 form 狀態 |
| 3 | BINARY | 二進位資料 |

第二個元素是字串，包含一到多行 RSC Payload row。

### 2KB 分割策略

Next.js 會把 payload 切成 **2KB** 左右的 chunk。如果一個 row 太長（比如一個大元件的 props 包含大量資料），它會被切成多個 script tag：

```html
<script>self.__next_f.push([1,"7:[\"$\",\"div\",null,{\"children\":\"這是一段很長的"])</script>
<script>self.__next_f.push([1,"內容，被切斷了\"}]\n"])</script>
```

兩個 push 的字串會被拼接起來，然後才做行解析。這個機制確保即使 payload 很大，瀏覽器也能逐步接收和處理，而不是卡在一個巨大的 script tag 上。

---

## Client Component 的邊界：'use client' 的底層真相

當 bundler（Turbopack 或 webpack）遇到 `'use client'` 指令，它做的事比你想像的更根本——它在那個檔案畫了一條**模組圖的邊界線**。

```
Server Module Graph          Client Module Graph
┌──────────────────┐        ┌──────────────────┐
│ page.tsx          │        │ Counter.tsx       │
│ layout.tsx        │   ──→  │ Button.tsx        │
│ lib/db.ts         │  引用   │ hooks/useState.ts │
└──────────────────┘        └──────────────────┘
                    'use client' 是邊界
```

在 Server Module Graph 裡，`import Counter from './Counter'` 不會引入 Counter 的實際程式碼。Bundler 會把這個 import 替換成一個 **module reference 物件**：

```javascript
{
  $$typeof: Symbol(react.module.reference),
  name: "default",        // 匯出名稱
  filename: "./src/Counter.tsx"  // 原始路徑
}
```

當 React Flight Server 在序列化的時候遇到這個物件，它就知道：「這個位置要放一個 I 行，告訴 client 端去載入對應的 JavaScript 模組。」

這個設計的精妙之處在於——**Server Component 可以把 props 傳給 Client Component，但這些 props 必須是可序列化的**。函式、class instance、DOM node 這些東西不能當 props 傳過去，因為它們沒辦法被 JSON stringify。

但有一個例外：**Server Component 可以當作 children 傳給 Client Component**。

```jsx
// Server Component
export default function Page() {
  return (
    <ClientWrapper>
      <ServerContent data={await fetchData()} />
    </ClientWrapper>
  );
}
```

為什麼這行得通？因為 `<ServerContent>` 會在 Server 端被執行並展開成 HTML 原生元素，然後這個展開後的結果會被序列化到 RSC Payload 裡，作為 `ClientWrapper` 的 `children` prop。Client Component 收到的不是一個 Server Component 函式，而是已經渲染好的 Virtual DOM 片段。

---

## Suspense 邊界與漸進式串流

Suspense 在 RSC 架構裡的角色不只是顯示 loading spinner——它定義了串流的切割點。

### 串流時序

假設你有這個結構：

```jsx
export default async function Page() {
  return (
    <main>
      <h1>Dashboard</h1>
      <Suspense fallback={<Skeleton />}>
        <SlowDataComponent />
      </Suspense>
    </main>
  );
}
```

Server 的串流順序是：

1. **立即送出**：`<main>` 和 `<h1>Dashboard</h1>` 的 HTML + RSC Payload
2. **同時送出**：Suspense 的 fallback `<Skeleton />` 的 HTML，以及在 RSC Payload 裡放一個 Lazy node（`$L`）
3. **等 SlowDataComponent await 完成後**：送出 resolve 後的 HTML 片段和對應的 RSC Payload 行

### 瀏覽器端的替換機制

瀏覽器收到 fallback 的 HTML 時，React 會插入一個帶特殊 ID 的 `<template>` 標籤當佔位符：

```html
<template id="B:0"></template>
<!-- fallback content -->
<div class="skeleton">Loading...</div>
<!--/$-->
```

當 Server 端 resolve 完畢，它會串流一段 inline script：

```javascript
$RC = function(b, c, e) {
  c = document.getElementById(c);  // 找到 fallback
  c.parentNode.removeChild(c);      // 移除 fallback
  var a = document.getElementById(b); // 找到 template 佔位符
  // 把 resolve 後的 HTML 插入正確位置
};
$RC("B:0", "S:0");
```

這個 `$RC` 函式（`$RC` 大概是 "Replace Content" 的縮寫）會把 fallback 的 DOM 替換成最終內容——**不需要 React hydration，純 DOM 操作**。

同時，RSC Payload 裡的 Lazy node 也會 resolve：

```
// 之前送出的
7:["$","$Sreact.suspense",null,{"fallback":["$","div",null,{"className":"skeleton"}],"children":"$L8"}]

// SlowDataComponent resolve 後送出的
8:["$","div",null,{"children":"Actual data here"}]
```

Row 8 的資料到了 client 端，React 就知道 `$L8` 這個 Lazy reference 已經 resolve 了，可以更新 Virtual DOM。

### 亂序 resolve

Suspense 的強大在於它允許 **out-of-order resolution**。如果你有三個 Suspense 邊界，第三個可能比第一個先 resolve。每個邊界獨立串流，不需要等前面的完成。這是行格式（line-delimited）的設計優勢——每行用 ROW_ID 自我標識，接收端不關心到達順序。

---

## Client 端 Hydration：拼圖的最後一塊

當所有 RSC Payload 都到齊，React 在 client 端做的事是：

### 步驟一：重建 Virtual DOM

Flight Client 解析所有行，用 `parseModelString` 把 `"$"`、`"@"`、`"$L"` 這些標記還原成 React element 物件。module reference（`@` 前綴）會被替換成實際載入的 Client Component 模組。

### 步驟二：Hydration

React 呼叫 `hydrateRoot`，拿重建好的 Virtual DOM 去比對瀏覽器裡已經存在的 HTML DOM。如果兩邊一致（正常情況），React 只需要「接手」這些 DOM 節點並掛上事件處理器，不需要重新建立任何 DOM 元素。

如果不一致——恭喜你遇到了 hydration error。這通常是因為 Server 和 Client 的渲染結果不同（比如在 Server Component 裡用了 `Date.now()`，Client 重新執行的時候得到不同的值）。

### 步驟三：Client Component 初始化

對那些被標記為 Client Component 的節點，React 會：

1. 載入 I 行指定的 JavaScript chunk
2. 取得模組的匯出（`default` 或具名）
3. 用 RSC Payload 裡序列化的 props 初始化元件
4. 掛載狀態（useState）、事件監聽器等

從這一刻起，Client Component 就有了自己的生命週期——它可以有 state、可以用 useEffect、可以回應使用者互動。而 Server Component 渲染的那些靜態內容，就只是 DOM 節點，永遠不會重新執行。

---

## 導航時的資料流：不用重載的更新

首次載入之後，當使用者用 `<Link>` 點擊導航，Next.js **不會**重新載入頁面。它只做一件事：向 Server 發一個 request，拿新頁面的 RSC Payload。

```
GET /dashboard?_rsc=xxxxx
Accept: text/x-component
```

Server 回傳的不是 HTML，而是純粹的 RSC Payload。Client 端的 React 拿到這份 payload 後，走正常的 reconciliation 流程——diff 新舊 Virtual DOM、更新需要變動的 DOM 節點、重新掛載新的 Client Component。

這就是為什麼 Next.js 的 client-side 導航那麼流暢：不用重新解析整頁 HTML、不用重新載入已經有的 JavaScript、Layout 不會被重新 mount（因為 RSC Payload 的 Layout 部分跟上一次一樣，diff 的結果是「不需要動」）。

Server Actions 也是這個流程的延伸。當你呼叫一個 Server Action，Server 端可以回傳更新過的 RSC Payload，React 會自動 reconcile，受影響的 Server Component 就「重新渲染」了——但只在 Server 端執行。

---

## 實戰：RSC Payload 的效能陷阱

理解了資料流，就能理解一些常見的效能問題。

### 陷阱一：Payload 膨脹

RSC Payload 會序列化所有從 Server Component 傳給 Client Component 的 props。如果你把整個資料庫查詢結果當 props 傳下去，這些資料會**完整出現在 HTML 裡**（透過 `self.__next_f.push`）。

```jsx
// 這會讓 payload 爆炸
export default async function Page() {
  const allProducts = await db.products.findMany(); // 5000 筆
  return <ProductList products={allProducts} />;
}
```

`allProducts` 的每一筆都會被序列化兩次：一次在 HTML 裡渲染成使用者可見的列表，一次在 RSC Payload 裡作為 props。

解法：只傳需要的欄位，做分頁，或者把資料查詢移到 Client Component 內部。

### 陷阱二：不小心把 Server Component 變成 Client Component

在一個 `'use client'` 元件裡 import 另一個元件，那個元件**自動變成 client bundle 的一部分**——即使它從來沒用過 `useState` 或任何 client API。`'use client'` 是一條單向的傳染邊界。

### 陷阱三：Suspense 邊界太少

沒有 Suspense 邊界，整個頁面必須等最慢的 async 元件 resolve 才能送出任何內容。每一個獨立的資料查詢都應該被包在 Suspense 裡，這不只是 UX 問題——它直接影響 Time to First Byte。

---

## 結語

RSC 在 Next.js 裡的資料流可以總結成一句話：**Server Component 執行完變成序列化的 Virtual DOM，透過 Flight 協議串流到瀏覽器，和預渲染的 HTML 合體完成 hydration**。

如果你只記一件事，記這個：RSC Payload 不是「額外的開銷」——它是整個架構的核心傳輸層。HTML 負責使用者體驗（First Contentful Paint），RSC Payload 負責 React 的認知（元件邊界、props、hydration 指引）。兩者缺一不可。

下次你在 DevTools 的 Network tab 看到一堆 `?_rsc=` 的 request，或者 view source 看到滿頁的 `self.__next_f.push`，你就知道那是什麼了——那是 React Flight 在跑。

---

## 延伸閱讀

- [RSC From Scratch. Part 1: Server Components](https://github.com/reactwg/server-components/discussions/5) — Dan Abramov 的 first-principles 教學，從零實現一套簡化版 RSC
- [How React Server Components Work: An In-Depth Guide](https://www.plasmic.app/blog/how-react-server-components-work) — 對 Flight 格式和 module reference 有完整的拆解
- [The Forensics of React Server Components](https://www.smashingmagazine.com/2024/05/forensics-react-server-components/) — 用「鑑識」的角度逆向分析 RSC Payload 的每一行
- [Understanding React Server Components](https://tonyalicea.dev/blog/understanding-react-server-components/) — 搭配視覺化圖表解釋 Flight Payload 的序列化流程
- [Decoding React Server Component Payloads](https://edspencer.net/2024/7/1/decoding-react-server-component-payloads) — 詳細解釋 `self.__next_f` 和 row type 的規格
- [Next.js 官方文件：Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components) — 官方對 Server/Client 邊界的說明
