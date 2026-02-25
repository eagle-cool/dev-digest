---
title: "React Server Components 序列化協議深度解析：Flight Protocol 的 Wire Format 到底長什麼樣？"
date: 2026-02-25
description: "深入拆解 React Server Components 背後的 Flight Protocol — 從行分隔的 wire format、$L 引用機制、到 Streaming 漸進式解析，帶你看懂 Server 和 Client 之間那層看不見的通訊協議。"
tags: [deep-dive, react, nextjs]
---

你有沒有打開過 Next.js App Router 的 Network tab，看到一堆 `text/x-component` 回應裡面是這種東西：

```
1:HL["/_next/static/media/font.woff2","font",{"crossOrigin":"","type":"font/woff2"}]
3:I["(app-pages-browser)/./node_modules/next/dist/client/components/app-router.js",["app-pages-internals","static/chunks/app-pages-internals.js"],""]
0:["$","html",null,{"lang":"en","children":["$","body",null,{"children":"$L3"}]}]
```

不是 HTML，不是 JSON，也不是 GraphQL。這是 React 團隊自己發明的序列化格式——**Flight Protocol**。它是 React Server Components（RSC）的通訊骨幹，負責把 Server 上 render 出來的元件樹，用一種能 streaming 的格式傳送到 Client 端重建。

React 團隊至今沒有公開文件化這個協議。Dan Abramov 說這是「內部實作細節」。但如果你想真正理解 RSC 的運作原理——不是那種「Server Component 跑在 server 上」的空話，而是**資料到底怎麼從 A 到 B**——你就得看懂 Flight。

今天我們來拆。

---

## 從 `renderToString` 到 Flight：為什麼需要新格式？

先回顧問題。傳統 SSR 的 `renderToString` 把整棵元件樹 render 成一個巨大的 HTML 字串，一次性送到 Client。Client 拿到 HTML 之後，還得下載整包 JavaScript，重新跑一遍所有元件來「hydrate」——本質上是在 client 端重做一次 server 已經做過的事。

這裡有兩個根本問題：

1. **All-or-nothing**：整頁 HTML 要全部生完才能送，一個慢的 database query 就卡住所有人
2. **全量 hydration**：不管元件需不需要互動，JavaScript 全包在 bundle 裡，Client 要全部執行一遍

React 18 的 Streaming SSR（`renderToPipeableStream`）解決了第一個問題——用 `<Suspense>` 把頁面切成獨立的 streaming 單元，先送 fallback HTML，資料好了再補上去。

但 RSC 要解決的是更根本的事：**有些元件永遠不需要送 JavaScript 到 client**。一個純展示的文章內容、一個從 database 讀取的列表——這些元件在 server 執行完就結束了，何必把它們的程式碼塞進 bundle？

問題來了：如果 Server Component 的程式碼不送到 client，那 server render 出來的東西要用什麼格式傳給 client？HTML 不行——因為 client 端的 React 需要的是**元件樹結構**，不是 DOM 字串。它要知道「這裡有一個 `<div>`，props 是這些，children 裡面有一個 Client Component 引用」。

這就是 Flight Protocol 存在的理由：**一個能 streaming、能表達 React 元件樹、能攜帶 Client Component 引用的序列化格式**。

---

## Flight Wire Format：一行一個 chunk 的串流世界

Flight 是一個**行分隔**（line-delimited）的文字協議。每一行是一個獨立的 chunk，格式如下：

```
ROW_ID:TAG_MARKERPAYLOAD\n
```

- **ROW_ID**：十六進位的數字 ID（`0`, `1`, `a`, `2f`）
- **`:`**：分隔符
- **TAG_MARKER**：可選的型別標記（`I`, `HL`, `E` 等）。如果沒有標記，表示是純 JSON model data
- **PAYLOAD**：序列化的資料內容

看個真實的 Next.js 頁面 payload：

```
1:HL["/_next/static/media/c9a5bc6a-s.woff2","font",{"crossOrigin":"","type":"font/woff2"}]
2:HL["/_next/static/css/app/layout.css","style"]
3:I["(app-pages-browser)/./node_modules/next/dist/client/components/app-router.js",["app-pages-internals","static/chunks/app-pages-internals.js"],""]
5:I["(app-pages-browser)/./node_modules/next/dist/client/components/client-page.js",["app-pages-internals","static/chunks/app-pages-internals.js"],"ClientPageRoot"]
a:["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","$L3",null,{"children":"$L5"}]}]}]
```

