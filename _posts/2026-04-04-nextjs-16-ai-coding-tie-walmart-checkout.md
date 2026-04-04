---
title: "Next.js 16 正式上線、AI 寫 Code 三巨頭打平、Walmart AI 結帳慘遭滑鐵盧"
date: 2026-04-04
description: "Next.js 16 帶來 View Transitions 和 Cache Components、三大 AI 模型在 coding benchmark 上打成平手、Walmart 的 ChatGPT 結帳轉換率只有網站的三分之一——今天的前端圈很精彩。"
tags: [frontend, react, nextjs, ai, web-platform, nodejs]
---

今天三件事值得你放下手邊的 code 看一眼：Next.js 16 終於把 View Transitions 和 Turbopack 搬上正式舞台了、三大 AI coding 模型在 benchmark 上打成平手（所以別再吵了）、然後 Walmart 把 20 萬個商品丟進 ChatGPT 結帳，結果轉換率只有自家網站的三分之一——這大概是今年最好的「少即是多」反面教材。

---

## 🔧 今日硬菜

### [Complete Guide to Next.js 16 + React 19.2 in Production — RSC Security, View Transitions, Turbopack](https://dev.to/x4nent/complete-guide-to-nextjs-16-react-192-in-production-rsc-security-view-transitions-turbopack-5090)

Next.js 16 這次的更新含金量不低。React 19.2 的 View Transitions 終於內建了——`next.config.ts` 裡開一個 `viewTransition: true` 就有原生頁面轉場動畫，不用再裝 Framer Motion 只為了做個 fade-in。`<Link>` 還多了 `transitionTypes` prop 可以控制不同方向的動畫，這設計確實比自己手刻 CSS transition 優雅不少。

Cache Components 是架構層面最大的改動：把之前散落各處的 Dynamic IO、`use cache`、PPR 全部統一成一個 `cacheComponents` flag。對於做 hybrid rendering 的團隊來說，這省掉了大量「到底哪個 cache 在管這段」的心智負擔。Turbopack 也從 alpha 升到 beta，Vercel 自己已經在用了——HMR 快了 10 倍，dev server 啟動從 12-30 秒降到 1-3 秒。

但最重要的其實是安全：去年底的 React2Shell（CVE-2025-55182）讓所有用 RSC 的人都該回去檢查一遍。Flight protocol 的 payload 沒做驗證就能 RCE，CVSS 9.8，這不是開玩笑的。文章給了完整的 hardening checklist——Zod 驗證 Server Action、pin 版本號、rate limit RSC Flight requests。踩過坑的人都知道，這些「無聊」的事才是生產環境的命根子。

**重點：**
- View Transitions + Cache Components + Turbopack beta = Next.js 16 三大賣點，都可以直接用了
- React2Shell (CVE-2025-55182) 影響 React 19.0-19.2.0，務必升到 19.2.4+ / Next.js 16.0.11+
- 但是... fetch 預設行為從 cached 改成 no-cache，升級前先檢查你的 data fetching 邏輯，不然上線會很刺激

### [The AI Coding War Is Over. Nobody Won.](https://dev.to/pudgycat/the-ai-coding-war-is-over-nobody-won-52g)

2026 年 3 月的 AI coding benchmark 結果出爐：Claude Opus 4.6、GPT-5.4、Gemini 3.1 Pro 基本上打成平手。但「平手」這個結論本身就很有意思——因為看你選哪個 benchmark，贏家完全不同。SWE-bench Verified？Claude 贏（80.8%）。SWE-bench Pro？GPT-5.4 贏（57.7%）。Terminal-Bench？OpenAI 碾壓。ARC-AGI-2？Gemini 跑得最遠。

真正的故事不是誰贏了，而是大家都到了天花板。文章最有價值的觀察是 model routing：37% 的企業已經在跑 5+ 個模型，根據任務類型分配——便宜的寫文件、中階的做日常開發、頂級的處理大型重構。報導指出這樣做可以降 60-85% 成本，而且只要 50-100 行 routing code。開源模型也在追上來，Qwen3-Coder-Next 80B 已經能打 Claude Sonnet 4.5 的 SWE-bench Pro 分數。

**重點：**
- 三大模型各有強項：Claude 擅長大 codebase、GPT-5.3-Codex 擅長 terminal 自動化、Gemini 性價比最高
- 聰明的做法是 model routing，不是死守一家——根據任務複雜度選模型
- 但是... benchmark 收斂也意味著差異化越來越靠生態系和價格戰，而不是模型能力本身

### [Walmart's AI Checkout Converted 3x Worse. The Interface Is Why.](https://dev.to/kuro_agent/walmarts-ai-checkout-converted-3x-worse-the-interface-is-why-44o0)

這篇是今天最該讀的 UX 分析。Walmart 把 20 萬商品放上 ChatGPT 的 Instant Checkout——使用者可以在聊天視窗裡直接瀏覽和購買，零摩擦、超流暢。結果？in-chat 購買的轉換率只有跳轉到 Walmart 官網的**三分之一**。OpenAI 最後直接砍掉了這個功能。

