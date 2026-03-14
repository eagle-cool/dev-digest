---
title: "Vite 8 用 Rolldown 幹掉 Rollup、Claude Code 百萬 token 上線、React Hooks 避坑"
date: 2026-03-14
description: "Vite 8 正式換上 Rust 寫的 Rolldown，Linear 建置時間從 46 秒砍到 6 秒。Claude Code 開放百萬 token 上下文視窗。附贈 React Hooks 經典踩坑指南。"
tags: [frontend, react, tooling, ai]
---

今天三件大事。Vite 8 把 Rollup 和 esbuild 都換成 Rolldown 了——一個用 Rust 寫的 bundler，Linear 的建置時間直接從 46 秒掉到 6 秒，這數字不是跑分，是真實 production。然後 Claude Code 開放百萬 token 上下文視窗，終於可以把整個 codebase 丟進去了（但你的錢包要小心）。最後，有人整理了 React Hooks 的經典踩坑清單，看完你會懷疑自己是不是真的會寫 React。

---

## 🔧 今日硬菜

### [Vite 8 Shipped Rolldown. Linear's Builds Went From 46 Seconds to 6.](https://dev.to/adioof/vite-8-shipped-rolldown-linears-builds-went-from-46-seconds-to-6-2cki)

Vite 從第一天開始就有人格分裂的問題——開發用 esbuild，生產用 Rollup。兩套工具、兩種行為、兩套 bug。每個 Vite 使用者都踩過「dev 沒問題 prod 爛掉」的經典坑：tree-shaking 不一樣、chunk splitting 不一樣、什麼都不一樣。Vite 8 直接把這兩個都幹掉，換上 Rolldown——一個用 Rust 寫的 bundler，一條 pipeline 打天下。踩過那個 dev/prod 不一致坑的人，大概會想站起來鼓掌。

**重點：**
- Linear 建置時間 **87% 降幅**（46s → 6s），Ramp 快了 57%，Mercedes-Benz.io 快了 38%——這些都是真實 production 數字，不是 synthetic benchmark
- 大部分現有 Vite 插件直接相容 Rolldown，遷移基本上就是升級加測試，還有 registry.vite.dev 可以查插件相容性
- 但代價是 Vite 多了約 15MB（打包了 lightningcss 和 Rolldown binary），部分進階設定（如 `manualChunks`）需要改成新的 `rolldownOptions` 格式。如果你的建置超過 10 秒，這點代價不算什麼

### [Claude Code 1M Context Window Is Live](https://raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/CHANGELOG.md)

Anthropic 把 Claude Opus 4.6 和 Sonnet 4.6 的 context window 開到了一百萬 token——Max、Team、Enterprise 方案都有，不用額外加價。意思是你理論上可以把整個中型 codebase 塞進去做 refactor 或跨檔案依賴分析。從 200K 跳到 1M，聽起來很爽，但先冷靜一下。

有人追蹤了自己一個月的 token 用量，發現最燒錢的 session 其實都在做簡單的事——只是 context 被灌得太肥。大部分任務根本不需要百萬 token，單一函式修改、單檔案 bug fix 用標準 context 就夠了。百萬 token 的甜蜜點是全局重構和 legacy 系統理解，其他場景你只是在燒錢。

