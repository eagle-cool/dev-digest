---
title: "Vercel 開大招：Workflows 出爐、Flags 正式 GA、AI SDK useChat 生產踩坑記"
date: 2026-04-17
description: "Vercel 在同一天把 durable execution 的新程式模型跟 Flags GA 一起推出，再加上一篇 AI SDK useChat 上線 30 天的踩坑報告。今天前端圈值得聊的，Vercel 一家就佔了一半。"
tags: [frontend, react, nextjs, ai, tooling, nodejs]
---

今天 Vercel 像過年一樣同時放兩個大的：一個是叫 Workflows 的 durable execution SDK，一個是 Flags 正式 GA。前者想把 BullMQ、Temporal、自架 queue worker 那套全吃掉，後者把 feature flag 收進平台層。再加一篇 useChat 上線 30 天的實戰檢討，前端跟後端之間那條線又被踩了一次。剩下就是一堆 React 效能、CSS 緩動曲線、跟 Next.js 15 Server Actions 到底什麼時候用的文章——看完至少不會在 PR 裡問笨問題。

---

## 🔧 今日硬菜

### [A new programming model for durable execution](https://vercel.com/blog/a-new-programming-model-for-durable-execution)

Vercel 把這東西取名叫 Workflows，野心很大：在 function 頂端寫 `"use workflow"`，在子函式寫 `"use step"`，剩下 retry、持久化、事件 log、串流斷線續傳全部平台幫你包。背後是三件事——event log 當單一事實來源、Fluid compute 執行單步、內建 Queue 自動排程。單 step 允許 50 MB、單 run 允許 2 GB，明顯是為 AI agent 傳圖傳影片鋪路。Beta 期間跑了一億次 run、1500+ 客戶，現在正式 GA（TypeScript stable、Python 還在 beta）。

**重點：**
- 不用再跟 BullMQ、Temporal、Inngest 那些 queue 架構分家，寫法接近 async/await，代價藏在平台黑盒裡
- 付費模式是 step 執行時才算錢，sleep/hook 掛著不計費——對 human-in-the-loop 流程很香
- SDK 開源，透過 "Worlds" adapter 可以跑在自架 Postgres 甚至 Cloudflare 上，不算完全鎖死

**但是……** 這是第二次 Vercel 把「平台原語直接進你 code」的賭局玩得這麼大（第一次是 Server Actions）。程式模型漂亮，但一旦你寫慣 `"use workflow"`，要遷回 Temporal 或自家 queue 就會很痛。要上之前先想清楚退路。

### [Vercel Flags is now generally available](https://vercel.com/changelog/vercel-flags-ga)

Feature flag 這場子本來 LaunchDarkly、GrowthBook、ConfigCat 已經很熱鬧，Vercel 這次 GA 拎著兩張牌進場：一是 dashboard 直接管 targeting rules 跟 segments，二是 Next.js 跟 SvelteKit 有 framework-native 的 Flags SDK。其他框架用 OpenFeature adapter 相容，算是給沒吃 Next 全餐的人留條路。重點是——這個東西跟 Workflows、AI Gateway 同一天公告，看得出來 Vercel 今年主軸就是「框架原生的 runtime 服務」。

**重點：**
- 跟 Next.js 深度整合，flag 可以直接影響 SSR/ISR 路徑，不用自己在客戶端判斷造成 layout shift
- OpenFeature 相容意味著你不一定要全家人搬來 Vercel 才能用
- 定價還是要看——有幾個客戶被 Vercel bill 嚇到的故事還沒冷

**但是……** Feature flag 最怕的是 vendor lock，以前 LaunchDarkly 也一樣痛。Vercel 走 OpenFeature 算是明智，但 targeting rules 的語意跨 provider 真的能無縫轉嗎？這種細節要等有人踩了才知道。

### [Vercel AI SDK useChat in Production: Lessons From 30 Days of Real Traffic](https://dev.to/whoffagents/vercel-ai-sdk-usechat-in-production-lessons-from-30-days-of-real-traffic-4gbo)

作者把 `useChat` 拿去跑 30 天真實流量，結論很前端工程師：這 hook 在 demo 漂亮，上線就一堆地雷。最大的坑是 message state 每次 re-render 都產生新物件，沒用 `useMemo` 的話子元件每 token 都會被炸一次 render——光這個 fix 就砍了 60% 的 render overhead。其他三顆地雷：行動網路斷線沒有內建 retry、沒設 `maxTokens` 會被單一惡意用戶把你 Anthropic 帳單打爆、session 預設 stateless 重整就沒了。

**重點：**
- `useChat` 的預設值是給 demo 用的，正式環境要自己加 memoize、retry、token 上限、rate limit、localStorage 持久化
- 串流 token 觸發父元件 re-render 這個坑在 React 19 也沒變——Vercel 的 hook 不會自動幫你隔離
- p99 延遲跟普通 API 不一樣，串流中斷點才是要監控的指標

**但是……** 這篇看起來比較像作者在推廣自己的 whoff-automation 專案，例子有點「太乾淨」。實務上 token 預算、retry backoff、session TTL 都還要配合後端（Anthropic rate limit、DB、redis）一起考慮，不是 frontend 加幾個 `useEffect` 就搞定。當 checklist 參考可以，別直接抄進 production。

---

## ⚡ 一句話帶過

