---
title: "React Suspense 不只是 Loading Spinner：Streaming SSR 如何從根本改變渲染架構"
date: 2026-02-28
description: "深入拆解 React Suspense 的 thrown promise 機制、Fizz 引擎的 HTML 串流注入原理、Selective Hydration 的事件驅動優先排程，以及 React 19 sibling rendering 爭議。從原始碼層級理解 Suspense 如何讓 SSR 從「全有或全無」進化成漸進式架構。"
tags: [deep-dive, react, nextjs, ssr]
---

你大概用過 `<Suspense fallback={<Spinner />}>`，然後覺得「喔，就是個 loading 狀態的語法糖嘛」。

如果你真的這樣想，那你只看到了冰山的 1%。Suspense 是 React 18 以來最重要的架構原語——它重新定義了 Server-Side Rendering 的運作方式，讓瀏覽器能在 React JavaScript 都還沒載入的情況下就開始替換頁面內容，還能根據使用者的點擊行為動態決定哪些元件要優先 hydrate。

今天我們要從原始碼層級拆解這一切。

---

## 傳統 SSR 的三個致命瓶頸

在 Suspense 之前，React 的 SSR 靠 `renderToString` 運作。這個 API 的問題不是「慢」，而是架構上的根本限制——Dan Abramov 在 React 18 Working Group 裡把它總結成三句話：

1. **Fetch everything before you can show anything** — 伺服器必須拿到所有資料，才能開始產生任何 HTML
2. **Load everything before you can hydrate anything** — 所有 JavaScript 必須載入完畢，才能開始 hydrate 任何元件
3. **Hydrate everything before you can interact with anything** — 整棵元件樹必須全部 hydrate 完，使用者才能跟任何元件互動

這三個限制的共同點是「全有或全無」。一個慢的 API 呼叫會卡住整個頁面的 TTFB；一個大的 JS bundle 會延遲整個頁面的互動。你的 navbar 早就渲染好了，但使用者就是不能點，因為頁面底部的留言區還在 hydrate。

```
傳統 SSR 時間軸（一切都是序列的）：

伺服器：  [===== 全部取資料 =====][===== 全部渲染 HTML =====]
                                                              ↓ 送出完整 HTML
瀏覽器：                                                      [=== 載入全部 JS ===][=== Hydrate 全部 ===]
                                                                                                         ↓ 可互動
```

Suspense + Streaming SSR 的目標就是把這三個「全部」拆開，讓每個部分都能獨立進行。

---

## Thrown Promise：React 最不正統的設計決策

Suspense 的核心機制，說出來可能會讓你皺眉：**元件透過 `throw` 一個 Promise 來告訴 React「我還沒準備好」**。

沒錯，用 `throw`。不是 return null，不是 set state，是直接把一個 Promise 丟出去當作「異常」。React 的 reconciler 在 render 階段的錯誤處理路徑中攔截它，判斷這不是真正的錯誤，而是一個「暫停信號」。

看看 React 原始碼中 `throwException` 的簡化邏輯：

```javascript
// packages/react-reconciler/src/ReactFiberThrow.js（簡化版）
function throwException(root, returnFiber, sourceFiber, value) {
  if (
    value !== null &&
    typeof value === 'object' &&
    typeof value.then === 'function'
  ) {
    // 這是一個 thenable（Promise）——元件暫停了
    const wakeable = value;

    // 往上找最近的 Suspense 邊界
    const suspenseBoundary = getNearestSuspenseBoundaryToCapture(returnFiber);

    if (suspenseBoundary !== null) {
      // 標記這個邊界要顯示 fallback
      markSuspenseBoundaryShouldCapture(suspenseBoundary, ...);
      // 當 Promise resolve 時通知 React 重新嘗試
      attachPingListener(root, wakeable, rootRenderLanes);
      attachRetryListener(suspenseBoundary, root, wakeable, rootRenderLanes);
      return;
    }
  }
  // 如果不是 Promise，就當作真正的錯誤處理
}
```

