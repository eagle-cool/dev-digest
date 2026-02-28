---
title: "React Foundation 成立、Cloudflare 一週重建 Next.js、MCP 供應鏈拉警報"
date: 2026-02-28
description: "React 正式脫離 Meta 成立基金會、Cloudflare 工程師用 AI 一週重建 Next.js 還快 4.4 倍、MCP 生態被掃出兩萬個安全隱患。前端圈今天很熱鬧。"
tags: [frontend, react, nextjs, nodejs, ai, tooling, web-platform]
---

今天三件事值得坐下來好好聊：React 終於脫離 Meta 自立門戶了、Cloudflare 一個工程師用 AI 一週幹出一套比 Next.js 還快的東西、然後 MCP 那邊被人掃出一堆供應鏈地雷。前端圈的週五，比平常刺激。

---

## 🔧 今日硬菜

### [This Week In React #270: React Foundation, Cloudflare Vinext, Oxfmt, Hermes-node](https://dev.to/sebastienlorber/this-week-in-react-270-nextjs-react-router-hermes-react-navigation-css-grid-maestro--106o)

這期 TWIR 資訊密度爆表，挑三個最炸的講。**React Foundation 正式成立**，掛在 Linux Foundation 底下，React、React Native、JSX 不再是 Meta 的私有財產，華為也加入了創始成員。這對整個生態的治理模式是個大轉折——開源項目的命運不再繫於一家公司的戰略調整。

第二個是 **Cloudflare 的 Vinext 專案**：一個工程師用 AI 花了一週、$1,100 美元，重建了 Next.js 的 API 層。用 Oxc/Rolldown 取代 Webpack，build 快 4.4 倍、bundle 小 57%，通過 2000+ 測試、94% API 覆蓋率，而且已經在 cio.gov 跑了。這不只是「AI 寫程式」的 demo，這是在挑戰整個 meta-framework 的建置架構。

第三個：**Oxfmt beta 釋出**，Prettier 相容的 formatter，快 30 倍。前端工具鏈的 Rust 化浪潮又推進了一步。

**重點：**
- React Foundation 成立 = 治理去中心化，社群對 roadmap 有更多話語權
- Vinext 證明現有 bundler 架構有巨大的效能改善空間，Oxc/Rolldown 是真的猛
- 但是... Vinext 只覆蓋 94% API，剩下的 6% 可能正好是你在用的那些 edge case

### [滾動鎖定：AI 串流輸出時如何防止「滾動奪權」](https://juejin.cn/post/7611345744185524262)

做過 AI 聊天介面的都懂這個痛——AI 正在吐字，你想往上翻歷史紀錄，結果新訊息不斷把你的視窗往下推，文字瘋狂跳動。這篇從底層滾動機制切入，給出了一套完整的「視口守衛」方案。

核心思路是用 `IntersectionObserver` 監聽底部哨兵節點，取代效能極差的 `onscroll`。搭配 `overflow-anchor` CSS 屬性處理圖片異步載入造成的高度塌陷，再用 `requestAnimationFrame` 做渲染緩衝避免高頻 `scrollTo` 掉幀。文章還整理了 8 個細分場景的避坑指南，從手機鍵盤彈起到代碼塊高亮渲染都有覆蓋。

**重點：**
- `IntersectionObserver` + 哨兵節點是效能最好的「是否在底部」偵測方案
- AI 串流場景用 `behavior: 'instant'` 而非 `smooth`，避免動畫疊加
- 但是... 這套方案在虛擬滾動（virtual scroll）場景下還需要額外處理，文章沒提到這塊

### [MCP Has a Supply Chain Problem](https://dev.to/0x711/mcp-has-a-supply-chain-problem-1nb8)

還記得 2018 年 `event-stream` 那次供應鏈攻擊嗎？MCP 正在走同一條路，只是更快。這篇掃描了 42,000+ 個 MCP 工具，找出了 **19,830 個安全發現**，其中 485 個是 CRITICAL 等級。

最常見的問題：502 個 MCP server 用 `npx -y` 安裝卻沒有 pin 版本號，代表每次啟動都拉最新版，一旦套件被入侵就自動執行惡意程式碼。1,050 個配置指向遠端伺服器但沒有任何身份驗證。1,679 個工具定義裡包含 `pip install` 指令。你的 AI agent 不只能讀你的檔案，還能在你的機器上安裝軟體——這是設計功能，不是 bug。

**重點：**
- 最低成本的自保：把 `npx -y some-mcp-server` 改成 `npx -y some-mcp-server@1.2.3`
- npm 當年走過的路（lockfile、audit），MCP 生態現在才剛開始
- 但是... MCP 工具不是跑在沙箱裡，是跑在你的 shell、你的檔案系統、你的 credentials 旁邊，爆炸半徑是整台電腦

