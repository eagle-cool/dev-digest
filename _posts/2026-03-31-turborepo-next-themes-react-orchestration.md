---
title: "Turborepo 快 96%、next-themes 有人接手了、React 該怎麼管複雜流程"
date: 2026-03-31
description: "Vercel 把 Turborepo task graph 加速 96%、next-themes 棄更一年終於有人寫了替代品、React Orchestration Pattern 讓你的 useEffect 地獄解脫"
tags: [frontend, react, nextjs, tooling]
---

今天三件事值得坐下來好好看。Turborepo 2.9 用人機協作把 task graph 跑快了 96%，技術細節紮實到可以當效能優化教材。next-themes 那個每週 2200 萬下載的套件擺爛一年了，終於有人受不了寫了 drop-in replacement。然後有篇 React Orchestration Pattern 的文章，專治你那些寫到自己都看不懂的 useEffect 連鎖反應。

---

## 🔧 今日硬菜

### [Making Turborepo 96% faster with agents, sandboxes, and humans](https://vercel.com/blog/making-turborepo-ninety-six-percent-faster-with-agents-sandboxes-and-humans)

Vercel 這篇是近期最紮實的效能優化實戰紀錄。Turborepo 2.9 在 1000+ package 的 monorepo 上，task graph 計算從 8.1 秒壓到 0.716 秒——91% 的提升，Time to First Task 快了 11 倍。手段不花俏但刀刀見血：把循序操作改成並行（git index、glob、lockfile parsing 同時跑）、砍掉冗餘的記憶體分配（stack-allocated Git OID 直接省掉上萬次 heap allocation）、用 `gix-index` 取代 `libgit2` 再取代 git subprocess 來壓 syscall 數量。最有意思的是方法論：先讓 8 個 AI agent 跑一輪找低垂果實，再由人類工程師用自製的 Markdown profiling 格式引導 agent 做精準優化，四天內產出 20+ 個 PR。人機協作不是讓 AI 自己搞，是人類全程帶節奏。

**重點：**
- 1000+ package monorepo：8.1s → 0.716s（91% faster），小型 repo 也有 80% 提升
- 關鍵手段：並行化、消除記憶體分配、減少 syscall（三個 stat 壓成一個 open）
- 但是... 這些優化高度依賴 Rust 層面的底層控制，JS 生態的工具想照搬不太容易

### [I was tired of next-themes being abandoned, so I built a drop-in replacement](https://dev.to/wrksz/i-was-tired-of-next-themes-being-abandoned-so-i-built-a-drop-in-replacement-4dbk)

next-themes 每週 2200 萬下載，43 個 open issues，16 個 PR 沒人合，一年沒 release。React 19 噴 script warning、`cacheComponents` 導致 theme 卡住、production minification 靜默壞掉——這些 bug 全在那邊躺著。作者受不了了，寫了 `@wrksz/themes` 做 drop-in replacement，遷移只要改一行 import。技術上做對了幾件事：用 `useServerInsertedHTML` 取代 Client Component 裡的 inline script 解決 React 19 warning；用 `useSyncExternalStore` 搭配 per-instance store 解決 stale theme；加了 cookie storage 實現真正的 zero-flash SSR——不用再靠 `suppressHydrationWarning` 這種 hack。還多了 generic TypeScript types、nested providers、`ThemedImage` 元件這些本來就該有的東西。

**重點：**
- 修掉 next-themes 所有已知 bug：React 19 script warning、stale theme、minification、multi-class removal
- 新增 cookie storage zero-flash SSR、generic TS types、nested providers、meta theme-color
- 但是... 畢竟是個人專案，22M 週下載的生態要遷移需要觀察社群接受度和長期維護承諾

### [Mastering the Orchestration Pattern in React: Taming Complex Component Logic](https://dev.to/maximlogunov/mastering-the-orchestration-pattern-in-react-taming-complex-component-logic-5c9i)

踩過 multi-step form 那種「十幾個 useEffect 互相追趕」的坑的人，看這篇會有共鳴。Orchestration Pattern 的核心概念不複雜：把散落在各元件裡的流程邏輯抽到一個 orchestrator hook，用 `useReducer` 做狀態機管理步驟轉換、錯誤處理、rollback。文章用 checkout 流程當範例，從地址驗證→運費計算→折扣→付款，每一步都有明確的 dispatch action 和狀態遷移。元件變成純粹的 presentation layer，只管呼叫 orchestrator 的方法。好處很實在：可單獨測試流程邏輯（不用 render UI）、同一個 orchestrator 可以給不同 UI 用、加 logging/analytics 只要改一處。但文章最後那個「compose orchestrators」的建議值得注意——別把它變成另一個 god object。