- **[Next.js 15 Server Actions vs Route Handlers: When to Use Each](https://dev.to/whoffagents/nextjs-15-server-actions-vs-route-handlers-when-to-use-each-i-got-this-wrong-for-3-months-49hm)** — 作者三個月才搞懂：表單寫入用 Server Actions，需要 REST 語意、快取控制、第三方 webhook 就回去 Route Handlers。別把所有東西都塞進 action 裡。
- **[Claude Opus 4.7 on AI Gateway](https://vercel.com/changelog/opus-4.7-on-ai-gateway)** — Vercel AI Gateway 接上 Opus 4.7，長時間 agent 任務終於不用自己串 retry，跟上面那個 Workflows 是同一套劇本的兩塊拼圖。
- **[Vite doesn't transform accessor keywords](https://dev.to/joeedh/vite-doesnt-transform-accessor-keywords-3oa3)** — 用 TC39 `accessor` 關鍵字寫 decorator 又被 Vite 默默吃掉的考古報告，踩過的人直接把 `esbuild.target` 調到 `es2022` 以上吧。
- **[JavaScript Bloat en 2026: los 3 pilares que inflan tus dependencias npm](https://dev.to/lu1tr0n/javascript-bloat-en-2026-los-3-pilares-que-inflan-tus-dependencias-npm-1474)** — 西文好文：polyfill 重複、全量打包的 util 套件、過度樂觀的 peer deps 三位一體，把你的 node_modules 養到 700MB 從來不是意外。
- **[HTML to Image in Node.js — Without Puppeteer](https://dev.to/ozgurs/html-to-image-in-nodejs-without-puppeteer-5h5m)** — 300MB 的 Chromium 依賴終於有替代方案了，Serverless 冷啟動用 Puppeteer 的人可以認真評估。
- **[Level up CSS transitions with cubic-bezier](https://dev.to/railsdesigner/level-up-css-transitions-with-cubic-bezier-3955)** — 不要再 ease-in-out 了，調一下 cubic-bezier(0.4, 0, 0.2, 1) 你的 UI 馬上從無印良品變蘋果。
- **[Otimizando a performance do Schepta com React](https://dev.to/jreeeedd/otimizando-a-perfomance-do-schepta-com-react-2k7h)** — 葡文實戰：大表單每按一鍵整個樹 re-render 的經典病，作者用 uncontrolled inputs + 局部狀態根治，該知道的技巧但還是有人在踩。
- **[Modern Next.js Essentials: Building Scalable Full-Stack Applications](https://dev.to/preyumkr/modern-nextjs-essentials-building-scalable-full-stack-applications-2l0f)** — Next.js App Router 入門總結文，對還在 Pages Router 的人算是合適的地圖，資深的可以跳過。
- **[Open-source tab suspender after The Great Suspender got removed for malware](https://dev.to/ml3dev/i-built-an-open-source-tab-suspender-after-the-great-suspender-got-removed-for-malware-4gj2)** — 那場「熱門擴充被賣掉塞廣告」的經典案例終於有開源繼任者，Chrome 工程師常開 200 個 tab 的可以收藏。
- **[How I Handle Stripe Webhooks in Production (The Right Way)](https://dev.to/whoffagents/how-i-handle-stripe-webhooks-in-production-the-right-way-32jd)** — 冪等性、重試、交易隔離一次講完，Stripe webhook 被重發三次還扣兩次錢的都該來讀一下。
- **[How To Read Apple Mail Without AppleScript (It's 1000x Faster)](https://dev.to/whoffagents/how-to-read-apple-mail-without-applescript-its-1000x-faster-cji)** — 直接查 Apple Mail 的 SQLite 檔快上千倍，自動化工作流的人會爽死，但記得只讀不寫。

---

## 📚 慢慢啃

- **[Frontend Performance That Actually Moves the Needle](https://dev.to/codescoop/frontend-performance-that-actually-moves-the-needle-41b7)** — 讀完你會知道為什麼 Lighthouse 分數 95 不代表使用者不卡，RUM vs 實驗室數據怎麼取捨，大流量平台的優先順序怎麼排。
- **[The 'Privacy-First' Mirage: Why Your Analytics Hash is Still Fingerprinting](https://dev.to/zenovay/the-privacy-first-mirage-why-your-analytics-hash-is-still-fingerprinting-44ac)** — 所有把 IP hash 完就自稱 privacy-first 的產品都該讀，深入解釋 fingerprint 為什麼 SHA-256 也救不了，GDPR 稽核前必看。
- **[6 Accessibility Checks Most Scanners Miss](https://dev.to/chille87/6-accessibility-checks-most-scanners-miss-and-how-accessguard-catches-them-2gcf)** — axe、WAVE、Lighthouse 都抓不到的六個盲點，focus trap 跟 live region 那段尤其實用，a11y 做久了的人也會學到東西。
- **[Automate Code Reviews with Claude API and GitHub Actions in TypeScript](https://dev.to/whoffagents/automate-code-reviews-with-claude-api-and-github-actions-in-typescript-4j2e)** — 用 TypeScript + Claude API 寫一個 PR reviewer bot 的完整教學，程式碼都貼出來了，想把 AI 進 CI/CD 的週末抄一份來玩。
- **[queryd: driver-agnostic slow query detection for Node.js (optional Prisma)](https://dev.to/olegkoval/queryd-driver-agnostic-slow-query-detection-for-nodejs-optional-prisma-3l0m)** — 不挑 ORM、連 raw SQL 都能攔截的慢查詢偵測套件介紹，Prisma 用戶的 N+1 地獄救星，內部實作原理也值得讀。