---

## ⚡ 一句話帶過

- **[Express vs Fastify: The Numbers Behind the Hype](https://dev.to/royce_fabbd83cb268312e928/express-vs-fastify-the-numbers-behind-the-hype-4he2)** — Fastify 80K req/s vs Express 25K，3 倍差距是真的，但 Express 週下載量還是 Fastify 的 14 倍，生態就是這麼現實
- **[React vs Vue in 2026: What the npm Data Actually Says](https://dev.to/royce_fabbd83cb268312e928/react-vs-vue-in-2026-what-the-npm-data-actually-says-4nll)** — React 96M 週下載 vs Vue 9M，10 倍差距，但下載量不等於「適合你的下一個專案」
- **[TailwindCSS v4 Migration Guide](https://dev.to/umesh_malik/tailwindcss-v4-migration-guide-what-changed-and-how-to-upgrade-525g)** — 最大改變：設定從 `tailwind.config.js` 搬到 CSS 裡了，CSS-first configuration，升級比想像中順
- **[當高階函數遇到 AI：自動化生成業務邏輯攔截器](https://juejin.cn/post/7611432097543651355)** — 用 HOF 把權限檢查、Token 審計、異常降級包成可組合的攔截器，AI 最擅長生成這種模式化程式碼
- **[Cloudflare Turnstile & Challenge Pages 大改版](https://blog.cloudflare.com/the-most-seen-ui-on-the-internet-redesigning-turnstile-and-challenge-pages/)** — 號稱網路上最多人看到的 UI 組件重新設計，Cloudflare 的設計團隊終於出手了
- **[Vibe Coded App Exposed 18K Users](https://www.theregister.com/2026/02/27/lovable_app_vulnerabilities/)** — 用 Lovable vibe coding 出來的 app 被抓到一堆基本安全漏洞，暴露 18K 用戶資料，AI 寫的 code 你還是得 review
- **[MCP Benchmark: MCP Won and It Wasn't Close](https://dev.to/tobrun/i-benchmarked-how-claude-code-consumes-apis-mcp-won-and-it-wasnt-close-4k1)** — 有人認真跑了 benchmark，MCP 在 AI agent 消費 API 的場景下完勝 OpenAPI spec，結束爭論
- **[PageAgent: The GUI Agent Living in Your Web Page](https://dev.to/simon_luv_pho/pageagent-the-gui-agent-living-in-your-web-page-1cda)** — 不需要 headless browser，直接在頁面裡跑的 AI agent JS 函式庫，思路有意思
- **[workz: Git Worktrees 的 .env + node_modules 救星](https://dev.to/rohansx/i-built-workz-the-zoxide-for-git-worktrees-that-finally-fixes-env-nodemodules-hell-in-2026-2dpj)** — 用 git worktree 跑多個 AI agent 的人一定踩過這個坑，終於有人做了 symlink 管理工具
- **[Spoosh: 不想再寫 fetcher 和 cache key 了](https://dev.to/nxnom/anyone-else-tired-of-writing-fetchers-and-keys-in-network-calls-4bhm)** — 又一個 data fetching 函式庫，但自動 tag invalidation 和 SSE 支援確實解決了真實痛點

---

## 📚 慢慢啃

- **[Angular to Next.js + GraphQL 大遷徙實錄](https://dev.to/ujja/how-were-surviving-20-domains-and-100-sql-tables-while-migrating-our-legacy-net-backend-to-e38)** — 600+ Angular 元件、20 個 domain、100 張 SQL table 的漸進式遷移，這種規模的實戰經驗很難得，值得認真讀
- **[Cypress 5000 個測試不再重複寫登入](https://dev.to/cypress/how-i-stopped-declaring-login-in-each-of-my-5k-tests-37km)** — Cypress 的 session 快取機制讓你不用在每個測試都跑一次登入流程，測試套件越大省越多
- **[Custom Icon Fonts vs SVG：輕量化圖示策略](https://dev.to/supreet_pradhan/why-custom-icon-fonts-are-the-ultimate-lightweight-icon-strategy-3m85)** — SVG 是主流沒錯，但當你只需要 20 個圖示時，自訂 icon font 可能才是最簡單的答案
- **[JPG to SVG：瀏覽器端向量化原理](https://dev.to/zepubocode/jpg-to-svg-how-vectorization-works-in-the-browser-no-server-required-2ink)** — 不靠 server，純前端把點陣圖轉成向量路徑的兩種方法（嵌入 vs 描邊），搞懂原理後很多場景用得上
