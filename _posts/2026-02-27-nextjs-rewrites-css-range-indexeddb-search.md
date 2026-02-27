---
title: "Next.js Rewrites 被低估了、CSS 開始自己算邏輯、IndexedDB 搜尋不再卡頓"
date: 2026-02-27
description: "深入 Next.js Rewrites 的架構意義、CSS Range Syntax 讓樣式層自己做條件判斷、IndexedDB 前端全文搜尋的倒排索引實戰。另有 rev-dep 前端清理工具、MCP Git Server 漏洞、Claude Code 技術選型分析等。"
tags: [frontend, nextjs, css, typescript, tooling, ai]
---

今天三道硬菜都跟「你以為你會，但其實沒用到精髓」有關。Next.js Rewrites 不只是路由對映，是讓你的 URL 架構跟實作徹底解耦的基礎設施；CSS 正在從「被動接受 class」進化到「自己判斷條件再渲染」；然後，拜託別再 `getAll()` 加 `filter()` 了，你的 IndexedDB 搜尋可以快一百倍。

---

## 🔧 今日硬菜

### [Understanding Next.js Rewrites](https://dev.to/cole_ruche/understanding-nextjs-rewrites-234j)

大多數人用 Next.js 就是 routing、SSR、API routes，然後就沒了。但 Rewrites 這個功能被嚴重低估——它讓你能在不改變瀏覽器 URL 的前提下，把請求導向完全不同的目的地。聽起來很像 redirect？差遠了。Redirect 是叫瀏覽器重新發請求（使用者看得到 URL 變了），Rewrites 是 Next.js 內部處理完，瀏覽器毫無感覺。

這在架構上的意義是什麼？URL 是一個公開契約。一旦使用者、爬蟲、外部系統依賴了你的 URL，改它的成本就極高。Rewrites 讓你保留這個契約的同時，底層隨便重構。最實用的場景是 API Proxying——前端打 `/api/users`，實際請求到 `https://external-service.com/users`，避開 CORS、藏好 API key、前端不用改。更進階的是 conditional rewrites，根據 header 判斷同一個 URL 導向不同頁面，搞 multi-tenant 或 feature flag 都行。

**重點：**
- Rewrites 在路由解析之前執行，`req.url` 不一定反映最終目的地，middleware 要小心測
- 最佳使用場景：API proxy、漸進式遷移、URL 跟實作解耦
- 但是... Rewrites 太隱形了，團隊成員如果不知道有設定 rewrite rules，debug 起來會很痛苦

### [Rethinking UI State: CSS Range Syntax vs Class Toggling](https://dev.to/polyuretanio/rethinking-ui-state-css-range-syntax-vs-class-toggling-2c75)

這篇探討一個我覺得很有趣的方向：以前 UI 狀態都是 JS 迴圈加 `classList.add/remove`，現在 CSS 正在長出自己判斷條件的能力。用一個日曆選取範圍的例子來說——傳統做法是 JS 遍歷每個 day 元素、比較數值、切換 class。新做法？JS 只設定 `--day-start` 和 `--day-end` 兩個 CSS custom properties，然後 CSS 用 Range Syntax 自己判斷：`if(style(--day-start <= --day <= --day-end))` 就上色。

沒有迴圈，沒有 class toggling，沒有 DOM mutation。JS 負責狀態，CSS 負責呈現——關注點分離做到極致。而且重構 DOM 結構也不怕壞，因為 JS 根本不 query DOM。

目前 Range Syntax 還需要 Chrome 142+（實驗性），但文章也給了 `clamp()` 配合算術運算的 fallback 方案，今天就能用。

**重點：**
- CSS custom properties + Range Syntax = 視覺邏輯回歸 CSS 層
- Fallback 用 `clamp()` 模擬 AND 邏輯，verbose 但 production-ready
- 但是... 團隊要有共識。CSS 裡塞條件邏輯對很多人來說可讀性不如 JS 直覺

### [毫秒级响应：前端本地搜索的"降维打击"](https://juejin.cn/post/7611143309210615817)

你還在用 `getAll()` 配 `Array.prototype.filter()` 在 IndexedDB 裡搜東西嗎？資料量一破萬，主線程直接卡到掉幀。這篇從根本問題講起：IndexedDB 的索引是 B-Tree，只支援前綴匹配，不支援 `LIKE %keyword%`。所以要嘛引入 FlexSearch（目前 Web 端最快的全文搜索庫，比 Fuse.js 快一個數量級），要嘛自己手寫倒排索引——利用 `multiEntry: true` 讓 IndexedDB 為陣列中每個元素建獨立指標。

更進階的做法是把整個搜尋邏輯丟進 Web Worker，主線程只管接收輸入和渲染結果。加上防抖、分片載入、搜尋結果 `<mark>` 高亮，體感可以做到毫秒級。這個架構思路在任何需要前端離線搜尋的場景都適用。

