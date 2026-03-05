---
title: "Next.js 效能調校實戰、AI 爬蟲看不見你的 React App、Node.js 與 FastAPI 的事件迴圈對決"
date: 2026-03-05
description: "Next.js App Router 拿下 Lighthouse 96+ 的實戰拆解、AI 爬蟲為何看不到你的 React 內容、Node.js 和 FastAPI 的非同步架構深度比較。前端工程師今天值得讀的都在這。"
tags: [frontend, react, nextjs, nodejs, ai, web-platform]
---

今天值得聊的就三件事：有人用 Next.js App Router 在 mobile 拿到 Lighthouse 96+（不是 landing page，是有動態內容的正式站），AI 爬蟲看不懂你的 React app 這件事終於有人寫清楚了，然後 Node.js 跟 FastAPI 的事件迴圈比較又冒出一篇——這次倒是把 libuv 和 uvloop 的關係講得挺到位。

---

## 🔧 今日硬菜

### [Optimizing Next.js Performance: A Practical Case Study (96+ Lighthouse Mobile)](https://dev.to/trlz/optimizing-nextjs-performance-a-practical-case-study-96-lighthouse-mobile-31da)

這篇不是那種「用 Next.js 建了一個 Hello World 然後炫耀 100 分」的文章。作者拿的是一個有 hero image、動態內容、analytics scripts、client components 的正式產品站，在 mobile 上跑出 96+ 的 Lighthouse 分數。技術含量在於他把每一個決策都攤開來講：`font-display: swap` 防 FOIT、全域 CSS 只留 reset 和 variables、LCP image 用 WebP + `priority` + `fetchPriority="high"`、甚至在 build time 把外部 S3 圖片先下載到 `/public` 來消除 DNS lookup 延遲。最實用的一點是他對 `"use client"` 的態度——hero section 和 above-the-fold 全部走 Server Component，互動邏輯往 below-the-fold 推。這才是 App Router 該有的用法。

**重點：**
- LCP image 優化三件套：WebP 格式 + `priority` 屬性 + 預設容器尺寸消除 CLS
- Build time 預下載外部圖片，避免 runtime 的 DNS/TLS 延遲影響 TTFB
- 但是... 這套做法高度依賴靜態內容場景，如果你的頁面是高度動態的（像電商即時庫存），build time 下載圖片這招就不適用了

### [Why AI Crawlers Can't See Your React App](https://dev.to/jsvisible/why-ai-crawlers-cant-see-your-react-app-137n)

踩過 SEO 坑的人都知道 Google 有 rendering queue，CSR 頁面雖然慢但最終會被索引。但 AI 爬蟲完全是另一回事——GPTBot、PerplexityBot、ClaudeBot 根本不會執行 JavaScript。它們就是發一個 HTTP request，拿到什麼 HTML 就看什麼，沒有第二次機會。所以你的 Create React App 如果 server 回傳的是一個空 `<div id="root">`，在 AI 搜尋引擎眼中你就是不存在。最要命的是 hybrid rendering 的陷阱：你的 Next.js 首頁可能有 SSR，但 blog 和產品頁是 CSR，結果就是部分頁面在 AI 搜尋中完全隱形，而且你根本不知道。

**重點：**
- AI 爬蟲不渲染 JS：GPTBot、PerplexityBot 只看 server 回傳的原始 HTML
- 最簡單的檢查法：Chrome 右鍵 → View Page Source，看到的就是 AI 爬蟲看到的
- 但是... 文章推的 SSR 方案對效能和成本的影響隻字未提。不是每個團隊都扛得住 SSR 的 server cost

### [Node.js vs FastAPI Event Loop: A Deep Dive into Async Concurrency](https://dev.to/dipcb05/nodejs-vs-fastapi-event-loop-a-deep-dive-into-async-concurrency-35b1)

又一篇 Node vs Python 的文章，但這次有個有趣的切入點：FastAPI 常用的 uvloop 底層其實就是 libuv——沒錯，就是 Node.js 用的那個。所以說到底，「FastAPI + uvloop = Python + libuv」，不同語言，同一個非同步引擎。文章把兩邊的 concurrency model 拆得很清楚：Node 是單線程事件迴圈 + libuv thread pool，CPU 密集任務會卡死整個 server；FastAPI 用 Gunicorn 開多個 worker process，每個 process 各跑一個事件迴圈，天然支援多核。不過文章有個常見的誤解沒提到——Node 的 `worker_threads` 早就不是實驗性功能了，現在要做 CPU 密集計算也不是沒辦法。

