---
title: "前端框架 Bundle Size 大亂鬥、JS 為什麼不是函數式語言、Better Auth 接班 Auth.js"
date: 2026-04-14
description: "有人用 TodoMVC 做了前端框架 bundle size 基準測試，結果 Vue 3 碾壓 React 和 Angular；一篇從面試觀察出發的深度分析告訴你 JS 離真正的 FP 有多遠；Auth.js 團隊加入 Better Auth，前端認證生態正在洗牌。"
tags: [frontend, react, typescript, web-platform, tooling]
---

今天三道硬菜：有人認真做了前端框架 bundle size 基準測試，數據會讓某些人不太舒服；一篇從兩年面試經驗出發的文章解釋了為什麼你在 JS 裡寫的「函數式」其實不太函數式；然後 Auth.js 團隊跑去 Better Auth 了，你的認證方案可能該重新評估一下。

---

## 🔧 今日硬菜

### [Frontend Framework Bundle Size Benchmark: React/Vue/Angular vs Fine-Grained Runtimes](https://dev.to/qingkuai/frontend-framework-bundle-size-benchmark-reactvueangular-vs-fine-grained-runtimes-2nk0)

終於有人做了一件大家嘴上說重要但沒人認真量化的事：用同一個 TodoMVC 規格，在統一的測量條件下比較各框架的 bundle size。數據很直白——單一元件 minified 後，Angular 201 KB、React 189 KB、Vue 3 只要 65 KB。差距主要來自 runtime 成本：React 和 Angular 的 runtime 都超過 185 KB，Vue 3 只有 61 KB。

Fine-grained reactivity 那邊更有意思。Solid 起點最小（19 KB），但隨著元件數量增加，成長曲線反而比較陡。Svelte 4 的 1 元件只要 11 KB，看起來很美，但到大型應用時 bundle 膨脹速度最快——因為它把實作邏輯嵌進每個元件輸出，沒有共用 runtime 來攤提成本。

**重點：**
- Vue 3 在主流框架中 bundle size 優勢明顯，runtime 只有 React 的三分之一
- Fine-grained 不等於小——Solid 起點最低但成長最快，選框架要看「斜率」不是只看起點
- 但是... bundle size 只是效能拼圖的一塊，runtime 執行效率、tree-shaking 效果、實際載入策略都會影響最終體驗

### [Why JS/TS Is Not a Functional Language (And Why It Matters)](https://dev.to/divide_/why-jsts-is-not-a-functional-language-and-why-it-matters-1hp8)

這篇從兩年面試 JS/TS 工程師的觀察出發，系統性地解釋了 JS 為什麼不是函數式語言——不是說你不能在 JS 裡寫 FP 風格，而是語言本身的設計讓 FP 成為逆流而上。八個面向逐一拆解：`const` 只鎖定 reference 不鎖定資料、V8 刻意不實作 TCO（因為會毀掉 stack trace 和 DevTools 體驗）、錯誤是控制流爆炸而不是值、沒有 lazy collection、沒有 HKT、沒有 pattern matching。

最讓我有感的是「語言引力」那段：Haskell 和 Scala 的引力把你往 FP 拉，而 JS 的引力把你往 imperative 拉。你在 JS 裡堅持寫 FP 不是因為語言支持你，是因為你在跟語言的預設行為對抗。踩過這個坑的人都知道——團隊裡推 pure function 和 immutability，推到最後只剩你一個人在堅持，其他人早就 `let` 配 `push` 寫得很開心了。

**重點：**
- V8 不實作 TCO 是刻意的 trade-off：stack trace 和 DevTools 體驗 > 函數式遞迴優化
- TypeScript 的 discriminated union 看似有 pattern matching，但 `sealed` 語意不存在，exhaustiveness check 只在單一檔案內有效
- 但是... Effect TS 正在用不同路線（單一 `Effect<A, E, R>` 型別）繞過 HKT 限制，值得關注

### [Clerk vs Better Auth (2026) — We Verified Every Price So You Don't Have To](https://dev.to/thiago_alvarez_a7561753aa/clerk-vs-better-auth-2026-we-verified-every-price-so-you-dont-have-to-13pk)

Auth.js 團隊在 2025 年 9 月加入 Better Auth，這件事的影響比你想的大。Auth.js 曾經是 Next.js 認證的事實標準，現在它的精神繼承者是 Better Auth——一個開源、self-hosted 的方案。

Clerk 的 DX 確實是 9/10，十分鐘搞定認證，但數學不會騙人：10 萬 MAU 月費 $2,025 美金。Better Auth 同樣規模的成本？你的 Postgres 月費，大概 $25-50。一年差距 $24,000，某些市場夠請一個工程師了。另外 Clerk 的用戶資料只存在美國伺服器，沒有 EU data residency 選項，GDPR 合規是個隱患。