**重點：**
- 核心：用 `useReducer` + custom hook 集中管理複雜多步驟流程，元件只負責渲染
- 適用場景：multi-step form、wizard、有依賴關係的連續 API call、需要 rollback 的操作
- 但是... 簡單 CRUD 別用這個，過度抽象比 useEffect 地獄更難除錯

---

## ⚡ 一句話帶過

- **[WCAG 2.2: What Changed, Why It Matters, and How to Implement It](https://dev.to/thoithoi_shougrakpam_6753/wcag-22-what-changed-why-it-matters-and-how-to-implement-it-98e)** — 九個新 success criteria，如果你的專案還在對標 2.1，你已經落後了
- **[AnalogJS 2.4: Vite 8, Vitest Snapshot Serializers, Astro v6 Support](https://dev.to/analogjs/analogjs-24-vite-8-vitest-snapshot-serializers-astro-v6-support-and-more-3hgo)** — Angular 生態的 meta-framework 跟上 Vite 8，Angular 開發者終於有正經的 SSR 選擇
- **[React Native vs Flutter vs Expo vs Lynx 2026](https://dev.to/krunal_groovy/react-native-vs-flutter-vs-expo-vs-lynx-2026-which-to-choose-for-your-app-30h6)** — ByteDance 的 Lynx 加入戰局，但 2026 年的答案還是看你團隊會什麼
- **[salt-theme-gen: Zero-dependency OKLCH theme generator](https://dev.to/hasansarwer/i-open-sourced-salt-theme-gen-2dph)** — 用 OKLCH 色彩空間生成一致的 theme tokens，dark mode 終於不用靠感覺調
- **[Prometheus Metrics for Node.js Circuit Breakers](https://dev.to/axiom_agent/prometheus-metrics-for-your-nodejs-circuit-breakers-opossum-prom-3b31)** — 你的 opossum circuit breaker 有加監控嗎？沒有的話你根本不知道它什麼時候 open
- **[Designing a Scalable Multi-Tenant Architecture with NestJS](https://dev.to/sircatalyst/designing-a-scalable-multi-tenant-architecture-with-nestjs-lessons-from-building-real-world-systems-4j9)** — NestJS + MongoDB/PostgreSQL 多租戶實戰，踩完坑的經驗比官方文件有用
- **[Your AI Doesn't Generate Bad Designs. You Do!](https://dev.to/alaa_elghamry_144eeef8b71/your-ai-doesnt-generate-bad-designs-you-do-13j6)** — 前端仔用 AI 生 UI 老是醜？問題不在 AI，在你不會下 prompt
- **[The Future Of Software Engineering according to Anthropic](https://dev.to/kodus/the-future-of-software-engineering-according-to-anthropic-30ei)** — AI 寫 code 很快，但 review AI 寫的 code 比自己寫還累，這個矛盾很真實
- **[Centralized Documentation for Oxlint and Oxfmt](https://dev.to/pavkode/centralized-documentation-needed-for-oxlint-and-oxfmt-compatibility-with-frameworks-and-file-types-56d6)** — Oxlint 到底支不支援 React Native？沒人知道，因為文件根本沒寫
- **[Learn Claude Code by doing, not reading](https://claude.nagdy.me/)** — HN 109 分，互動式教學比讀文件有效，AI coding agent 的入門門檻正在降低

---

## 📚 慢慢啃

- **[认识 Service Worker](https://juejin.cn/post/7622896022320988169)** — Service Worker 的完整入門，從 install/activate 生命週期到離線快取策略，PWA 基礎功不能少
- **[I Built a JavaScript Framework From Scratch](https://dev.to/codewithdareios/i-built-a-javascript-framework-from-scratch-heres-what-happened-357p)** — Virtual DOM、template compiler、reactive hooks、router、HMR 全部從零刻，理解框架原理最好的方式就是自己寫一個
- **[The Ultimate Technical SEO Checklist for Developers](https://dev.to/monsur/the-ultimate-technical-seo-checklist-for-developers-2026-edition-3cmd)** — 專門給 Next.js/React 開發者的 SEO checklist，你的 code 再好搜尋引擎找不到也是白搭
- **[Google Calendar — Day View](https://dev.to/arghya_majumder/google-calendar-day-view-42a0)** — 拆解 Google Calendar 日視圖的前後端架構：virtual scrolling、drag-and-drop snapping、重疊事件排版算法，前端面試經典題的完整解法
