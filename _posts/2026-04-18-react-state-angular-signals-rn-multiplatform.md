---
title: "React 狀態管理該選誰、Angular Signals 砍半複雜度、React Native 不只做手機"
date: 2026-04-18
description: "今天前端圈聊的都是狀態管理：React 那幾套方案的 runtime 差異、Angular 把 ComponentStore 改寫成 Signals 的得失，還有 React Native 跨 7 個平台的經濟學。順便提醒一下 openai-agents 0.13.x 偷偷 break 了 v1。"
tags: [frontend, react, nodejs, tooling, ai]
---

今天值得聊的東西不多，但剛好全都圍繞著同一件事：**狀態管理**。React 老問題被重新討論一次，Angular 把 ComponentStore 改寫成 Signals 然後說「複雜度砍半」，React Native 則是講怎麼靠共享狀態層同時餵飽手機跟電視。巧合嗎？我看不見得——這就是前端成熟到一定程度之後會反覆回來敲打的老主題。

另外，openai-agents 0.13.x 悄悄把 openai v1 支援砍了，有在用的先別急著升。

---

## 🔧 今日硬菜

### [ReactJs Performance ~ State Management ~](https://dev.to/kkr0423/reactjs-performance-state-management-3bj2)

終於有人願意把 Context / Redux / Zustand / Jotai 放到同一張桌子上，**不談 DX 只談 runtime**——這才是真正該問的問題。文章直接點出 Context 的經典坑：你以為 `useContext` 很乾淨，但只要 provider 的 `value` 變一次，底下所有 consumer 全部重 render，`theme` 改個色，整棵 tree 跟著抽搐。作者給的解法也很傳統但正確：**按更新頻率拆 context**（帳號資料放 rare context、主題放 medium、通知放 frequent）；不想拆就搬去 Zustand 用 selector，反正訂閱粒度天生就比 Context 細。

Jotai / Recoil 的 atom-based 心智模型適合衍生狀態很多的場景，但團隊新人上來會先愣三秒。Redux 就是那個「大家都說不需要，但大型 app 一複雜又會偷偷切回去用」的存在。

**重點：**
- Context 不是萬惡，但**拿來放高頻變動的全域狀態就是自找麻煩**
- Zustand 的 selector-based 訂閱是大部分中型 app 最划算的選擇
- 但是...文章對 atom-based 方案著墨不深，真的要選 Jotai 前再多看兩篇 benchmark

### [I Rewrote Angular Component Store with Signals - And Cut the Complexity in Half](https://dev.to/bock92/i-rewrote-angular-component-store-with-signals-and-cut-the-complexity-in-half-32hk)

8 年 Angular 老手把 NgRx ComponentStore 改寫成 Signal Store，然後很老實地說：「這不是語法糖的問題，是**大腦處理這段 code 的方式變了**。」選 RxJS 那套你得在腦中模擬 stream、在 selector / updater / effect 間跳來跳去、還要追蹤 `switchMap` / `forkJoin` 的非同步行為；改成 Signal Store 後就回歸到最原始的「做動作、更新 state」兩步驟。

但他沒迴避 trade-off：**失去的是 `switchMap` 自動取消、stream 組合性、以及跨多個資料源的 reactive 協調**。對一個「載入使用者資料 + 級聯選單」這種功能來說，確實不值得用 RxJS 那套殺雞宰牛，但如果你的 store 要處理即時搜尋 debounce + 可取消請求 + 多串流合併，改寫成 imperative 就會馬上懷念 operators。這篇文章最值得收藏的不是 code，是**「我在解決複雜度，還是只是比較會管理複雜度？」這個問題**——每個資深工程師都該定期問自己一次。

**重點：**
- Signal Store 把狀態管理從「reactive 流」拉回「命令式 flow」，認知負擔顯著下降
- NgRx 不是錯，是**用錯地方**的代價很大
- 但是...`switchMap` 級別的自動取消你得自己手刻，team 裡沒人懂 AbortController 之前先別亂搬

### [React Native Cross-Platform Development: One Codebase for Mobile, TV, and Beyond](https://dev.to/sundr_dev/react-native-cross-platform-development-one-codebase-for-mobile-tv-and-beyond-30jf)

9 年經驗的作者把 2026 年的 React Native 定位講得很清楚：**「跨平台」不再只是 iOS + Android**。RN 0.82 把 New Architecture 設為強制、0.84 直接把舊 bridge 實體移除；Hermes V1 成預設引擎，啟動速度快 55%、記憶體省 26%。Shopify 說他們 code 共用率 86%、畫面載入快 59%；Microsoft 40+ Office 體驗跑在 RN 上；最勁爆的是 Amazon Vega OS 直接把 Hermes 烤進 OS 層，app 冷啟動近乎即時。

文章最有參考價值的是**那個 60-80% 共用的實際內容清單**：Zustand store、API client、資料模型、驗證邏輯——全部跨平台。剩下 20-40% 才是真正吃成本的地方：TV 需要的 D-pad 焦點管理、Tizen 跟 webOS 的 back 鍵 keycode 完全不同（10009 vs 461）、TV 的 512MB 記憶體跑不動手機的那 10 條並行動畫。Flutter vs RN 在這個維度上根本不是對手——Flutter 的 TV 支援還停在實驗階段。

**重點：**
- 「一份 code 上架 7 個平台」在 2026 年對內容型 app 來說是**實務可行**而非 marketing
- 商業邏輯、data layer 幾乎 100% 共用；**真正貴的是焦點管理和輸入事件**
- 但是...如果你做的是 highly interactive 的工具型 app，共用率會掉到 55%，省下的人力沒想像中多

