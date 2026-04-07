---
title: "Vibe Coding 是邪教？JS 記憶體洩漏抓鬼、PgBouncer 救你的 PostgreSQL"
date: 2026-04-07
description: "Bram Cohen 怒嗆 vibe coding 是 dogfooding 走火入魔，JavaScript 記憶體洩漏完整偵查手冊，Node.js 連線池正在默默殺死你的 PostgreSQL——今天三道硬菜都是實戰踩坑等級。"
tags: [frontend, nodejs, typescript, ai, web-platform]
---

今天值得聊的就三件事。BitTorrent 之父 Bram Cohen 寫了篇文章把 vibe coding 文化噴了個體無完膚（HN 上 445 分，大家都在搬板凳看戲），然後有人用 3 AM 的 React dashboard 當機寫了一篇 JS 記憶體洩漏偵探小說，最後是一個 Node.js 開發者發現自己的 PostgreSQL 被 280 條閒置連線活活吃掉記憶體的故事。三篇都是「踩過坑才寫得出來」的等級。

---

## 🔧 今日硬菜

### [The cult of vibe coding is dogfooding run amok](https://bramcohen.com/p/the-cult-of-vibe-coding-is-insane)

Bram Cohen——對，就是寫 BitTorrent 那位——在 Claude Code source code 洩漏事件後寫了這篇。他的觀點很直白：vibe coding 本身不是問題，dogfooding 走火入魔才是。

他指出 Claude Code 的程式碼裡有大量重複的 agents 和 tools，而這種混亂的根源是開發團隊「堅持不看 source code」。因為 vibe coding 的文化就是：看底層等於作弊。但問題是，這些程式碼是用英文寫的，任何人都看得懂，隨便掃一眼就能發現「欸，這兩個東西不是一樣的嗎？」

Cohen 的核心論點是：AI 其實很擅長清理技術債，但前提是你得告訴它哪裡有亂。你不需要自己寫 code，但你得花幾分鐘看一下、指出問題，然後用 Ask mode 跟 AI 討論邊界情況。「純 vibe coding 是個迷思——你還是在建 plan files、skills、rules 這些框架，AI 沒有這些東西根本跑不動。」

**重點：**
- Vibe coding ≠ 完全不看 code，你還是在用人類語言建構框架
- AI 很擅長清理 spaghetti code，但需要人類指出問題方向
- 但是... 如果連 Anthropic 自己都不看自家產品的 source code，那「AI 寫的 code 品質好不好」這問題的答案可能比我們想的更悲觀

### [JavaScript Memory Leaks: How to Find, Fix, and Prevent Them](https://dev.to/alex_aslam/javascript-memory-leaks-how-to-find-fix-and-prevent-them-2e3a)

凌晨三點，一個 React dashboard 吃了 1.2 GB RAM，而它最多只顯示一千行資料。這位作者用偵探小說的手法寫了一篇 JS 記憶體洩漏完整指南，而且每個案例都是真實踩坑。

他把 GC 比喻成美術館的夜間清潔工——它只清理沒人看的畫，但如果你一直指著一幅你根本不在乎的畫，清潔工就只能聳聳肩走掉。文章逐一拆解四大嫌疑犯：意外全域變數、吸血鬼 closure（捕獲了整個 Redux store 但只用了一個 flag）、忘記清理的 `setInterval`/`addEventListener`（SPA modal 開了就沒關的 timer），以及無限增長的 cache 物件。

最實用的部分是 Chrome DevTools 的 Memory 偵查 workflow：heap snapshot 比對、allocation timeline、Detached DOM nodes 篩選——這套組合拳足以定位 99% 的記憶體洩漏。

**重點：**
- Closure 會捕獲整個 scope 的變數，即使你沒用到——GC 無法分辨你用不用
- React 裡每個 `useEffect` 都必須 return cleanup，這不是建議，是生存法則
- 但是... 文章沒提到 `FinalizationRegistry` 和最新的 WeakRef 實戰用法，這塊在 2026 年應該要補上了

### [Your Node.js App Is Probably Killing Your PostgreSQL (Connection Pooling Explained)](https://dev.to/polliog/your-nodejs-app-is-probably-killing-your-postgresql-connection-pooling-explained-1db2)

一台 4GB RAM 的 server，PostgreSQL 跑在 94% 記憶體。罪魁禍首不是 query 慢，不是資料量大，而是 280 條閒置連線。每條連線大約吃 5-10MB（PostgreSQL 會為每條連線 spawn 一個獨立 process），280 × 7MB ≈ 2GB，直接把記憶體吃光。

Node.js 的架構天生容易 over-connect：3 個 web server replicas × 10 連線 + 3 個 worker × 10 連線 + job scheduler，光 idle 就 75 條。流量一來，直接破 150。而 PostgreSQL 預設 `max_connections` 才 100。

解法是 PgBouncer，用 transaction pooling 模式。文章給了完整的 Docker Compose 設定和前後對比數據：PostgreSQL 連線從 idle 75 降到 8、peak 210 降到 25、RAM 從 1.47GB 降到 175MB、p99 latency 從 340ms 降到 95ms。但 transaction pooling 會踩到幾個坑：named prepared statements 會壞、`SET` session 變數會消失、`pg_advisory_lock` 要改用 `pg_advisory_xact_lock`、`LISTEN/NOTIFY` 要用獨立連線繞過 PgBouncer。

