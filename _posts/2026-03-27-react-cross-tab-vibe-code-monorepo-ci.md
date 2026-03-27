---
title: "React 跨 Tab 同步、AI App 一成漏資料、Monorepo CI 砍半實戰"
date: 2026-03-27
description: "React 用 BroadcastChannel 跨分頁同步狀態、Vibe Coding 產出的 App 十分之一有資料外洩漏洞、22 個微服務的 Monorepo CI 從 10 分鐘砍到 5 分鐘的完整記錄"
tags: [frontend, react, nodejs, tooling, ai]
---

今天三道硬菜各有各的痛點。React 跨分頁狀態同步這件事，踩過的人都知道——用戶 A 分頁登出了，B 分頁還在那邊快樂購物。然後有人統計了 Vibe Coding 工具產出的 App，十分之一有資料外洩漏洞，這數字說實話不意外但看到還是會抖一下。最後一篇是 22 個微服務的 Monorepo CI 優化實戰，從工具鏈到測試到快取，每一步都有數據，這才叫工程。

---

## 🔧 今日硬菜

### [Real-time React: Syncing State Across Browser Tabs](https://dev.to/childrentime/real-time-react-syncing-state-across-browser-tabs-hn5)

每個寫過電商或 SaaS 的前端都碰過這個問題：用戶在 A 分頁登出，B 分頁完全不知道；改了深色模式，其他三個分頁還是亮的。瀏覽器的每個 Tab 各跑各的 JavaScript context，React state 天生不共享。這篇把 BroadcastChannel API 和 localStorage storage event 兩條路都走了一遍，從手刻 30 行的痛苦版本，到用 ReactUse 的 `useBroadcastChannel` + `useLocalStorage` 一行搞定。

最有價值的是四個實戰 pattern：auth 跨 Tab 登出、主題同步、購物車狀態、還有一個 leader election（只讓一個 Tab 跑 WebSocket 連線）。特別是 auth 那段，用 `useLocalStorage` 同步 token + `useBroadcastChannel` 發即時 LOGOUT 訊號的組合拳，新開的 Tab 讀到 null、已開的 Tab 即時跳轉，兩邊都顧到了。

**重點：**
- BroadcastChannel 走記憶體不碰 storage，快但不持久；localStorage 慢但新 Tab 打開就能讀到
- 兩個搭配用是正解：localStorage 管持久 + BroadcastChannel 管即時
- 但是... 這整篇基本上是 ReactUse 的推廣文。核心 API 是瀏覽器原生的，不想加依賴的話 30 行自己寫也不是不行

### [Your Vibe-Coded App Will Break in Production. Here's Why.](https://dev.to/geminate_solutions_9b6035/your-vibe-coded-app-will-break-in-production-heres-why-2ik9)

1,645 個 Lovable 產出的 App 裡有 170 個存在資料外洩漏洞——10.3%，每十個就有一個。這篇作者的公司過去半年「搶救」了 12 個 Vibe Coding App，問題永遠是那幾個：Supabase service_role key 直接塞在前端 bundle 裡（打開 DevTools 就能拿到完整 DB 權限）、server-side validation 完全沒做、JWT 沒設過期時間。

最扎心的是成本算數：Lovable 用戶平均花 400 credits 修一個 bug，但 AI 只修表面不修根因，修完一個又冒出另一個。算下來修 15-30 個生產 bug 要燒 $600-$3,000 的 credits，而請一個資深工程師 $40/hr 花 2-4 小時修一個，修完就是修完。作者給出了 7-12 天的 hardening checklist，從 auth、DB security、API hardening 到監控，這不是重寫，是補課。

**重點：**
- Supabase service_role key 暴露在前端 = 整個資料庫裸奔，RLS 直接被繞過
- AI 生成的 code 只做 client-side validation，server 端照單全收——負數金額？Infinity？字串？都行
- 但是... 這篇有明顯的業配味（作者是軟體外包公司 CEO）。數據和 checklist 是實在的，但「找我們修」的結論自己過濾一下

### [Journey of Systematically Cut Our Monorepo CI Time in Half](https://dev.to/jimmyyeung/journey-of-systematically-cut-our-monorepo-ci-time-in-half-ec8)

22 個 Django 微服務 + React 前端 + 共用 library 的 Monorepo，CI 跑一次 10-15 分鐘，工程師等到開始切 context switch。這篇的方法論教科書級：先寫 script 撈 GitHub Actions API 算 P50/P75/P90，找到瓶頸在最大服務的 unit test（P50 就要 10 分鐘），然後一層層剝。