---

## ⚡ 一句話帶過

- **[Building a Multi-Language Pokémon Fansite with Next.js 16 (en/ja/ko/zh)](https://dev.to/tryonfy/building-a-multi-language-pokemon-fansite-with-nextjs-16-enjakozh-4en8)** — Next.js 16 App Router 做 i18n 靜態站的完整實戰，SSG 輸出模式幾個不明顯的坑值得一看
- **[Lessons from building a home services marketplace in Morocco](https://dev.to/allo_maison/lessons-from-building-a-home-services-marketplace-in-morocco-2i96)** — 148 個靜態頁 + App Router + Tailwind 的真實上線心得，build-time 細節比大部分 tutorial 扎實
- **[Per-page OG images on an Astro site, without the SaaS](https://dev.to/dacforge/per-page-og-images-on-an-astro-site-without-the-saas-54ak)** — 每頁動態 OG 圖不靠 Vercel OG 或第三方 SaaS，純 Astro 自幹的方法，省錢又完全可控
- **[I Built a Neumorphic CSS Library with 77+ Components — Here's What I Learned](https://dev.to/_dssid/i-built-a-neumorphic-css-library-with-77-components-heres-what-i-learned-5687)** — 純 CSS 無框架做 77 個 Neumorphic 元件，陰影配方可以抄，但這風格 2020 年就該入土了吧
- **[Convert an ASP .NET MVC application to Vue JS 3 TS page-by-page](https://dev.to/yogesh_bhavsar/convert-an-asp-net-mvc-application-to-vue-js-3-ts-page-by-page-4djn)** — ASP.NET MVC 逐頁遷移到 Vue 3 + TS 的 strangler pattern 實作，需要搞這類遷移的人才懂這種文章多珍貴
- **[How I built a Stripe Webhook in Node.js (Full Guide)](https://dev.to/apollo_ag/how-i-built-a-stripe-webhook-in-nodejs-full-guide-4go0)** — 簽章驗證、錯誤處理、idempotency 都有寫，新手入門剛好，老手快轉到 idempotency 那段就好
- **[Your API is down — now what? Capturing failure context in Node.js](https://dev.to/riyon_sebastian/your-api-is-down-now-what-capturing-failure-context-in-nodejs-2gea)** — API 掛掉當下要在 request lifecycle 各階段撈什麼 context，production 踩過的人看了會點頭
- **[Supabase Row Level Security in Production: Patterns That Actually Work](https://dev.to/whoffagents/supabase-row-level-security-in-production-patterns-that-actually-work-2l78)** — RLS 的雷區比你想的多，policy 寫錯一行整張表就裸奔，這篇講的都是真的踩過才知道的 pattern
- **[PostgreSQL Performance Optimization: Why Connection Pooling Is Critical at Scale](https://dev.to/abdullahmubin/postgresql-performance-optimization-why-connection-pooling-is-critical-at-scale-27bk)** — 資料庫效能瓶頸九成不是 SQL 而是連線數，PgBouncer 還沒裝的後端先看這篇
- **[Building an emoji list generator with the GitHub Copilot CLI](https://github.blog/ai-and-ml/github-copilot/building-an-emoji-list-generator-with-the-github-copilot-cli/)** — GitHub 官方直播用自家 Copilot CLI 現場寫 app，看官方怎麼用自家工具比看 docs 有感
- **[Bringing more transparency to GitHub's status page](https://github.blog/news-insights/company-news/bringing-more-transparency-to-githubs-status-page/)** — GitHub 最近狂宕機之後官方認錯，status page 要更新、可用性要加碼——說好，做不做得到再看
- **[openai-agents 0.13.x Silently Dropped openai v1 Support — Here's What Breaks](https://dev.to/peytongreen_dev/openai-agents-013x-silently-dropped-openai-v1-support-heres-what-breaks-4ibm)** — 沒改 major 版本就砍 v1 相容，這是什麼 semver？升級前先看這份 breaking 清單避雷

---

## 📚 慢慢啃

- **[5 Things I Learned Reverse-Engineering Claude Code's Architecture](https://dev.to/_2b847605e5fbe8a8c9e26/5-things-i-learned-reverse-engineering-claude-codes-architecture-42hb)** — 有人認真把 Claude Code 的 TypeScript 原始碼拆給你看，production-grade agent 架構的五個設計決策——週末泡杯咖啡值得細讀
- **[1x -> 10x -> 100x Engineer with Claude Code and CLIs](https://dev.to/josiahbryan/1x-10x-100x-engineer-with-claude-code-and-clis-fc6)** — 標題看起來很浮誇，但內容是實打實的「怎麼把 Claude Code + 自幹 CLI 串成工作流」深度分享，抄得動就抄一份
- **[Skill-tree: finding the Claude Code habits you never touch](https://dev.to/palo_alto_ai/skill-tree-finding-the-claude-code-habits-you-never-touch-3bm5)** — 拿 Anthropic 的 AI Fluency Index 分類自己的 session，找出你從沒碰過的用法——比自我感覺良好更有用的自我盤點法
- **[Terminal themes tuned for prose legibility, not syntax highlighting](https://dev.to/palo_alto_ai/terminal-themes-tuned-for-prose-legibility-not-syntax-highlighting-g7e)** — AI 時代 terminal 上 80% 畫面是 LLM 推理輸出而非程式碼，配色該為文字可讀性設計而非 syntax highlight，這觀察很準