現代的 `use()` Hook 就是這個機制的官方封裝：

```javascript
function Comments({ articleId }) {
  const comments = use(fetchComments(articleId));
  // 如果 Promise 還沒 resolve，use() 會 throw 它
  // 如果已經 resolve，直接返回結果
  return (
    <ul>
      {comments.map(c => <li key={c.id}>{c.text}</li>)}
    </ul>
  );
}
```

### ShouldCapture → DidCapture：狀態機的兩步轉換

當 React 偵測到 thrown promise 後，並不是立刻切換到 fallback。它用了一個兩階段的 flag 狀態機：

```javascript
// 階段一：在 throwException 中設定 ShouldCapture
markSuspenseBoundaryShouldCapture(suspenseBoundary, ...);

// 階段二：在 completeWork / unwindWork 中轉換
case SuspenseComponent:
  if (flags & ShouldCapture) {
    // 把 ShouldCapture 換成 DidCapture，觸發重新渲染
    workInProgress.flags = (flags & ~ShouldCapture) | DidCapture;
    return workInProgress; // 重新進入 render 階段
  }
```

在 `updateSuspenseComponent` 中，`DidCapture` flag 決定了要渲染 primary children 還是 fallback：

```
狀態轉換表：

之前顯示    現在狀態        動作
────────    ────────       ──────
內容        內容           正常 reconciliation
內容        暫停(DidCapture)  切換到 fallback
fallback    暫停           更新 fallback children
fallback    解決           恢復 primary children
```

有個關鍵細節：React 不會丟掉暫停中的 primary children。它們被包在一個 `OffscreenComponent` fiber 裡，以 `mode: "hidden"` 保留在 fiber tree 中。這意味著當 Promise resolve 後，元件可以恢復之前的 state，不會從頭開始。

這和 Error Boundary 形成了對稱關係：Suspense 攔截 thrown promise，Error Boundary 攔截 thrown error。兩者共用 React 的錯誤處理路徑，但走不同的分支。如果一個 Promise reject 了，React 會找 Error Boundary 而不是 Suspense。

---

## Fizz 引擎：不需要 React JS 就能運作的 HTML 串流

Suspense 真正改變 SSR 架構的地方，是 React 的 Fizz 串流渲染引擎和 `renderToPipeableStream` API。

### Shell 的概念

Fizz 把你的元件樹分成兩部分：

- **Shell**：所有 `<Suspense>` 邊界之外的內容——即時送出
- **Deferred**：`<Suspense>` 邊界之內的內容——資料好了再串流送出

```jsx
function Page() {
  return (
    <html>
      <body>
        <Nav />                              {/* ← Shell */}
        <main>
          <h1>文章標題</h1>                    {/* ← Shell */}
          <Suspense fallback={<Skeleton />}>
            <Comments />                      {/* ← Deferred */}
          </Suspense>
        </main>
      </body>
    </html>
  );
}
```

### 串流的三步流程

**第一步：送出 Shell**

當 `onShellReady` callback 觸發時，React 已經渲染好 Shell。伺服器開始送出 HTML，但故意**不送 `</body>` 和 `</html>` 的結束標籤**——HTTP 連線保持開啟。瀏覽器的容錯解析器會自動補上缺失的標籤開始渲染。

```html
<!-- 第一個 chunk -->
<!DOCTYPE html>
<html><body>
  <nav>...</nav>
  <main>
    <h1>文章標題</h1>
    <!--$?--><template id="B:0"></template><div>Loading...</div><!--/$-->
  </main>
  <!-- 注意：沒有 </body></html> -->
```

看到那個 `<!--$?-->` 了嗎？那是 Suspense 邊界的標記。`<template id="B:0">` 是佔位符，`$?` 表示「這個邊界正在等待內容」。

**第二步：注入延遲內容**

當 Comments 元件的資料終於到了，React 在同一個 HTTP 回應中繼續送出新的 HTML chunk：