工具鏈升級是第一刀：Yarn → Bun、Webpack → Rspack、ESLint 部分規則換 oxlint，每個替換省 30-40%。Docker build 最佳化（加 `.dockerignore`、清理假依賴）砍了 50%。然後是測試：有人在 production code 裡寫了 `time.sleep(10)` 而測試沒 mock 它——光這一個就白等 10 秒。最後是快取策略：GitHub Actions 只有 10 GB cache quota，branch-keyed cache 浪費空間，改成 lockfile hash 才是正解。

**重點：**
- 工具鏈現代化（Bun + Rspack + oxlint）是 ROI 最高的第一步，改動最小收益最大
- pytest-xdist `-n auto` 在單一 runner 上免費 2x 加速；matrix sharding 要花錢但值得
- 但是... 最大服務拆成 7 個 shard，billable minutes 其實增加了。wall clock time 和帳單是 trade-off，得看團隊規模算值不值

---

## ⚡ 一句話帶過

- **[Node.js Memory Leaks in Production: Finding and Fixing Them Fast](https://dev.to/axiom_agent_1dc642fa83651/nodejs-memory-leaks-in-production-finding-and-fixing-them-fast-2cna)** — Memory leak 不會自己報到，它就是那個讓你的服務每隔幾天就要重啟一次的沈默殺手
- **[Audit Your SvelteKit Codebase with a JSON Feed of 34 Svelte 5 Patterns](https://dev.to/rickcogley/audit-your-sveltekit-codebase-with-a-json-feed-of-34-svelte-5-patterns-4l6e)** — 從 React/Vue/Angular 遷移到 Svelte 5 的對照表，300 個 entry 可以餵給 AI 幫你批次轉換
- **[不懂模块化就别谈前端工程化](https://juejin.cn/post/7621443853846085674)** — 用 Next.js + NestJS + Tiptap 實作的協同文檔平台，從模組化角度講前端工程化，有實戰有深度
- **[We Rewrote JSONata with AI in a Day, Saved $500K/Year](https://www.reco.ai/blog/we-rewrote-jsonata-with-ai)** — 用 AI 一天重寫 JSONata 省了 50 萬美金年費，HN 上 58 則留言吵翻了，信不信自己看
- **[What's coming to our GitHub Actions 2026 security roadmap](https://github.blog/news-insights/product-news/whats-coming-to-our-github-actions-2026-security-roadmap/)** — 供應鏈攻擊打到 CI/CD 本身了，GitHub 的 2026 安全路線圖值得 DevOps 人看一眼
- **[GitHub Copilot Is Training on Your Private Code Now](https://dev.to/alanwest/github-copilot-is-training-on-your-private-code-now-you-probably-didnt-notice-2f6)** — 昨天聊過政策變更，這篇更深入分析了私有 repo 的灰色地帶和 credential 外洩風險
- **[I Built a Production Pay-Per-Lead Marketplace with Next.js 16 + Supabase + Stripe](https://dev.to/mkcvte/i-built-a-production-pay-per-lead-marketplace-with-nextjs-16-supabase-stripe-596j)** — Next.js 16 + Supabase + Stripe 的生產架構拆解，加拿大魁北克的在地化服務平台
- **[How I used Next.js and Claude 3.5 to stop my PM from nagging me about Jira](https://dev.to/phantasm0009/how-i-used-nextjs-and-claude-35-to-stop-my-pm-from-nagging-me-about-jira-1d81)** — 用 Claude 自動把 Slack 訊息同步到 Jira，PM 不再追殺你了（但 AI 會）
- **[I spent 1 month building my first NPM package from scratch](https://dev.to/victin16/i-spent-1-month-building-my-first-npm-package-from-scratch-and-here-is-the-result-3hk1)** — Web Doctor：一個分析 HTML/CSS/JS 的 CLI 審計工具，靈感來自 React Doctor 但不限框架
- **[GRAFSPEE Build faster with 60+ components](https://dev.to/nouryahyaoui/grafspee-build-faster-with-60-components-kn0)** — 60+ 個 React/Next.js + TypeScript 的 production-ready 元件，copy-paste 流派的福音

---

## 📚 慢慢啃

- **[Supercharge Your Web Apps: AI in the Background with Service Workers](https://dev.to/programmingcentral/supercharge-your-web-apps-ai-in-the-background-with-service-workers-502k)** — Service Worker 不只是離線快取，拿來跑 AI 推論讓主執行緒不卡頓，這個思路值得花 20 分鐘理解
- **[Interfacing Pure Functions with Our Impure World](https://dev.to/mahush/interfacing-pure-functions-with-our-impure-world-5e8f)** — Functional Core / Imperative Shell 架構講得很清楚，測試好寫、邏輯好讀，適合週末重新思考你的 code 結構
- **[AI-Generated Code Requires a Different Code Review Process](https://dev.to/kodustech/ai-generated-code-requires-a-different-code-review-process-1bea)** — AI 寫的 code 語法完美但語意可能全錯，你原本的 review 直覺會被騙，這篇講怎麼調整 review 心態和流程