逐行翻譯：

- **Row 1-2 (`HL`)**：Resource Hints——告訴 client「先 preload 這個字體和這個 CSS」
- **Row 3, 5 (`I`)**：Import/Module 引用——「這些是 Client Component 的模組定義，client 要下載這些 chunk 來 render」
- **Row a**：元件樹的 JSON 序列化——這才是 React 要用的虛擬 DOM 結構

### 核心 Row Tags 一覽

| Tag | 名稱 | 用途 |
|-----|------|------|
| *(無)* | Model/JSON | 序列化的 React 元件樹或資料 |
| `I` | Import | Client Component 模組引用 |
| `HL` | Hint/Link | 資源預載指令（字體、CSS、JS） |
| `E` | Error | 錯誤資訊（含 message 和 stack） |
| `T` | Text | 純文字 chunk（ReadableStream 用） |
| `B` | Binary | 二進位資料 chunk |
| `D` | Debug | 元件 debug 資訊（開發模式） |
| `C` | Completion | Stream 完成標記 |
| `R` / `r` | Stream | ReadableStream 開始標記 |

這不是 HTTP/2 的 frame、不是 Protocol Buffers——是 React 團隊為了自己的需求手搓的格式。簡單到幾乎像是在 debug print，但正是這種簡單讓 streaming 解析變得容易。

---

## `$` 前綴：JSON 的超集魔法

Flight 裡最精妙的設計是它的 **inline 值編碼系統**。在 JSON payload 裡，任何 `$` 開頭的字串都有特殊含義：

```javascript
"$"        → React Element 標記（REACT_ELEMENT_TYPE）
"$3"       → 引用 Row 3 的值
"$L3"      → Lazy 載入引用，指向 Row 3（用於 Client Component）
"$@8"      → Promise 佔位符，Row 8 到達時 resolve
"$F0"      → Server Action 函式引用
"$Sreact.suspense" → Symbol.for("react.suspense")
```

這是 React 把普通 JSON 變成「Progressive JSON」的關鍵。看 Dan Abramov 描述的場景：

```
// Server 先送出框架結構，慢的部分用引用佔位
0:{"header":"$1","post":"$@2","footer":"$3"}

// header 資料先到（database query 很快）
1:"歡迎光臨我的部落格"

// footer 也到了
3:"感謝閱讀"

// post 資料最後才到（慢 query）
2:{"content":"$4","comments":"$@5"}
```

**Row 不需要按順序到達**。Client 端把未填充的引用表示為 Promise——當對應的 Row 到達時，Promise resolve，React 透過 Suspense 機制顯示內容。不需要等全部資料齊全才開始 render。

更多特殊前綴：

| 前綴 | 型別 | 說明 |
|------|------|------|
| `$$` | 跳脫 | 字面值 `"$hello"` |
| `$n123` | BigInt | BigInt 值 |
| `$D2024-01-01` | Date | Date 物件 |
| `$N` | NaN | NaN |
| `$I` / `$-I` | Infinity | 正負無限大 |
| `$u` | undefined | undefined |
| `$Q0` | Map | Map 物件 |
| `$W0` | Set | Set 物件 |

這些前綴讓 Flight 成為 JSON 的超集——能序列化 JSON 原生不支援的型別。React 團隊把它叫做「結構化 clone 的子集」，有點像瀏覽器的 `structuredClone` 但專門為 React 元件樹最佳化。

---

## React Element 的序列化：`["$","div",null,{...}]`

React 元件在 Flight 裡被序列化為一個陣列：

```javascript
["$", type, key, props]
```

- `"$"` 是 `REACT_ELEMENT_TYPE` 的標記
- `type` 是元素類型——字串（`"div"`、`"span"`）或引用（`"$L3"` 指向 Client Component）
- `key` 是 React key 或 `null`
- `props` 是屬性物件，`children` 也在裡面

為什麼不直接序列化成 `{ $$typeof: Symbol.for("react.element"), ... }`？因為 `Symbol` 不能 JSON 序列化。所以 server 用 `"$"` 當魔法標記，client 的 parser 再把它還原成真正的 Symbol。

看一個完整的例子——假設你有這個元件結構：