```html
<!-- 後續 chunk -->
<div hidden id="S:0">
  <ul>
    <li>寫得太好了！</li>
    <li>終於有人把這個講清楚了</li>
  </ul>
</div>
<script>$RC("B:0","S:0")</script>
```

一個隱藏的 `<div>` 裝著真正的內容，加上一個內聯 `<script>` 呼叫 `$RC` 函式。

**第三步：DOM 替換——在 React JS 載入之前！**

這是最精妙的部分：**`$RC` 函式在 React 的 JavaScript bundle 載入之前就執行了**。它是一個極小的內聯腳本，直接操作 DOM：

```javascript
// React Fizz 指令集（簡化版）
// 來自 ReactDOMFizzInstructionSetShared.js

function completeBoundary(suspenseBoundaryID, contentNodeID) {
  const suspenseNode = document.getElementById(suspenseBoundaryID);
  const contentNode = document.getElementById(contentNodeID);

  // 找到 fallback 內容的範圍（兩個 comment node 之間）
  // 移除 fallback
  // 把真正的內容從隱藏的 div 搬過來
  const parentInstance = suspenseNode.parentNode;
  while (contentNode.firstChild) {
    parentInstance.insertBefore(contentNode.firstChild, suspenseNode);
  }
  // 移除佔位符
  suspenseNode.remove();
  contentNode.remove();
}
```

還有一個進階版 `completeBoundaryWithStyles`，它會處理 CSS 依賴——先構建 `<link>` 和 `<style>` 標籤，用 `Promise.all()` 等所有 stylesheet 載入完畢，才替換內容。這避免了 FOUC（Flash of Unstyled Content）。

```
Streaming SSR 時間軸（平行 + 漸進）：

伺服器：  [= Shell =]──送出──→  [== 取留言資料 ==]──串流──→
                    ↓                              ↓
瀏覽器：            [渲染 Shell]    [載入 JS...]    [$RC 替換]   [Selective Hydrate]
          使用者看到頁面 ↑         使用者看到留言 ↑             可互動 ↑
```

### 伺服器端完整設定

```javascript
import { renderToPipeableStream } from 'react-dom/server';

app.get('*', (req, res) => {
  let didError = false;

  const { pipe, abort } = renderToPipeableStream(
    <App url={req.url} />,
    {
      bootstrapScripts: ['/client.js'],
      onShellReady() {
        // Shell 渲染完成，開始串流
        res.statusCode = didError ? 500 : 200;
        res.setHeader('Content-Type', 'text/html');
        pipe(res);
        // 連線保持開啟，Suspense resolve 時繼續送 chunk
      },
      onShellError(error) {
        // Shell 本身就失敗了——送一個最基本的錯誤頁面
        res.statusCode = 500;
        res.send('<h1>Something went wrong</h1>');
      },
      onError(error) {
        didError = true;
        console.error(error);
      },
    }
  );

  setTimeout(() => abort(), 10000); // 10 秒超時
});
```

注意 `onShellReady` 和 `onShellError` 的分離——Shell 失敗是災難性的（沒有頁面結構），但個別 Suspense 邊界失敗只會觸發該邊界的 `clientRenderBoundary`，讓 client 端的 Error Boundary 處理。這就是「per-boundary error recovery」。

---

## Selective Hydration：使用者點哪裡，就先 Hydrate 哪裡

傳統 hydration 是一個大鎖——整棵樹 hydrate 完才能互動。Selective Hydration 把每個 `<Suspense>` 邊界變成獨立的 hydration 單元，然後用一個事件驅動的優先排程來決定順序。

### 事件攔截機制

React 在 capture 階段攔截事件，區分兩種類型：

**離散事件**（click、keypress、input）：
- 在 capture phase 觸發同步的 selective hydration
- 如果該 Suspense 邊界的 code 已經載入，React **立即 hydrate 它**
- 事件接著正常傳播，就好像元件一直都是互動的
- 製造出「hydration 是即時的」的錯覺

**持續性事件**（mouseover、pointerover、focusin）：
- 排入佇列，hydration 完成後重播
- 只重播每種事件的最後一次（不會累積）