**重點：**
- Prisma 搭配 PgBouncer 必須加 `?pgbouncer=true` 參數，否則 prepared statement 會莫名其妙爛掉
- 降低 `max_connections` 反而能提升效能，因為多出的 RAM 可以給 `shared_buffers` 和 `work_mem`
- 但是... 文章沒提到 Supabase 已經換成 Supavisor（不再是 PgBouncer），行為有些微差異，生產環境別照抄

---

## ⚡ 一句話帶過

- **[EmDash: A Full-Stack TypeScript CMS Built on Astro + Cloudflare](https://dev.to/recca0120/emdash-a-full-stack-typescript-cms-built-on-astro-cloudflare-can-it-replace-wordpress-5584)** — 用 Astro + Cloudflare 打造的全 TypeScript CMS，想取代 WordPress 的又一位勇者，但 content 存 JSON 不存 HTML 這個設計值得注意
- **[The most powerful pattern in TypeScript, Discriminated Unions](https://dev.to/dhwang/the-most-powerful-patterns-in-typescript-discriminated-unions-2inb)** — TypeScript 的 discriminated unions 教學，不是新知識但每次 code review 都有人寫錯，推薦轉發給團隊新人
- **[requestAnimationFrame vs requestIdleCallback](https://dev.to/dhwang/requestanimationframe-vs-requestidlecallback-1m8c)** — rAF 是「下一幀之前做」，rIC 是「你閒了再做」，一張表搞定兩者差異，面試必備
- **[Query and manage Marketplace databases from the dashboard](https://vercel.com/changelog/query-and-manage-marketplace-databases-from-the-dashboard)** — Vercel 現在可以直接在 dashboard 上 query Aurora Postgres、Neon、Prisma、Supabase，不用再開另一個 DB client 了
- **[Vercel CLI commands now scoped to local directory](https://vercel.com/changelog/vercel-cli-commands-now-scope-to-the-local-directory)** — `vc project ls` 終於會自動吃 local directory 的 scope 了，這個 DX 小改動等了好久
- **[Software Supply Chain Security After Axios](https://dev.to/jeremy_longshore/software-supply-chain-security-after-axios-3dh4)** — Axios 供應鏈攻擊事後分析，Google 歸因到北韓 UNC1069，每週一億次下載的套件被塞後門只花了三小時
- **[Show HN: TTF-DOOM — A raycaster running inside TrueType font hinting](https://github.com/4RH1T3CT0R7/ttf-doom)** — 有人在 TrueType 字型的 hinting bytecode 裡跑了一個 raycaster，用 glyph「A」的 16 條 contour 當螢幕像素，這已經不是工程是藝術了
- **[14 patterns AI code generators get wrong](https://dev.to/radpdx/14-patterns-ai-code-generators-get-wrong-and-how-to-catch-them-45l9)** — AI 生成的 code 不會 crash 但會有一類特定的 bug：看起來對但其實錯，整理了 14 種常見 pattern，值得加進 code review checklist
- **[A macOS bug that causes TCP networking to stop working after 49.7 days](https://photon.codes/blog/we-found-a-ticking-time-bomb-in-macos-tcp-networking)** — macOS 有個 TCP 計時器 bug，跑 49.7 天後網路會斷，HN 94 分，如果你的 Mac 是 always-on server 這篇必看
- **[Mandelbrot Set in JS — Smooth Scroll Zoom & Fixing Floating-Point Precision](https://dev.to/foqc/mandelbrot-set-in-js-smooth-scroll-zoom-fixing-floating-point-precision-peo)** — 用 Canvas + Web Workers 畫 Mandelbrot，zoom 到第 16 層 floating-point 就爆了，解法寫得很漂亮

---

## 📚 慢慢啃

- **[How to Use Replicate the Right Way in Your Next.js App](https://dev.to/lusrodri/how-to-use-replicate-the-right-way-in-your-nextjs-app-and-ship-a-real-product-with-it-38dg)** — 不是又一篇「call API 就好」的教學，而是用真實產品 Goodbye Watermark 當 case study，講 Next.js 整合 Replicate 的 production patterns 和踩過的坑
- **[How Chat Apps Send Messages Instantly (WebSockets Breakdown)](https://dev.to/neuraldownload/how-chat-apps-send-messages-instantly-websockets-breakdown-400e)** — HTTP 是每句話講完就掛電話，WebSocket 是保持通話。用這個比喻展開的 WebSocket 完整解析，適合想搞懂即時通訊底層原理的人
- **[Zooming UIs in 2026: Prezi, impress.js, and why I built something different](https://news.ycombinator.com/item?id=47665194)** — 比較了三種 web zooming UI 的實作方式和 trade-off，如果你在做 canvas-based 互動介面，這篇的技術選型分析很有參考價值
- **[Webhooks Are Broken by Design — So I Built a Fix](https://dev.to/roombambar9/webhooks-are-broken-by-design-so-i-built-a-fix-4pk7)** — Webhook 看起來簡單（一個 POST 嘛），但 retry、ordering、idempotency、auth 全都是坑，這篇系統性地列出問題並提出解法
