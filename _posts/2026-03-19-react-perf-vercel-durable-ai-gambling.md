---
title: "React 效能十式、六人扛三百萬客戶、AI coding 根本是賭博"
date: 2026-03-19
description: "React 效能優化十個資深工程師都在用的 pattern、Vercel 上六個工程師服務三百萬客戶的架構拆解、以及為什麼 AI 寫程式本質上跟拉吃角子老虎機沒兩樣。"
tags: [frontend, react, ai, tooling]
---

今天三道硬菜各有各的味道。一篇把 React 效能調校寫成了速查手冊——不是什麼新東西，但整理得夠實戰；一篇是 Vercel 的客戶案例，六個工程師用 Next.js + AI SDK 撐起三百萬用戶，讀完你會重新思考「要不要自己搭 infra」這件事；最後一篇是 HN 上炸了三百多則留言的文章——有人說 AI coding 本質上就是拉吃角子老虎機，扎心但真實。

---

## 🔧 今日硬菜

### [React Performance Optimisation: 10 Patterns Senior Devs Use](https://dev.to/waqarhabib/react-performance-optimisation-10-patterns-senior-devs-use-1m00)

老實說，這十個 pattern 對有經驗的 React 開發者來說不算新東西——`useMemo`、`useCallback`、`React.memo`、code splitting、虛擬化長列表、`useTransition`、Context 拆分。但作者把它們整理成一份「效能急救手冊」的方式是對的：先 Profile，再動手。文章最精華的部分是那句 20/80 法則——80% 的效能提升來自三件事：消除不必要的 re-render、code splitting、虛擬化長列表。剩下七個 pattern 處理剩餘 20%。踩過 React 效能坑的人都知道，最怕的不是不會優化，是優化錯地方。

**重點：**
- Profile First：React DevTools Profiler 是免費的，先看火焰圖再動手，別靠直覺
- 80% 的效能提升來自 `memo` + `useCallback`、路由級 code splitting、長列表虛擬化
- 但是... 這些 pattern 在 React 19 的 compiler 時代可能需要重新審視——自動 memoization 來了，手動 `useMemo` 的日子可能不多了

### [360 billion tokens, 3 million customers, 6 engineers](https://vercel.com/blog/360-billion-tokens-3-million-customers-6-engineers)

Durable 是一個 AI 驅動的網站建置平台，讓創業者幾分鐘內就能上線。重點數字：三百萬客戶、每天 11 億 token、每年 3600 億 token——六個工程師，沒有專職 DevOps。他們全押 Vercel 的技術棧：Next.js、Fluid Compute、AI SDK、AI Gateway，連多租戶路由都交給 Vercel CDN + Domains API 處理。這篇的 trade-off 很明確——他們用「平台鎖定」換「不用養 infra 團隊」。CTO 的原話是：「我們是 AI 原生應用，應該專注在用 agent 創造價值，不是在搭 AWS infrastructure。」聽起來很爽，但你得問自己：當 Vercel 調價或功能不合需求時，你的逃生路線在哪？

**重點：**
- 核心解決三個問題：模型調度（不被供應商綁死）、租戶隔離（防止上下文洩漏）、成本歸因（per-customer 的 AI 花費追蹤）
- 比自建 infra 便宜 3-4 倍，每位工程師產出放大 10 倍
- 但是... 六人團隊的「神話」建立在 Vercel 全家桶之上，平台依賴度極高——這是 trade-off，不是免費午餐

### [AI coding is gambling](https://notes.visaint.space/ai-coding-is-gambling/)

HN 上 280 分、328 則留言，顯然戳到了很多人的痛點。作者的論點很簡單：AI coding 的體驗本質上就是拉吃角子老虎機——每次 prompt 就是拉一次桿，你在賭它這次會不會產出你要的東西。這解釋了為什麼它讓人上癮——科技業最擅長的機制不就是賭博嗎？pull-to-refresh 拉了這麼多年。但作者真正想說的是靈魂層面的問題：coding 原本是「想通一件事然後實現它」的過程，現在變成「收拾 AI 搞砸的殘局」。從 figuring out 變成 mopping up，成就感大打折扣。這篇不長，但如果你最近覺得 AI coding 讓你焦躁又停不下來，值得讀一下。

**重點：**
- AI coding 的上癮機制跟賭博一樣：每次 prompt 都是一次「拉桿」，賭它產出對的東西
- 真正的問題不是效率，是滿足感——從「解決問題」變成「清理 AI 的爛攤子」
- 但是... 作者自己也承認他不是典型工程師（設計師背景），你的 mileage may vary