```
Selective Hydration 排程：

初始狀態：[Nav ✓] [Sidebar 待hydrate] [Content 待hydrate] [Comments 待hydrate]

使用者點擊 Content 區域：
→ React 在 capture phase 攔截 click
→ 立即 hydrate Content（提升優先級）
→ click 事件重播
→ Sidebar 和 Comments 降低優先級，idle 時再處理

結果：[Nav ✓] [Content ✓] [Sidebar 待] [Comments 待]
                ↑ 使用者感覺不到延遲
```

這個設計的關鍵洞察是：**使用者不會同時跟所有元件互動**。所以不需要全部 hydrate 完——只要保證使用者正在互動的那個元件是活的就夠了。

---

## React 19 的 Sibling Rendering 爭議

2024 年底，React 19 引入了一個看似合理但引發軒然大波的行為改變：**當一個 Suspense 邊界內的某個子元件暫停時，React 停止渲染它的兄弟元件，立刻顯示 fallback**。

React 團隊的邏輯是：既然要顯示 fallback，幹嘛還花時間渲染其他兄弟？不如早點讓使用者看到 loading 狀態。

問題是，這破壞了平行資料載入：

```jsx
// React 18：兩個 fetch 同時發出（平行）
// React 19：第二個要等第一個完成才會 render（瀑布！）
<Suspense fallback={<Spinner />}>
  <RepoData name="tanstack/query" />   {/* fetch 先發出 */}
  <RepoData name="tanstack/table" />   {/* 等上面的 resolve 才 render，才發 fetch */}
</Suspense>
```

TkDodo（TanStack Query 維護者）寫了一篇 "React 19 and Suspense — A Drama in 3 Acts" 詳細記錄這場風波。Poimandres 社群甚至考慮 fork React。React 團隊最終承認「misjudged how many people rely on this today」。

這場爭議揭示了 Suspense 設計中的一個根本張力：**「盡快顯示 fallback」和「盡可能平行取資料」是矛盾的**。你想要早點看到 loading 狀態，但也想要所有資料請求同時發出。這沒有完美解答——只有 trade-off。

### 避免瀑布的正確做法

```jsx
// ❌ 同一個 Suspense 下的兄弟——React 19 會造成瀑布
<Suspense fallback={<Spinner />}>
  <UserProfile />
  <UserPosts />
</Suspense>

// ✅ 分開的 Suspense 邊界——平行載入
<Suspense fallback={<ProfileSkeleton />}>
  <UserProfile />
</Suspense>
<Suspense fallback={<PostsSkeleton />}>
  <UserPosts />
</Suspense>

// ❌ 同一個元件內多次 use()——序列瀑布
function Dashboard() {
  const users = use(fetchUsers());   // 暫停
  const posts = use(fetchPosts());   // 等 users resolve 才跑到這行
  return <div>...</div>;
}

// ✅ 拆成獨立元件，各自有 Suspense 邊界
function Dashboard() {
  return (
    <>
      <Suspense fallback={<Skeleton />}>
        <UsersSection />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <PostsSection />
      </Suspense>
    </>
  );
}
```

---

## 在 Next.js App Router 中的實踐

Next.js App Router 是 Streaming SSR 最主流的消費者。它把 Suspense 嵌入了檔案系統路由的設計中：

```
app/
  layout.tsx          → Shell（永遠立即送出）
  loading.tsx         → 自動包成 <Suspense fallback={<Loading />}>
  page.tsx            → 包在 Suspense 裡面
  posts/
    loading.tsx       → 巢狀的 Suspense 邊界
    page.tsx
```

每個 `loading.tsx` 都是一個 Suspense 邊界。Server Components 使用 `async/await` 時自然就會暫停。這意味著你不用手動管理 Suspense——Next.js 的檔案結構就是你的 Suspense 架構。

但這也意味著你的**目錄結構決定了串流的粒度**。如果你把一個慢的資料請求放在 `layout.tsx` 裡，整個 layout 都會被阻塞——因為 layout 是 Shell 的一部分，不在任何 Suspense 邊界內。