```jsx
// Server Component
export default function Page() {
  return (
    <div>
      <h1>Hello</h1>
      <ClientButton text="Click me" />
    </div>
  )
}
```

Flight 輸出大概長這樣：

```
2:I["./app/components/Button.tsx",["client-chunk","static/chunks/client.js"],"default"]
0:["$","div",null,{"children":[["$","h1",null,{"children":"Hello"}],["$","$L2",null,{"text":"Click me"}]]}]
```

Row 2 定義了 `ClientButton` 的模組引用（`I` 標記），Row 0 是元件樹——`$L2` 是 Lazy 引用，告訴 client「到了這裡，去載入 Row 2 指向的那個模組來 render」。

---

## Server 端序列化：ReactFlightServer.js 的核心管線

整個序列化引擎在 `packages/react-server/src/ReactFlightServer.js`，核心邏輯大約一千多行。我們來看關鍵流程。

### Request 物件

每次 RSC render 都會建立一個 Request 物件，追蹤整個序列化 session：

```javascript
// 簡化的 Request 結構
type Request = {
  status: OPENING | OPEN | ABORTING | CLOSING | CLOSED,
  nextChunkId: number,              // 下一個可用的 Row ID
  pendingChunks: number,            // 還在等待的 chunk 數
  writtenClientReferences: Map,     // 已寫出的 Client Component 引用（避免重複）
  writtenObjects: WeakMap,          // 已序列化的物件（避免重複）
  completedRegularChunks: Array,    // 完成的一般 chunk（等待 flush）
  completedErrorChunks: Array,      // 完成的錯誤 chunk
  // ...
}
```

### 元件處理分流

當 server 遇到一個 React element，它根據 `type` 決定怎麼處理：

- **Server Component（函式）**：直接在 server 執行，只序列化**輸出結果**，函式本身不會出現在 payload
- **Host Element（`div`, `span`）**：序列化為 `["$","div",key,props]`
- **Client Component（模組引用）**：打包器已經把 import 替換成引用物件，server 發現後產生 `I` Row 並用 `$L` 引用
- **Fragment**：沒有 key 的 Fragment 直接展開 children，有 key 的保留結構
- **Lazy / Memo / forwardRef**：遞迴展開直到拿到實際元件

### Client Component 引用的處理

這裡有個關鍵的協作——不是 React 自己知道哪些是 Client Component，**是打包器告訴它的**。

Webpack 或 Turbopack 在編譯時，會把 `'use client'` 檔案的 import 替換成一個模組引用物件（而不是實際的元件函式）。Server 端的 React 遇到這個引用，呼叫 `resolveClientReferenceMetadata(bundlerConfig, reference)` 拿到打包器產生的 metadata，然後輸出一個 `I` Row：

```
3:I{"id":"./src/ClientButton.tsx","chunks":["client3","client3.chunk.js"],"name":"default","async":false}
```

React 原始碼裡有對應不同打包器的實作目錄：`react-server-dom-webpack/`、`react-server-dom-turbopack/`、`react-server-dom-parcel/`。每個打包器產生的 metadata 格式不同，但 Flight 協議本身是統一的。

### Flush 順序的講究

`flushCompletedChunks` 函式負責把完成的 chunk 寫入串流，而且有優先順序：

1. **Module chunk 先寫**（`I` Row）——讓 client 能盡早開始預載 JS
2. **Regular JSON chunk 次之**——元件樹結構
3. **Error chunk 最後**

這不是隨便排的。Module chunk 先到，client 就能在 parse 元件樹之前開始下載對應的 JS bundle——這是一個隱藏的效能最佳化。

### Outlining：大 Row 的拆分策略

當一個序列化的 Row 超過 `MAX_ROW_SIZE = 3200` 字元，server 會「outline」它——把這個值獨立成一個新的 Row，原本的位置換成 `$id` 引用。這保持每個 Row 體積小，方便 streaming parser 漸進處理。

---

## Client 端反序列化：ReactFlightClient.js 的狀態機

Client 的解析器在 `packages/react-client/src/ReactFlightClient.js`，是一個經典的**有限狀態機**：

```javascript
const ROW_ID = 0;              // 正在解析數字 Row ID
const ROW_TAG = 1;             // 正在解析型別標記
const ROW_LENGTH = 2;          // 正在解析長度（binary row 用）
const ROW_CHUNK_BY_NEWLINE = 3; // 讀到換行為止（text row）
const ROW_CHUNK_BY_LENGTH = 4;  // 讀固定長度（binary row）
```

