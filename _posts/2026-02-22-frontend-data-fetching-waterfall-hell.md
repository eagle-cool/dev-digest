---
title: "前端資料抓取的瀑布地獄：為什麼 2026 年了，從 REST API 組合資料還是這麼痛苦"
date: 2026-02-22
description: "從一位工程師受夠了 useQuery/Promise.all 義大利麵說起，深入剖析前端 data fetching waterfall 問題的根源、DataLoader 的 batching 哲學、BFF 與 GraphQL 的權衡，以及 React Server Components 是否真的終結了這場戰爭。"
tags: [deep-dive, frontend, programming]
---

最近在 DEV Community 上看到一篇很有共鳴的文章：一位 React 工程師受夠了在微服務架構下抓資料的痛苦，寫了一個叫 `@nimir/references` 的 library 來解決 nested reference resolution 的問題。他的後端是一堆「哥布林微服務」（他自己的用詞），每個 API 只回傳 ID，前端要自己串接所有關聯——先抓 ticket，再用 assigneeId 抓 user，再用 teamId 抓 team，再用 leadUserId 抓 team lead。每多一層就是一次 round trip，每多一個欄位就是十行 boilerplate。

聽起來很熟悉，對吧？因為這是每一個在微服務架構下寫前端的工程師都踩過的坑。而且不只是 2026 年——這個問題從 REST API 被發明的那天就存在了。真正值得討論的不是某個 library 怎麼用，而是：**為什麼這麼多年過去了，前端的 data fetching 還是一團亂？**

---

## 瀑布問題：比你想的還要昂貴

先定義一下「瀑布」到底是什麼。當你的元件 A 抓完資料後才 render 元件 B，元件 B 抓完資料後才 render 元件 C——恭喜你，你剛剛創造了一個 request waterfall。如果每個 fetch 花 1 秒，三層就是 3 秒。但如果它們可以平行執行，理論上只需要 1 秒。

根據 Sentry 的分析，這是 React 應用中最常見也最昂貴的效能問題之一。罪魁禍首就是那個無處不在的 pattern：

```javascript
useEffect(() => {
  fetchData().then(setData);
}, []);
```

每個元件各自在 `useEffect` 裡 fetch 資料，先顯示 Loading，再 render 子元件——子元件又重複同樣的流程。這不是 bug，這是 React 的元件模型和 data fetching 之間的根本衝突。元件是樹狀的，fetch 邏輯被散落在樹的各個節點上，執行順序天然就是瀑布式的。

但原始文章裡的問題比單純的 waterfall 更惡劣。那不只是「元件各自 fetch」，而是**同一個頁面要 resolve 一整棵 reference tree**：

```javascript
const ticket = await fetchTicket('t-1');
const assignee = ticket.assigneeId ? await fetchUser(ticket.assigneeId) : null;
const watchers = await Promise.all(
  (ticket.watcherIds ?? []).map(id => fetchUser(id))
);
const team = assignee?.teamId ? await fetchTeam(assignee.teamId) : null;
const lead = team?.leadUserId ? await fetchUser(team.leadUserId) : null;
```

這是兩層。實際的產品頁面有幾十個 reference field。更要命的是，同一個 user ID 可能在 assignee、watchers、team lead 裡出現三次——代表三次重複的 HTTP request。沒有 batching，沒有 deduplication，純粹的浪費。

TanStack Query（前身 React Query）的文件也直說了：「The biggest performance footgun when using any data fetching library that lets you fetch data inside of components is request waterfalls.」這不是某個框架的問題，這是整個 fetch-in-component pattern 的結構性缺陷。

---

## DataLoader 的啟示：Facebook 在 2010 年就解過的題

有趣的是，這個問題 Facebook 在 2010 年就遇過了，而且他們的解法後來成為 GraphQL 的基石之一。

DataLoader 是 Facebook 開發的一個通用 batching 和 caching 工具。它的核心概念簡單到優雅：在同一個 event loop tick 中，把所有 `.load(key)` 的呼叫收集起來，合併成一個 `batchLoadFn(keys)` 呼叫。

```javascript
// 這三個呼叫發生在同一個 tick
userLoader.load('user-1');
userLoader.load('user-2');
userLoader.load('user-1'); // 重複的 key

// DataLoader 合併成一個呼叫：
// batchLoadFn(['user-1', 'user-2'])
// user-1 只 fetch 一次，結果被 cache 並回傳給兩個 caller
```

沒有 DataLoader 的 GraphQL server，一個查詢可能產生 13 次 database request。有了 DataLoader，最多 4 次。這個差距在高流量的場景下是生死之別。