---

## renderToString vs renderToPipeableStream：不只是 API 差異

| 面向 | `renderToString` | `renderToPipeableStream` |
|------|-------------------|--------------------------|
| 輸出 | 完整 HTML 字串 | Node.js Stream（分 chunk） |
| TTFB | 整頁渲染完才能送 | Shell 好了就送 |
| Event Loop | 阻塞 Node.js event loop | 非阻塞，chunks 之間會 yield |
| Suspense | 只渲染 fallback，不會串流 | 完整串流支援 |
| `React.lazy` | SSR 不支援 | SSR 支援 |
| 錯誤處理 | 全頁成功或全頁失敗 | Per-boundary 獨立處理 |
| Selective Hydration | 不支援 | 支援 |
| 記憶體 | 整個 HTML 存在記憶體 | 串流，記憶體用量低 |

有一個值得注意的 trade-off：有案例報告從 `renderToString` 切換到 `renderToPipeableStream` 後，伺服器吞吐量從 50 RPS 降到 15 RPS。串流有額外開銷——HTTP 連線保持開啟、chunk 管理、內聯腳本注入。如果你的頁面很簡單、資料取得很快，串流的好處可能不明顯。**串流的效益跟頁面複雜度和資料延遲成正比。**

---

## 你現在該怎麼做

1. **預設使用 `renderToPipeableStream`**（或讓 Next.js App Router 幫你處理）。`renderToString` 在架構上已經是死路。

2. **把 Suspense 邊界當成架構決策來設計**，不是隨手加的 loading 狀態。每個邊界定義了：串流的粒度、hydration 的單元、錯誤恢復的範圍。

3. **避免在同一個 Suspense 邊界內放多個獨立的資料請求**。分開成各自的 Suspense 邊界，確保平行載入。

4. **Shell 要輕**。Layout、navbar、sidebar 的靜態結構應該是 Shell 的一部分，能立即送出。任何需要 async 資料的東西都該用 Suspense 包起來。

5. **不要對所有東西都用 Suspense**。一個只需要 50ms 的 API 呼叫，加上 Suspense 反而會因為 fallback 的閃爍讓體驗更差。Suspense 解決的是「慢的、不確定的」非同步操作。

Suspense 從來就不只是一個 loading spinner 的包裝器。它是 React 對「漸進式渲染」這個問題的完整回答——從伺服器端的 HTML 串流，到客戶端的選擇性 hydration，再到並發模式下的優先排程。理解這整條鏈路，才算真正理解現代 React 的渲染架構。

---

## 延伸閱讀

- [New Suspense SSR Architecture in React 18 — Discussion #37](https://github.com/reactwg/react-18/discussions/37) — Dan Abramov 的完整架構說明，理解 Streaming SSR 的第一手資料
- [React `<Suspense>` 官方文件](https://react.dev/reference/react/Suspense) — 官方 API 參考和支援的資料源列表
- [React `renderToPipeableStream` 官方文件](https://react.dev/reference/react-dom/server/renderToPipeableStream) — Shell 概念和 callback 完整說明
- [React 19 and Suspense — A Drama in 3 Acts (TkDodo)](https://tkdodo.eu/blog/react-19-and-suspense-a-drama-in-3-acts) — React 19 sibling rendering 爭議的完整記錄
- [How Suspense Works Internally in Concurrent Mode (jser.dev)](https://jser.dev/react/2022/04/02/suspense-in-concurrent-mode-1-reconciling/) — 原始碼級別的 Suspense 實現分析
- [React Fizz Instruction Set 原始碼](https://github.com/facebook/react/blob/main/packages/react-dom-bindings/src/server/fizz-instruction-set/ReactDOMFizzInstructionSetShared.js) — `$RC`、`$RS` 等串流指令的實際實現
- [Selective Hydration — Discussion #130](https://github.com/reactwg/react-18/discussions/130) — 事件重播和優先排程的技術細節