**重點：**
- FlexSearch 是前端全文搜索首選，tokenize + cache 配置後開箱即用
- `multiEntry` 倒排索引是零依賴方案，純 IndexedDB 原生能力
- 但是... 中文分詞永遠是大坑。二元分詞（bigram）是最簡單的妥協方案，精確度還行

---

## ⚡ 一句話帶過

- **[GraphQL to TypeScript: Automated Code Generation Guide](https://dev.to/arenasbob2024cell/graphql-to-typescript-automated-code-generation-guide-1e8p)** — 還在手寫 GraphQL 的 TypeScript 型別？codegen 設定完一次，schema 改了型別自動跟，省下的 debug 時間夠你喝三杯咖啡
- **[Show HN: Rev-dep – 20x faster knip.dev alternative](https://github.com/jayu/rev-dep)** — 用 Go 重寫的 unused export 偵測工具，號稱比 knip 快 20 倍。前端 monorepo 清垃圾的又多了一個選擇
- **[I built a real-time audio pipeline from the browser to my server](https://dev.to/flo152121063061/i-built-a-real-time-audio-pipeline-from-the-browser-to-my-server-heres-what-actually-works-5465)** — 瀏覽器到 server 的即時音訊串流，聽起來兩行就能搞定但其實踩坑無數。做過 WebRTC 的人都懂
- **[Why I Built a Filesystem for the Browser](https://dev.to/apireno/why-i-built-a-filesystem-for-the-browser-3kpa)** — 給 AI agent 用的瀏覽器檔案系統抽象層。把 raw HTML 轉成結構化操作介面，思路挺有意思
- **[存储配额：用 navigator.storage.estimate() 预判浏览器什么时候会删你的数据](https://juejin.cn/post/7610971570484445235)** — 瀏覽器是個無情房東，磁碟空間不夠就自動清你的 IndexedDB，連通知都不給。用 Storage API 做生存預判是正經事
- **[CSS to Tailwind: The Complete Migration Guide for 2026](https://dev.to/arenasbob2024cell/css-to-tailwind-the-complete-migration-guide-for-2026-1cgn)** — 逐屬性對照的遷移指南，適合正在搬家的團隊當 cheatsheet 用
- **[CVE-2026-27735: MCP Git Server Path Traversal](https://dev.to/cverports/cve-2026-27735-git-outta-here-exfiltrating-secrets-via-cve-2026-27735-5dff)** — MCP Git Server 的路徑穿越漏洞，CVSS 6.4。用 MCP 工具鏈的人注意一下，LLM 可能被誘導 commit repo 以外的檔案
- **[What Claude Code Chooses](https://amplifying.ai/research/claude-code-picks)** — 有人分析了 Claude Code 的技術選型偏好，HN 上 218 個讚。用 AI 寫 code 的人值得看看它的「品味」
- **[JavaScript Generators and Iterators: A Practical Guide](https://dev.to/arenasbob2024cell/javascript-generators-and-iterators-a-practical-guide-3bha)** — JS 最被低估的特性之一。`yield` 用得好，async flow 和 lazy evaluation 都能優雅很多
- **[Preview Deployments with Firebase Hosting & GitHub Actions](https://dev.to/ozantunca/preview-deployments-with-firebase-hosting-github-actions-27ag)** — PR preview deployment 的完整設定教學，DX 提升立竿見影
- **[How I Cut My AI Coding Agent's Token Usage by 65%](https://dev.to/nicolalessi/how-i-cut-my-ai-coding-agents-token-usage-by-65-without-changing-models-47m)** — 問題不在 model，在於你餵了太多垃圾 context。跟 prompt engineering 一樣，少即是多

---

## 📚 慢慢啃

- **[Will vibe coding end like the maker movement?](https://read.technically.dev/p/vibe-coding-and-the-maker-movement)** — HN 301 讚的長文，把 vibe coding 跟當年 maker movement 類比。當熱潮退去，留下的是工具還是泡沫？值得靜下來想想
- **[JPEG vs WebP vs AVIF in WordPress: Real Benchmark Data](https://dev.to/biancarus/jpeg-vs-webp-vs-avif-in-wordpress-real-benchmark-data-4-plugins-tested-j83)** — 同一張圖、同一個 WordPress、4 個 plugin 實測。AVIF 壓縮率最高到 91%，但 plugin 之間的差異比你想像的大
- **[Deep Dive: How Claude Code Remote Control Actually Works](https://dev.to/chwu1946/deep-dive-how-claude-code-remote-control-actually-works-50p6)** — 掃個 QR code 就能在手機上接管筆電的 Claude Code session。沒有 SSH、沒有 port forwarding——這到底怎麼做到的？20 分鐘的技術拆解
- **[Peeking Under the Hood: How Cloudflare R2 Really Works](https://dev.to/krish_kakadiya_5f0eaf6342/peeking-under-the-hood-how-cloudflare-r2-really-works-and-why-your-frontend-apps-will-thank-1nmg)** — 前端圖片和靜態資源從 S3 搬到 R2 能省多少 egress 費用？這篇從架構面解釋為什麼 R2 對前端應用特別友善