原文作者的 `@nimir/references` 做的事情，本質上就是把 DataLoader 的哲學搬到前端。定義 source（怎麼批次 fetch），宣告哪些 field 是 reference，然後讓 library 處理 batching、deduplication、和 nested traversal。概念上沒有新東西，但放在前端的 context 裡，卻填補了一個很少人正式解決的空缺。

因為 GraphQL 解決這個問題的方式是在 server side 做 resolution——client 只發一個 query，server 的 resolver 用 DataLoader 處理 N+1。但如果你沒有 GraphQL 呢？如果你面對的是一堆 legacy REST API，每個都是不同 team 維護的微服務，schema 各自為政？這時候 resolution 的責任就落在前端身上，而前端缺少一個 DataLoader 等級的抽象。

---

## 那為什麼不用 GraphQL？

每次有人抱怨 REST 的 data fetching 問題，總會有人跳出來說：「用 GraphQL 啊。」這句話技術上沒錯，但實務上忽略了幾個殘酷的現實。

**第一，你需要有權力改變後端架構。** 如果你是大型企業裡的前端 team，後端是十幾個 team 各自維護的微服務，要推動一個 GraphQL gateway 不是技術問題——是政治問題。你需要說服每個 team 暴露 GraphQL resolver，需要一個中央 team 維護 schema stitching 或 federation，需要解決 N+1 問題（又回到 DataLoader），還需要處理 authorization、caching、和 rate limiting 在 GraphQL 層的實作。

**第二，GraphQL 的學習曲線和維護成本是真的。** Fragment colocation（Relay 的核心設計哲學）是個好主意：每個元件宣告自己需要的 data，compiler 把 fragment 拼成完整的 query。但 Relay 的設定複雜度是出了名的。Apollo Client 比較平易近人，但你還是得維護 schema、寫 resolver、處理 code generation、管理 cache normalization。對於一個已經跑在 REST 上的系統，遷移成本不可忽視。

**第三，GraphQL 不是萬靈丹。** Server-side waterfall 照樣可以在 GraphQL resolver 裡發生。如果你的 resolver 沒有用 DataLoader 做 batching，或是 resolver 之間有依賴關係，你只是把 client-side waterfall 搬到了 server side。Sentry 的分析文章精準地指出：「waterfalls are concurrency problems, not rendering problems——moving code to the server without parallelizing requests merely shifts the issue upstream.」

所以現實是：很多 team 用 GraphQL 的條件不成熟。他們需要一個在 REST API 上也能用的解法。

---

## BFF：前端的救贖，還是新的負擔？

另一個常被提出的方案是 BFF（Backend for Frontend）——在前端和微服務之間加一個專門為前端服務的 backend 層。它接收前端的一個 request，在 server side 幫你打完所有微服務的 API，組合好資料再回傳。

Marmelab 在 2025 年的一篇分析裡很實在地列出了 BFF 的好處：減少 network round trip、前端 team 可以獨立開發不用等後端改 API、集中化 client-specific 邏輯。有些組織導入 BFF 後「feature delivery speed 提高 2-3 倍」、「app load time 減少 30-60%」。這些數字不是騙人的。

但 BFF 也有它的代價。你現在多了一個服務要維護、要部署、要監控。如果你有 web、iOS、Android 三個前端，理論上你需要三個 BFF。更危險的是，如果 BFF 開始堆積 business logic，它就變成一個 mini-monolith——你從微服務架構出發，結果在中間層重新蓋了一個 monolith。

BFF 的核心準則是：**它必須是一個薄的翻譯層，只負責 data aggregation 和 transformation，絕不包含 business logic。** 一旦違反這個準則，你就是在挖一個比原始問題更大的坑。

Azure 和 AWS 的官方架構指南都推薦 BFF pattern，但同時也都說：如果你只有一個前端、或是你的 API 已經支援 field selection，BFF 是 overkill。

---

## React Server Components：這次真的不一樣？

2026 年的前端圈最大的敘事之一就是 React Server Components（RSC）。根據 State of JS 2025 調查，80% 的新 React 應用使用 RSC。Next.js App Router 把 RSC 設為預設。這不再是實驗性功能——它已經是主流。

RSC 對 data fetching 問題的解法很直觀：**把 fetch 搬到 server side，在 component 裡直接 `await`。** 因為 server component 在伺服器上執行，它可以直接存取資料庫、呼叫內部 API，不經過公開的 HTTP endpoint。更重要的是，server 到 microservice 的 latency 通常只有毫秒級，不像 client 到 server 可能需要幾百毫秒。