文章引用了三個獨立研究畫出同一個模式：METR 的研究發現開發者用 AI 工具實際上慢了 19%，但**自覺**快了 20%；Wharton 的研究顯示 80% 的人會照著 AI 的錯誤答案走，而且越錯越有信心。作者提出了「load-bearing friction」（承重摩擦力）的概念——那些看起來是「摩擦」的東西，其實是支撐認知決策的結構。Walmart 網站上的商品格狀排列、信任徽章、購物車狀態，看起來是 clutter，其實是幫你做決策的 cognitive scaffolding。把它們拿掉，介面更乾淨了，但決策品質塌了。

對所有在做 AI-powered UI 的前端工程師來說，這是一記警鐘：不是所有摩擦都該被消除。

**重點：**
- 感覺好 ≠ 結果好——三個獨立研究都證實「體驗」和「成效」可以完全脫鉤
- 設計 AI 介面時問自己：我拿掉的摩擦是不是在承重？
- 但是... Walmart 沒有放棄 ChatGPT，而是改成在裡面嵌入自家結構化的購物體驗——混合模式才是正解

---

## ⚡ 一句話帶過

- **[Profiling Puppeteer Memory Usage in Node.js](https://dev.to/dennis-ddev/profiling-puppeteer-memory-usage-in-nodejs-5a88)** — Puppeteer 跑久了 OOM？別猜，用 heap snapshot 找真正的洩漏點，這篇手把手教你
- **[10 Cool CodePen Demos (March 2026)](https://dev.to/alvaromontoro/10-cool-codepen-demos-march-2026-2gci)** — 用 `appearance: base-select` 做出來的 F1 車手選單，好看到不像原生 HTML select
- **[Efficient Real-Time Flight Tracking in Browsers: Framework-Free](https://dev.to/maxgeris/efficient-real-time-flight-tracking-in-browsers-framework-free-cross-platform-solution-35ha)** — 在瀏覽器裡渲染 10,000+ 架飛機的 3D 地球，不用任何框架——Web API 的肌肉展示
- **[Anthropic 不再允許 Claude Code 訂閱用於 OpenClaw](https://news.ycombinator.com/item?id=47633396)** — 第三方 harness 要另外付費了，Claude Code 生態系的圍牆又高了一點
- **[Claude Code Plugin: Block Compromised Packages](https://dev.to/hammadtariq/i-built-a-claude-code-plugin-that-blocks-compromised-packages-before-installation-1o3l)** — 上週 axios@1.14.1 被劫持，這插件在 npm install 前擋下已知的惡意套件
- **[Vercel: Observability Plus 取消基本費](https://vercel.com/changelog/no-base-fee-for-observability-plus)** — $10 月費砍了，改成純用量計費。Vercel 又在用定價策略圈人了
- **[Generic Undo Manager for React](https://dev.to/math-krish/generic-undo-manager-for-react-part-1-3dfh)** — 乾淨的 undo/redo 實作，用 command pattern 處理任意 state，比自己硬幹 history stack 優雅
- **[12 歲小孩寫了個 2KB 零依賴的 CASL 替代品](https://dev.to/creeperkid2014/im-12-and-i-built-a-2kb-0-dependency-alternative-to-casl-g4a)** — 用 one-pass linear scan 取代遞迴 graph-walking 做權限檢查，Socket 品質分 100/100。後生可畏
- **[The Frontend as an Intelligent Assistant](https://dev.to/rohith_kn/the-frontend-as-an-intelligent-assistant-14ab)** — 前端從「畫介面」變成「智慧助手」的演化論，概念對但缺實作細節
- **[Prisma vs Drizzle vs ZenStack: TypeScript ORM in 2026](https://dev.to/ymc9/a-fresh-look-at-the-orm-landscape-and-how-the-new-zenstack-stacks-up-against-prisma-and-drizzle-4p4l)** — TypeScript ORM 三國志更新版，ZenStack 想在 Prisma 之上加 access control layer

---

## 📚 慢慢啃

- **[Auth Strategies: The Right Tool for the Right Scenario](https://dev.to/opensite/auth-strategies-the-right-tool-for-the-right-scenario-4m51)** — Sessions、JWTs、OAuth、SAML、passkeys、magic links 全部攤開比較，不站隊只講適用場景。值得存書籤
- **[Building Structured Product Comparisons with Next.js and AI](https://dev.to/reviewiq/building-structured-product-comparisons-with-nextjs-and-ai-3kpg)** — 用 Next.js + AI 做結構化商品比較頁，月處理 50K+ 的 "X vs Y" 搜尋，SEO 和架構細節都有
- **[You Test Your Code. Why Aren't You Testing Your AI Instructions?](https://dev.to/lukasmetzler/you-test-your-code-why-arent-you-testing-your-ai-instructions-4j2p)** — CLAUDE.md、copilot-instructions.md 這些 AI 指令檔也該有測試——這觀點不新但做法很實在
- **[O1 vs O3-mini vs O4-mini: Code Review Comparison](https://dev.to/rahulxsingh/o1-vs-o3-mini-vs-o4-mini-code-review-comparison-a7p)** — 三個 OpenAI reasoning model 做 code review 的實測比較，看完你會知道什麼時候該用哪個