**重點：**
- Auth.js 團隊加入 Better Auth 是前端認證生態的重要訊號，社群和生態整合會加速
- Clerk 10K MAU 以下免費很香，但 scale up 後的帳單會讓你重新思考人生
- 但是... Better Auth 的 DX 只有 7/10，沒有現成 UI 元件，預計要多花 1-3 天自己刻登入介面

---

## ⚡ 一句話帶過

- **[Shadow DOM CSS Isolation: How to Embed a Widget Without Breaking the Host Page](https://dev.to/issuecapture/shadow-dom-css-isolation-how-to-embed-a-widget-without-breaking-the-host-page-4oio)** — 做過第三方嵌入元件的都懂那個 CSS 互相汙染的痛，Shadow DOM 是目前唯一可靠解法
- **[GitHub Stacked PRs](https://github.github.com/gh-stack/)** — GitHub 官方終於出了 stacked PR 工具，387 個 HN upvotes 說明大家等這個等多久了
- **[You're Probably Not Testing Accessibility the Way Users Experience It](https://dev.to/r3ticular/youre-probably-not-testing-accessibility-the-way-users-experience-it-46em)** — Axe 和 Lighthouse 告訴你「通過了」，但螢幕閱讀器使用者聽到的可能完全不是你想的
- **[Your VS Code Extensions Are a Supply Chain Attack Surface](https://dev.to/thegdsks/your-vs-code-extensions-are-a-supply-chain-attack-surface-3gp4)** — 一個叫 GlassWorm 的攻擊活動透過假 WakaTime 擴充套件植入 Zig 編譯的後門，去檢查你的擴充套件清單
- **[Why React Native Builds Break After Updating Dependencies](https://dev.to/asta_dev/why-react-native-builds-break-after-updating-dependencies-and-how-to-fix-it-a27)** — React Native 的依賴地獄永遠不缺新故事，這篇算是比較完整的除錯指南
- **[Copy-to-Prompt instructions now available for Flags](https://vercel.com/changelog/copy-to-prompt-instructions-now-available-for-flags)** — Vercel Feature Flags 現在可以直接把設定指令複製給 AI agent，DX 小改進
- **[MUI v4/v5 Style Conflicts in single-spa](https://dev.to/czhoudev/mui-v4v5-style-conflicts-in-single-spa-a-complete-debug-record-438k)** — 微前端 + MUI 版本衝突的完整除錯紀錄，踩過的人看了會流淚
- **[How to make Firefox builds 17% faster](https://blog.farre.se/posts/2026/04/10/caching-webidl-codegen/)** — 快取 WebIDL codegen 就省了 17% 建置時間，有時候最大的優化就是「別重複做同一件事」
- **[Neon vs Supabase (2026)](https://dev.to/thiago_alvarez_a7561753aa/neon-vs-supabase-2026-database-or-backend-the-real-tradeoffs-3ggn)** — Neon 被 Databricks 以 $1B 收購、Supabase 破百萬資料庫，Postgres 生態的選擇比以前更複雜了
- **[Someone Bought 30 WordPress Plugins and Planted a Backdoor in All of Them](https://anchor.host/someone-bought-30-wordpress-plugins-and-planted-a-backdoor-in-all-of-them/)** — 651 個 HN upvotes，有人買下 30 個 WordPress 外掛然後全部植入後門，供應鏈安全不是開玩笑的

---

## 📚 慢慢啃

- **[Towards an Open Source Print-Ready Publication Library in JavaScript](https://dev.to/kadetr/towards-an-open-source-print-ready-publication-library-in-javascript-19ba)** — 用 Node.js 做排版引擎 paragraf，實作了業界標準的連字和斷行演算法，讀完會對文字排版有全新認識
- **[How WebAssembly Powers Free Browser Tools With No Login](https://dev.to/nologintools/how-webassembly-powers-free-browser-tools-with-no-login-1bcf)** — 從 Squoosh 的圖片壓縮到各種零伺服器瀏覽器工具，WebAssembly 正在改變「工具一定要後端」的假設
- **[Giving AI Agents Eyes: 6 Tricks for Reading Web Pages Without Vision Models](https://dev.to/gauravchodwadia/giving-ai-agents-eyes-part-1-6-tricks-for-reading-web-pages-without-vision-models-4193)** — 不用 vision model 也能讓 AI agent 理解網頁結構的六種技巧，前端 × AI 整合的實用參考
- **[Building a UK-wide fuel price tracker with Next.js and the CMA open data scheme](https://dev.to/ozzieuk/building-a-uk-wide-fuel-price-tracker-with-nextjs-and-the-cma-open-data-scheme-136b)** — 用 Next.js 串接英國政府公開 API 做油價追蹤器，從 API 設計到部署的完整實戰紀錄