```jsx
// Server Component — 直接在 server 上 fetch
async function TicketPage({ id }) {
  const ticket = await fetchTicket(id);
  const assignee = await fetchUser(ticket.assigneeId);
  const team = await fetchTeam(assignee.teamId);

  return <TicketDetail ticket={ticket} assignee={assignee} team={team} />;
}
```

看起來很美對吧？但魔鬼在細節裡。

**問題一：Server-side waterfall。** 上面那段 code 裡的三個 `await` 是依序執行的——你只是把 client-side waterfall 搬到了 server。解法是用 `Promise.all`：

```jsx
async function TicketPage({ id }) {
  const ticket = await fetchTicket(id);
  const [assignee, watchers] = await Promise.all([
    fetchUser(ticket.assigneeId),
    Promise.all(ticket.watcherIds.map(fetchUser)),
  ]);
  // ...
}
```

但如果 watchers 的 team 也要 fetch，team lead 也要 fetch，你又回到了同樣的 nested resolution 問題。`Promise.all` 只能平行化同一層的 request，跨層的依賴還是瀑布。

**問題二：Streaming 的錯覺。** RSC 搭配 Suspense 可以做 progressive streaming——先送 HTML 的 shell，資料到了再填入。Web Performance Calendar 2025 的研究指出，沒有 streaming 的 RSC 頁面 TTFB 在 350-550ms，啟用 streaming 後可以降到 40-90ms。這確實是巨大的改進。但 streaming 解決的是 perceived performance（使用者感知的速度），不是 total data fetching time（實際的資料載入總時間）。使用者更快看到畫面，但完整頁面的 paint 時間可能差不多。

**問題三：不是所有情境都適合。** RSC 適合 read-heavy 的場景——部落格、電商產品頁、dashboard 的首次載入。但對於高互動性的 CRUD 應用——使用者在前端做大量操作、資料頻繁更新——你還是需要 client-side 的 state management 和 data fetching。TanStack Query 在這個場景裡仍然不可取代，因為它的 cache invalidation、optimistic update、和 mutation handling 是 RSC 沒有涵蓋的領域。

所以 RSC 不是銀彈。它解決了 initial load 的 waterfall 問題（前提是你正確地平行化了 server-side fetch），但 client-side 的 reference resolution 問題在互動式應用裡依然存在。

---

## Route-Level Fetching：被低估的解法

在所有方案裡，有一個被嚴重低估的 pattern：**route-level data fetching**。

核心思想很簡單：不要在元件裡 fetch 資料，在路由層級就把所有需要的資料 fetch 好。React Router、Remix、TanStack Router 都支援這個 pattern。Remix 最早把這個概念帶入主流——每個 route module 有一個 `loader` function，在 navigation 發生時、元件 render 之前就執行。

```javascript
// Remix-style route loader
export async function loader({ params }) {
  const ticket = await fetchTicket(params.id);
  const [assignee, watchers, team] = await Promise.all([
    fetchUser(ticket.assigneeId),
    Promise.all(ticket.watcherIds.map(fetchUser)),
    ticket.teamId ? fetchTeam(ticket.teamId) : null,
  ]);
  return { ticket, assignee, watchers, team };
}
```

這個 pattern 的好處是：**所有 data requirement 集中在一個地方**，你可以清楚地看到（和平行化）所有的 fetch。沒有 component tree 深處藏著的 `useEffect` 驚喜。TanStack Query 也可以搭配 router 做 prefetching——在使用者 hover 連結時就開始 fetch，到了頁面時資料已經在 cache 裡了。

但代價是：你犧牲了元件的 data colocation。Relay 的 fragment colocation 之所以被推崇，是因為元件和它需要的資料宣告在一起——改元件時你不需要去別的檔案改 data requirement。Route-level fetching 把所有 data loading 集中到 route 層，元件變成純粹的 presentational component。這對大型應用來說可能是好事（關注點分離），也可能是壞事（改一個元件的資料需求要跨檔案修改）。

---

## 所以，2026 年的解法是什麼？

沒有單一解法。但有一個越來越清晰的 decision tree：

**如果你有能力改後端架構：** GraphQL + DataLoader 仍然是最完整的解法。讓 server side 處理 reference resolution，client 只要宣告需要什麼資料。Relay 的 compiler 加上 fragment colocation，在大型應用裡的維護性是最好的。