每個 chunk 在 client 端有自己的生命週期：

```
PENDING → RESOLVED_MODEL → INITIALIZED
                         ↗
PENDING → RESOLVED_MODULE → INITIALIZED
          ↘
           BLOCKED（等待依賴）→ INITIALIZED
          ↘
           ERRORED
```

核心的 `parseModelString` 函式檢查每個字串值的第一個字元：

- 看到 `$`？根據後面的字元分派到對應的處理邏輯
- 看到 `$L`？建立 Lazy wrapper，延遲到實際 render 時才初始化
- 看到 `$@`？建立 Promise，等待對應 Row 到達
- 普通字串？直接用

當 `$L3` 被 render 時，client 查 Row 3 拿到 `I` 類型的模組引用，動態 `import()` 對應的 chunk，取出 named export——Client Component 就準備好了。props 已經在序列化的元件樹裡，直接傳給它。

---

## Streaming + Suspense：不是一次到貨，是逐步交付

Flight 的殺手鐗是**漸進式 streaming**。這不是「等全部好了再送」，而是「好一個送一個」。

### 實際的 Suspense 串流流程

假設你有這個結構：

```jsx
<Suspense fallback={<Spinner />}>
  <SlowDataComponent />
</Suspense>
```

Flight 的處理：

```
// 第一波：馬上送出，SlowDataComponent 還在等 database
1:"$Sreact.suspense"
0:["$","$1",null,{"fallback":"Loading...","children":"$@2"}]

// ... 幾百毫秒後，database 回應了
2:["$","ul",null,{"children":[["$","li",null,{"children":"終於拿到資料了"}]]}]
```

Row 0 先到，client 看到 `$@2` 是一個 Promise 引用，React 知道這裡要 suspend——顯示 fallback `"Loading..."`。

Row 2 隨後到達，Promise resolve，React 用標準的 reconciliation 流程把 fallback 替換成實際內容。**整個過程沒有全頁刷新，client-side state（input focus、scroll position）完全保留。**

### Next.js 的 Payload 傳遞機制

在 Next.js 裡，Flight payload 透過 `<script>` 標籤注入頁面：

```javascript
self.__next_f.push([1, "0:[\"$\",\"div\",null,{\"children\":\"Hello\"}]\n"])
self.__next_f.push([1, "2:[\"$\",\"ul\",null,{\"children\":\"World\"}]\n"])
```

`[1, data]` 表示 payload data、`[0, ...]` 是初始化、`[2, ...]` 是 form state。內容以 2048 字元為上限分割成多個 script tag，實現 HTML streaming 過程中的漸進式 Flight payload 注入。

### 雙重資料問題

Next.js 實際上同時送**兩套東西**：

1. **HTML**：給瀏覽器直接 render，實現快速的 First Contentful Paint
2. **Flight Payload**：給 React 重建虛擬 DOM，用於後續的 reconciliation 和互動

這就是所謂的「double data」問題——總頻寬會增加。但換來的是：你看到頁面的速度更快（HTML 秒出），而 React 在背景用 Flight payload 建立可互動的元件樹。

---

## `use client` 邊界的真相

關於 RSC，最常見的誤解就是 `use client` 和 `use server`。讓我把事情說清楚。

**Server Component 不需要任何 directive**——它是預設值。`use server` **不是**用來宣告 Server Component 的，它是用來宣告 Server Function（Server Action）的。

`use client` 標記的不是「這個元件只跑在 client」，而是「從這裡開始是 client 邊界」。**Client Component 在初次載入時仍然會被 SSR**——它在 server 上執行一次生成 HTML，然後在 client 上 hydrate。`use client` 的意思是：這個元件需要 client 端的能力（state、effect、event handler），所以它的 JavaScript 要包在 bundle 裡。

### 可以跨邊界傳遞的型別

Flight 支援序列化的型別是有限的：

**可以傳：** `string`、`number`、`boolean`、`null`、`undefined`、`bigint`、`Date`、`Map`、`Set`、`Array`、`TypedArray`、`FormData`、`Promise`（可以在 server 發起、client 用 `use()` 消費）、`ReactNode`（已 render 的 JSX）

**不能傳：** 一般函式、class instance、`RegExp`、`Error` 物件、非全域 Symbol