**重點：**
- uvloop（FastAPI 常用）底層就是 libuv（Node.js 用的），同源不同命
- FastAPI 多 worker process 天然吃多核，Node 需要 cluster mode 才能做到
- 但是... 文章忽略了 Node.js `worker_threads` 的成熟度，還有 Bun/Deno 在這方面的進展

---

## ⚡ 一句話帶過

- **[React: Singletons aren't as evil as you think](https://dev.to/link2twenty/react-singletons-arent-as-evil-as-you-think-44m8)** — 在 React 裡幫 singleton 平反，用 module scope 管理跨元件狀態其實沒那麼邪惡，只要你知道自己在幹嘛
- **[告别暴力轮询：深度解锁浏览器"观察者家族"](https://juejin.cn/post/7613282719573213234)** — IntersectionObserver、MutationObserver、ResizeObserver、PerformanceObserver 一次講完，想戒掉 `scroll` 監聽的快去看
- **[How I Made Missing Translations a Compile-Time TypeScript Error](https://dev.to/akocan98/how-i-made-missing-translations-a-compile-time-typescript-error-5bjc)** — 用 TypeScript 型別系統在編譯期抓漏翻譯，不用等 runtime 才發現 UI 壞掉，react-scoped-i18n 的思路值得一看
- **[That inflight memory leak warning in your npm install? Here's the fix](https://dev.to/firekid846/that-inflight-memory-leak-warning-in-your-npm-install-heres-the-fix-4b00)** — 每次 `npm install` 都跳的那個 inflight deprecation 警告，原因和修法都在這，三分鐘讀完
- **[Stop Mixing UI Components With Business Logic](https://dev.to/andreluizlunelli/stop-mixing-ui-components-with-business-logic-9fj)** — 把 Vue 元件拆成 `app/`（業務邏輯）和 `ui/`（純視覺），道理老生常談但分類方式乾淨俐落
- **[AI Makes Experienced Developers Slower. Here's Why.](https://dev.to/choutos/ai-makes-experienced-developers-slower-heres-why-1ico)** — METR 的隨機對照實驗顯示 AI coding tools 讓資深開發者變慢，不是標題黨，是有數據的
- **[Atomic Design: A More Precise Compass for Atoms and Molecules](https://dev.to/massdev_9707b3b2b60cb8fac/atomic-design-a-more-precise-compass-for-atoms-and-molecules-gj6)** — Brad Frost 的 Atomic Design 邊界太模糊？這篇提出更精確的 atom/molecule 判斷標準
- **[We Built a 3KB WebMCP Polyfill (97x Smaller Than @mcp-b/global)](https://dev.to/up2itnow0822/we-built-a-3kb-webmcp-polyfill-97x-smaller-than-mcp-bglobal-3kg4)** — Chrome 146 原生支援的 `navigator.modelContext`，有人做了 3KB 的 polyfill，比官方的 285KB 小 97 倍
- **[How I Built Zero-Knowledge File Sharing Using the Web Crypto API](https://dev.to/graysoftdev/how-i-built-zero-knowledge-file-sharing-using-the-web-crypto-api-no-server-ever-sees-your-data-28cp)** — 用瀏覽器原生 Web Crypto API 做端到端加密檔案分享，金鑰永遠不離開瀏覽器
- **[Why your design tokens and your CSS are probably out of sync](https://dev.to/zetareticoli/why-your-design-tokens-and-your-css-are-probably-out-of-sync-and-how-to-check-3od6)** — 半年前定義的 design tokens，現在 CSS 裡到處都是 hardcoded 的 `#1A1A1A`，你懂的

---

## 📚 慢慢啃

- **[性能级目录同步：IntersectionObserver 实战](https://juejin.cn/post/7613303121368514598)** — 用 IntersectionObserver 實作滾動高亮目錄同步，從 `rootMargin` 的精確設定到效能對比，比起暴力 `getBoundingClientRect` 優雅太多
- **[Reactive Data Without the Async Headaches](https://dev.to/robert_sanders_04918a4344/reactive-data-without-the-async-headaches-5fbd)** — 試圖解決前端 async 地獄：不用 dependency array、不用 memoization、不用 subscription，直接描述資料讓系統自己處理
- **[The AI Code Review Bottleneck: When Generation Outpaces Human Judgment](https://dev.to/lizechengnet/the-ai-code-review-bottleneck-when-generation-outpaces-human-judgment-7k1)** — 20+ AI coding tools 的 system prompts 被公開後的反思：AI 生成程式碼的速度遠超人類 review 的能力，這個瓶頸怎麼解？
- **[Accessibility Issues Are Often Usability Issues](https://protovate.com/blog/you-dont-need-accessibility-until-you-do/)** — 無障礙設計其實就是可用性設計，這篇從實務角度講為什麼 a11y 不是「額外工作」而是「本來就該做的事」