**如果你無法改後端（legacy REST, 多團隊微服務）：** 考慮 BFF。用 Node.js 或 Go 在 server side 做 data aggregation，前端只打一個 endpoint。但記住保持 BFF 的薄度——只做 aggregation，不做 business logic。

**如果你用 Next.js / App Router：** RSC 是你的朋友，但要注意平行化 server-side fetch。用 `Promise.all` 打平同層 request，用 Suspense boundary 做 progressive streaming。Client-side 的互動式資料用 TanStack Query 管理。

**如果以上都不適用：** 原文作者的 batching + deduplication pattern 是正確的方向。不管你用什麼 library（`@nimir/references`、或自己寫一個 DataLoader-like 的 client-side batcher），核心觀念就是：

1. **Batch：** 把同一個 tick 裡的多個 fetch 合併成一個 batch request
2. **Deduplicate：** 同一個 ID 只 fetch 一次
3. **Cache：** 在 request lifecycle 內 cache 結果，避免重複 fetch

```javascript
// 最小化的 client-side batcher
class Batcher {
  constructor(batchFn) {
    this.batchFn = batchFn;
    this.queue = new Map();
    this.scheduled = false;
  }

  load(id) {
    if (this.queue.has(id)) return this.queue.get(id).promise;

    let resolve;
    const promise = new Promise(r => (resolve = r));
    this.queue.set(id, { resolve, promise });

    if (!this.scheduled) {
      this.scheduled = true;
      queueMicrotask(() => this.flush());
    }
    return promise;
  }

  async flush() {
    const entries = [...this.queue.entries()];
    this.queue.clear();
    this.scheduled = false;

    const ids = entries.map(([id]) => id);
    const results = await this.batchFn(ids);

    entries.forEach(([id, { resolve }], i) => resolve(results[i]));
  }
}
```

這 30 行 code 就是 DataLoader 的核心邏輯。不需要 GraphQL，不需要 BFF，不需要遷移到 RSC。在任何 REST API 上都能用。

---

## 我的判斷

前端 data fetching 的混亂根源不在前端——**在於微服務架構把 data aggregation 的責任推給了最不適合做這件事的一方**。Server 之間的 latency 是微秒級，client 到 server 是毫秒級甚至秒級。在 client side 做 reference resolution 天生就是事倍功半。

但現實是：大多數前端工程師沒有權力改變後端架構。你拿到的就是一堆 REST endpoint，每個回傳 ID，你得自己想辦法。在這個限制下，正確的做法是：

1. **短期：** 在前端實作 client-side batching 和 deduplication。不管用 library 還是自己寫，這是 ROI 最高的改善。
2. **中期：** 推動 route-level data fetching。把 data loading 從元件裡拉出來，集中在路由層級，方便平行化和 prefetching。
3. **長期：** 倡議 BFF 或 GraphQL gateway。這需要組織層面的支持，但效益是根本性的。

2026 年了，我們還是在 client side 手動 resolve reference、手動 deduplicate request、手動處理 loading state。這不是工具不夠好的問題——是架構決策把錯誤的複雜度推到了錯誤的地方。每個在 `useEffect` 裡寫 fetch chain 的前端工程師，都在為後端的 API 設計決策買單。

好消息是，解法已經在那裡了。DataLoader 的 batching pattern 是 2010 年的智慧，route-level fetching 被 Remix 帶入主流，RSC 讓 server-side composition 變得可行。工具不缺，缺的是做出改變的決心——和說服後端 team 的政治手腕。

---

## 延伸閱讀

- [I Got Tired of useQuery/Promise.all Spaghetti So I Built This](https://dev.to/mimikkk/i-got-tired-of-usequerypromiseall-spaghetti-so-i-built-this-2n73) — 本文的起點，一位 React 工程師的實戰解法
- [Fetch Waterfall in React — Sentry](https://blog.sentry.io/fetch-waterfall-in-react/) — 對 React fetch waterfall 問題最清楚的技術分析
- [DataLoader — GitHub](https://github.com/graphql/dataloader) — Facebook 在 2010 年開發的 batching 工具，GraphQL 的基石
- [Performance & Request Waterfalls — TanStack Query](https://tanstack.com/query/latest/docs/framework/react/guides/request-waterfalls) — TanStack Query 官方的 waterfall 防治指南
- [Do You Need a Backend for Frontend? — Marmelab](https://marmelab.com/blog/2025/10/01/do-you-need-a-backend-for-frontend.html) — 2025 年最務實的 BFF 利弊分析
- [React Server Components — React 官方文件](https://react.dev/reference/rsc/server-components) — RSC 的權威參考