函式不能序列化很好理解。`Error` 不能傳是安全考量——server 端的錯誤可能包含敏感資訊。`RegExp` 被擋是因為有安全顧慮（ReDoS 相關的實作漏洞）。

React 團隊明確拒絕了「可插拔的序列化系統」提案，理由是「我們擔心這對生態系的複雜度影響，以及元件無法在不同環境重用」。

---

## 效能的冷水：RSC 不是銀子彈

讓我丟一個可能不受歡迎的觀點：**RSC 本身的效能提升可能比你想像的小。**

developerway.com 做過一組嚴謹的 benchmark（6x CPU throttle + Slow 4G），結論是：

> 「Server Components 單獨使用時，如果應用是 Client 和 Server 元件的混合體，並不會顯著改善效能。它們對 bundle size 的減少不足以產生可測量的效能影響。」

真正帶來效能飛躍的是 **Streaming + Suspense**，不是 RSC 本身。在他們的測試中，App Router + RSC + Suspense 的 LCP 是 1.28 秒，而純 SSR + Server Fetch 是 2.16 秒——但差距主要來自 streaming 讓內容提前到達，不是因為少了幾 KB 的 bundle。

Bundle size 的減少在真實應用裡通常很有限。共享的 chunk（React runtime、router、UI library）不會因為你用了 RSC 就消失。真正受益的場景是：你的 Server Component 用了很重的 server-only library（像是 markdown parser、syntax highlighter），這些程式碼完全不進 bundle——這才是實質的節省。

另一個被忽略的 trade-off：RSC + Streaming 有時候反而**增加** Time to Interactive。因為 JavaScript 的下載被延後到 CSS 之後，中間會有一段「看得到、摸不著」的空窗期——使用者看到內容了，但按鈕還不能點。

---

## 結語：協議是骨架，理解它才能做出正確的架構決策

Flight Protocol 是一個沒有公開文件、隨時可能改變的內部協議。React 團隊說「這是實作細節」，他們說得對——你不應該依賴它的格式。

但理解它的設計哲學很有價值：

1. **行分隔 + 引用系統** = 天然適合 streaming，不需要等整個 payload 到齊
2. **Module chunk 優先 flush** = client 能在 parse 完元件樹之前就開始下載 JS
3. **`$L` Lazy 引用** = Client Component 只在需要時才載入，自動 code splitting
4. **`$@` Promise 引用** = Suspense 的 server-side 實現，async component 變成漸進式交付

下次你在做架構決策時——「這個元件該放 server 還是 client？」「Suspense 邊界要切在哪裡？」——想想 Flight payload 的流動方式。Module chunk 先到、Promise 引用靠 Suspense 消費、Server Component 的程式碼完全不進 payload。

具體建議：打開 [RSC Explorer](https://rscexplorer.dev) 或 [RSC Parser](https://rsc-parser.vercel.app)，把你自己 app 的 Flight payload 丟進去看看。你會發現很多「原來是這樣」的瞬間。

---

## 延伸閱讀

- [Progressive JSON](https://overreacted.io/progressive-json/) — Dan Abramov 解釋漸進式 JSON 的核心概念，理解 Flight streaming 的設計哲學必讀
- [Introducing RSC Explorer](https://overreacted.io/introducing-rsc-explorer/) — 互動式 RSC 協議視覺化工具，直接拿你的 payload 來拆解
- [How React Server Components Work: An In-Depth Guide](https://www.plasmic.app/blog/how-react-server-components-work) — Plasmic 團隊寫的完整 wire format 分析，含 Row type 解說
- [RSC From Scratch Part 1](https://github.com/reactwg/server-components/discussions/5) — React Working Group 的從零建構 RSC 教學
- [ReactFlightServer.js 原始碼](https://github.com/facebook/react/blob/main/packages/react-server/src/ReactFlightServer.js) — Server 端序列化引擎的完整實作
- [ReactFlightClient.js 原始碼](https://github.com/facebook/react/blob/main/packages/react-client/src/ReactFlightClient.js) — Client 端反序列化狀態機
- [Flight 編碼重構 PR #26082](https://github.com/facebook/react/pull/26082) — Sebastian Markbåge 重新設計 wire format 的 PR，理解格式演進
- [React Server Components 效能實測](https://www.developerway.com/posts/react-server-components-performance) — developerway.com 的嚴謹 benchmark，用數據說話