---

## ⚡ 一句話帶過

- **[MiniMax M2.7 is live on AI Gateway](https://vercel.com/changelog/minimax-m2.7-on-ai-gateway)** — Vercel AI Gateway 新增 MiniMax M2.7，主打 multi-agent 協作和 agentic workflow，又一個要你測的模型
- **[How We Use Next.js generateStaticParams to Pre-render 3,500 Baby Name Pages](https://dev.to/yunhan_dev/how-we-use-nextjs-generatestaticparams-to-pre-render-3500-baby-name-pages-2ck1)** — Next.js SSG 的教科書實作，三種動態頁面、3500+ 頁預渲染，`generateStaticParams` 用法很標準
- **[Edge Rendering Tactics for Personalized Landing Pages](https://dev.to/jasonbiondo/edge-rendering-tactics-for-personalized-landing-pages-that-convert-without-compromising-speed-1l7d)** — 個人化 landing page 想快又想動態？Edge rendering 的各種姿勢都在這了
- **[Automagically handling auth dependencies in Playwright](https://dev.to/jfuze/automagically-handling-auth-dependencies-in-playwright-4el3)** — Playwright 測試的 auth 管理一直是噩夢，這篇的 dependency-based 方案很實用
- **[PM2 has no web UI. Every open source alternative is dead. So I built one.](https://dev.to/orchidfiles/pm2-has-no-web-ui-every-open-source-alternative-is-dead-so-i-built-one-4ej3)** — PM2 每週兩百萬下載卻沒有開源 Web UI，有人終於受不了自己做了一個
- **[Why Your WebRTC App Breaks After 3 Users](https://dev.to/lakshya_purohit_8acdb147e/why-your-webrtc-app-breaks-after-3-users-and-how-zoom-fixes-it-3i0b)** — WebRTC P2P 在第三個用戶加入時就爆了，SFU 架構入門解說
- **[Vite, Vue 3, and Laravel 11: The Ultimate Zero-Config Local Dev Stack](https://dev.to/james_miller_8dc58a89cb9e/vite-vue-3-and-laravel-11-the-ultimate-zero-config-local-dev-stack-jdi)** — 從 Webpack 年代熬過來的人看到 Vite HMR 的速度都會感動落淚
- **[What MCP Servers Are and Why They Matter for Developer Tools](https://dev.to/dennis-ddev/what-mcp-servers-are-and-why-they-matter-for-developer-tools-1iap)** — MCP 入門科普，把 Anthropic 的 Model Context Protocol 講得挺清楚的
- **[I built a 6.6KB embeddable weather widget](https://dev.to/sellinios/i-built-a-66kb-embeddable-weather-widget-heres-how-it-works-aa)** — 6.6KB 的天氣 widget，支援五百萬地點和 51 種語言，這才叫前端工匠精神
- **[Git worktree like a boss](https://dev.to/metal3d/git-worktree-like-a-boss-2j1b)** — `git worktree` 是 Git 裡最被低估的功能，用對了再也不用 stash 來 stash 去

---

## 📚 慢慢啃

- **[We Can't Code Anymore. AI Won't. What Then?](https://dev.to/bagro/we-cant-code-anymore-ai-wont-what-then-46h3)** — 30 年 IT 老兵的反思：當大家都說不用學 coding 了，但 AI 又寫不出真正可靠的程式碼，中間的 gap 誰來填？讀完會重新思考「工程師」這個職位的本質
- **[The Vibe Check](https://dev.to/thesythesis/the-vibe-check-5b45)** — YC 最新一批有 25% 的 codebase 是 95% AI 生成的，但 45% 的 AI 程式碼有安全漏洞。當 vibe coding 的創辦人連自己都沒寫一行程式碼就上線了，我們該怎麼看待 code quality？
- **[Why Spec-Driven Development Fails](https://dev.to/casamia918/why-spec-driven-development-fails-and-what-we-can-learn-from-it-2pec)** — SDD 說好的「寫 spec 丟給 AI 就能出 code」為什麼在實戰中失敗了？核心假設就是錯的，但裡面有些可以救的方法論
- **[Your Cookie Consent Banner Is Probably Breaking Your Analytics](https://dev.to/anjab/your-cookie-consent-banner-is-probably-breaking-your-analytics-5c4h)** — 用戶拒絕 cookie 之後你的 GA 就直接瞎了，這篇聊 privacy-first analytics 的替代方案，GDPR 時代的前端工程師必讀