**重點：**
- 1M context 適合：全 codebase 重構、跨檔案依賴分析、理解 legacy 系統。不適合：寫一個函式、修一個 bug
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` 環境變數可調整自動壓縮閾值，在錯誤時機壓縮可能反而浪費更多 token（因為模型得重新發現已有的 context）
- 真正的省錢秘訣：scoped prompt。告訴模型「只看 src/auth/ 目錄」比「看所有東西然後修 bug」省 5 倍 token

### [React Hooks 避坑指南：那些讓你 debug 到凌晨的陷阱](https://juejin.cn/post/7616598497152712754)

這篇從一個「計數器 bug」開場——快速點擊按鈕 5 次，3 秒後 count 只變成 1。因為 `setTimeout` 裡的閉包捕獲的是點擊當下的 state，5 次都是 `setCount(0 + 1)`。然後一路帶你走過 `useEffect` 依賴陣列的無限循環、多個 `useState` 的競態條件、`useCallback`/`useMemo` 的濫用。寫 React 兩三年的人大概每個坑都踩過，但能像這樣系統性整理出來的不多。

**重點：**
- 閉包陷阱是 Hooks 最常見的坑——用函數式更新（`prev => prev + 1`）或 `useRef` 解決，別直接在 async callback 裡讀 state
- `useEffect` 的依賴陣列裡放物件或陣列？每次 render 都是新 reference，不用 `useMemo` 包就等著無限 re-render
- 但 `useCallback`/`useMemo` 不是免費午餐——它們本身有記憶成本和 GC 壓力，只在確實有 re-render 效能問題時才用。過度優化比不優化更糟

---

## ⚡ 一句話帶過

- **[96% of Devs Don't Trust AI Code. Companies Are Firing Devs Because of It.](https://dev.to/adioof/96-of-devs-dont-trust-ai-code-companies-are-firing-devs-because-of-it-4gkj)** — 72% 開發者每天用 AI 工具，96% 不信任產出的程式碼，然後公司用採用率當理由裁人。這矛盾到可以寫論文了
- **[Copilot CLI Weekly: PR Workflows, Extensions, and Multi-Turn Background Agents](https://dev.to/htekdev/copilot-cli-weekly-pr-workflows-extensions-and-multi-turn-background-agents-37bj)** — GitHub Copilot CLI 出到 v1.0.5，內建 PR workflow、多輪背景 agent、CLI extension 管理。已經不是 autocomplete 了，是整套 AI 開發流程
- **[clasp 3 Dropped TypeScript — Here's a Vite Plugin for Google Apps Script](https://dev.to/wakita181009/clasp-3-dropped-typescript-heres-a-vite-plugin-for-google-apps-script-49k8)** — clasp 3 砍了內建 TypeScript 支援，有人做了 Vite plugin 補上。用 Vite 寫 Google Apps Script，這組合真的沒想過
- **[Vercel → Cloudflare Migration](https://dev.to/ji_ai/vercel-cloudflare-migration-admin-dashboard-ai-news-automation-497c)** — 因為 Vercel 的 cron 要 Pro plan，直接搬到 Cloudflare Pages。三天 20+ commits 搞定，順便建了 admin dashboard
- **[Context Gateway – Compress Agent Context Before It Hits the LLM](https://github.com/Compresr-ai/Context-Gateway)** — 開源 proxy，坐在 coding agent 和 LLM 之間壓縮 tool output。Agent 自己不會管 context，得有人幫它管
- **[Liquid Glass UI — CSS @property Morphing Cards, Zero JavaScript](https://dev.to/ahmod_musa_bd1b2536d20e0e/liquid-glass-ui-2026-css-property-morphing-cards-zero-javascript-24o3)** — 純 CSS 做出液態玻璃卡片效果——conic-gradient 旋轉邊框、backdrop-filter 折射、SVG 噪點。CSS 的炫技天花板又高了
- **[CSS clamp() 完全指南](https://juejin.cn/post/7616542595252289586)** — 一個 `clamp()` 幹掉一堆 media query。如果你還在寫 breakpoint 切字型大小，是時候升級了
- **[Docker Engine v29](https://dev.to/benriemer/docker-engine-v29-a-foundation-release-that-shapes-the-future-51ba)** — 沒有華麗新功能，但底層架構大重構。打地基的版本，重要但不性感
- **[NanoClaw 的創作者花六週從 Side Project 到被 Docker 收購](https://techcrunch.com/2026/03/13/the-wild-six-weeks-for-nanoclaws-creator-that-led-to-a-deal-with-docker/)** — 又一個開源 side project 被大廠收購的故事。勵志歸勵志，但別以為每個 side project 都有這種結局
- **[Next.js Boilerplate：50 小時換你 30 分鐘](https://dev.to/salmanshahriar/i-spent-50-hours-building-a-nextjs-boilerplate-so-you-can-ship-in-30-minutes-61l)** — 包含 auth、i18n、RBAC、SEO 的 Next.js 起手式。又一個 boilerplate，但省下的設定時間是真的

---

## 📚 慢慢啃

- **[虛擬列表完全指南：從原理到實戰，輕鬆渲染 10 萬條資料](https://juejin.cn/post/7616584171635097606)** — 從為什麼 10 萬個 DOM 節點會吃掉 300MB 記憶體講起，到固定高度和動態高度的完整實作。前端效能的必修課，寫得比大部分教科書清楚
- **[前端性能優化終極清單：從 3 秒到 0.5 秒的實戰經驗](https://juejin.cn/post/7616598497152696370)** — 電商網站首屏從 3.2 秒優化到 0.8 秒，跳出率從 45% 降到 28%。Core Web Vitals 各指標的優化方法全覆蓋，附帶真實數據
- **[Vue Router 進階：路由懶載入、導航守衛與元資訊](https://juejin.cn/post/7616584171635081222)** — 超過路由配置的層次，深入 lazy loading、navigation guard、route meta 的企業級實踐。Vue 生態的人值得週末花時間看
- **[JavaScript 的 Symbol.iterator：手寫一個可迭代物件](https://juejin.cn/post/7616564460149899314)** — 從 `for...of` 到自定義 iterator 和 async iterator。JS 基礎功夫，但真正搞懂的人沒想像中多
