---
title: "CSS 動態選擇器黑科技、Edge.js 用 Wasm 沙箱跑 Node、Next.js 效能翻倍實戰"
date: 2026-03-18
description: "CSS sibling-index() 和 if() 讓 nth-child 變成動態變數驅動、Wasmer 推出 Edge.js 用 WebAssembly 沙箱跑 Node.js 且相容度屌打 Bun 和 Deno、Next.js 從 PageSpeed 55 衝到 94 的七個實戰優化"
tags: [frontend, css, nodejs, nextjs, web-platform, tooling, ai]
---

今天有三道硬菜值得坐下來好好看。CSS 又進化了——`sibling-index()` 加上 `if()` 讓你可以用變數控制 `:nth-child()` 的邏輯，這不是什麼花拳繡腿，是真正改變選擇器思維模式的東西。然後 Wasmer 推了個 Edge.js，號稱用 Wasm 沙箱跑 Node.js 還能 100% 相容，測試數據看起來確實猛。最後一篇 Next.js 效能優化實戰，從 55 分衝到 94 分，每一招都帶程式碼，直接抄作業。

---

## 🔧 今日硬菜

### [使用 sibling-index() 和 if() 实现动态的 :nth-child()](https://juejin.cn/post/7618060828449112105)

這篇文章讓我眼睛一亮。傳統 `:nth-child(An + B)` 最大的痛點是什麼？A 和 B 寫死在選擇器裡，想改就得改選擇器本身。現在 CSS 新特性 `sibling-index()` 可以拿到元素的索引，配合 `if()` 條件表達式和數學函數（`sign()`、`round()`），你可以把 A 和 B 變成 CSS 變數，用計算的方式判斷元素是否匹配。更猛的是，作者還用 `@function` 把整套邏輯封裝成自定義 CSS 函數 `--nth-child(A, B)`——是的，CSS 現在能寫函數了。

這不只是語法糖，這是把選擇器邏輯從「靜態匹配」升級成「可計算的變數」。意味著你可以用同一段 CSS，透過改變數就動態切換選擇規則，甚至讓匹配結果參與動畫、布局計算。踩過動態列表樣式那些坑的人都知道這有多香。

**重點：**
- `sibling-index()` 回傳元素在兄弟中的位置索引，搭配 `calc()` 可以重現 `An + B` 的數學判斷
- `@function` 自定義 CSS 函數讓你可以封裝複雜邏輯，呼叫時只需 `--nth-child(3, 2)`
- 但是...目前只有 Chrome 支援，Safari 和 Firefox 還沒跟上。生產環境要用還得等等

### [Edge.js: Run Node apps inside a WebAssembly sandbox](https://wasmer.io/posts/edgejs-safe-nodejs-using-wasm-sandbox)

Wasmer 放了個大招：Edge.js，一個用 WebAssembly 做沙箱的 Node.js 相容執行環境。跟 Deno 和 Cloudflare Workers 走 WinterCG 另起爐灶不同，Edge.js 的策略是「完全相容 Node.js v24，只把不安全的部分（系統呼叫、原生模組）關進 WASIX 沙箱」。JS 引擎本身跑原生，所以效能損耗只有 5-30%。

最殺的是那張相容性對照表：3626 個 Node.js 測試，Edge.js 通過 3592 個（99%），Bun 只過 1513 個（42%），Deno 過 1607 個（44%）。`node:http` 411 個測試全過、`node:fs` 342 個全過、`node:crypto` 150 個全過。這數字如果沒灌水的話，確實是目前 Node.js 相容度最高的替代方案。

架構上的巧思是用 NAPI 做 JS 引擎抽象層，所以理論上可以換 V8、JavaScriptCore 或 QuickJS。不綁定特定引擎這點，長期來看比 Bun 和 Deno 都更靈活。

**重點：**
- 用 `edge myscript.js` 就能跑，也能 `edge pnpm run dev` 包裝現有指令
- 支援 Next.js、Astro 等框架不需改 code，原生模組透過 NAPI 也能跑
- 但是...啟動速度目前比 Node.js 慢（缺少 JS snapshot），效能在 HTTP 密集場景差距較大，0.x 版本先求正確再求快

### [7 Next.js Performance Optimizations That Took Our Site from 55 to 94 on PageSpeed](https://dev.to/shota_wevosoft/7-nextjs-performance-optimizations-that-took-our-site-from-55-to-94-on-pagespeed-68i)

這篇沒有花俏的理論，就是一個團隊把 PageSpeed 從 55 分拉到 94 分的實戰記錄，每一招都附程式碼。最值得學的兩招：一是延遲載入 GTM——用 `setTimeout` 3 秒或等 scroll/click 再載入，LCP 直接省 800ms；二是 CSS 動畫害死 LCP 的隱形殺手——hero section 的文字用 `opacity: 0.08` 做淡入動畫，Lighthouse 就認為你的 LCP 元素是「不可見的」，直接把 LCP 算到動畫結束才算，5.6 秒。解法很暴力也很有效：LCP 元素不做動畫，其他隨便動。

另外 Next.js 預設的圖片快取 TTL 只有 60 秒這件事，不知道坑了多少人。改成 30 天 `minimumCacheTTL: 2592000`，回訪速度直接起飛。還有 ISR + on-demand revalidation 取代 `force-dynamic`，TTFB 從 400ms 降到 50ms。都是老生常談但很多人沒做到的事。

**重點：**
- GTM 延遲載入：3 秒 timeout 或 scroll/click 觸發，減少 130KB 未使用 JS
- LCP 元素不要做 opacity/filter 動畫，否則 Lighthouse 會等動畫完才算 LCP
- 但是...GTM 本身 126KB 未使用 JS 是個無解的痛，Google 自己的東西拖累自己的評分標準，挺諷刺的

---

## ⚡ 一句話帶過

- **[Stop Letting AI Write Untestable Code. Add Determinism Back with TWD](https://dev.to/kevinccbsg/stop-letting-ai-write-untestable-code-add-determinism-back-with-twd-3a02)** — AI 生程式碼速度不是瓶頸，「你敢不敢重構」才是，TWD（Test-Worthy Development）把確定性測試拉回主場
- **[TAILWIND CSS in serious apps](https://dev.to/moraa_oo_b141544e9be30688/tailwind-css-in-serious-apps-what-i-found-after-asking-ai-using-v0-and-digging-through-github-140o)** — 新手認真研究 Tailwind 在生產環境怎麼用，結論：別讓 AI 幫你生 50 個 CSS 變數，Tailwind 的 design token 就夠了
- **[WebP vs AVIF: Which Next-Gen Image Format Wins in 2026?](https://dev.to/pixotter/webp-vs-avif-which-next-gen-image-format-wins-in-2026-3m1d)** — AVIF 壓縮率贏 WebP 約 20%，但編碼慢五倍，CDN 沒快取的話反而更慢。答案還是老話：兩個都輸出，`<picture>` 讓瀏覽器自己選
- **[Updated GitHub App Permissions](https://vercel.com/changelog/vercel-github-app-updated-permissions)** — Vercel GitHub App 多要了 Actions 讀取和 Workflows 寫入權限，讓 Vercel Agent 能讀 CI log 幫你 debug，v0 也能直接改 workflow 檔案了
- **[I stopped babysitting Puppeteer](https://dev.to/boehner/i-stopped-babysitting-puppeteer-heres-what-i-use-instead-18kp)** — 每個 Node.js 專案最終都會需要截圖或抓 OG tags，然後就掉進 Puppeteer 的坑。這位仁兄改用 API 服務，省了維護 headless Chrome 的痛苦
- **[Get Shit Done: A Meta-Prompting, Context Engineering and Spec-Driven Dev System](https://github.com/gsd-build/get-shit-done)** — HN 153 分的 AI coding 框架，核心概念是先寫 spec 再讓 AI 實作，用 meta-prompt 控制品質。踩過 AI 亂飄的都懂這個痛點
- **[Announcing the Colab MCP Server](https://dev.to/googleai/announcing-the-colab-mcp-server-connect-any-ai-agent-to-google-colab-308o)** — Google 出了 Colab MCP Server，讓 Claude Code、Gemini CLI 這些 AI agent 可以直接在 Colab 上跑程式碼，不怕弄髒本機環境
- **[I Ran 23 AI Agents Simultaneously on One Codebase Overnight](https://dev.to/agent_paaru/i-ran-23-ai-agents-simultaneously-on-one-codebase-overnight-heres-what-happened-4ph4)** — 23 個 AI agent 同時改一個 Next.js 專案，一晚上從 28K 行變 56K 行、120 個 commit、零 TypeScript 錯誤。聽起來很瘋，但也很 2026
- **[Cypress Interceptors (E2E Testing)](https://dev.to/elabid_asmaa/cypress-interceptors-e2e-testing-1lc5)** — `cy.intercept()` 基礎教學，mock API、等待請求、驗證回應，E2E 測試的基本功整理得還算清楚
- **[built 140+ free browser tools in Next.js](https://dev.to/saif_ali_2c62caf75ac74565/built-140-free-browser-tools-in-nextjs-full-list-inside-mpf)** — 140 多個純前端工具（圖片壓縮、JSON formatter、色碼轉換⋯⋯），全跑在瀏覽器裡不上傳。Next.js 練手專案的天花板

---

## 📚 慢慢啃

- **[How to Structure Claude Code for Production: MCP Servers, Subagents, and CLAUDE.md](https://dev.to/lizechengnet/how-to-structure-claude-code-for-production-mcp-servers-subagents-and-claudemd-2026-guide-4gjn)** — 用 MCP Server + Subagent + CLAUDE.md 組織 AI 輔助開發工作流的完整指南，CodeRabbit 分析說這樣做 defect 少 1.7 倍、安全漏洞少 2.74 倍。值得花 20 分鐘讀完
- **[If you thought the code writing speed was your problem; you have bigger problems](https://andrewmurphy.io/blog/if-you-thought-the-speed-of-writing-code-was-your-problem-you-have-bigger-problems)** — HN 173 分的反思文。寫 code 的速度從來不是瓶頸，理解問題、設計方案、溝通對齊才是。AI 加速了打字，但沒加速思考
- **[Confident and Wrong](https://dev.to/maxrimue/confident-and-wrong-107o)** — 一位工程師分享 AI coding 從「助力」變「陷阱」的經驗。AI 信心滿滿地給你錯誤答案，你信了因為它看起來太對了。越資深越該看的警示文
- **[AI Coding Agents Need Enforcement Ladders, Not More Prompts](https://dev.to/douglasrw/ai-coding-agents-need-enforcement-ladders-not-more-prompts-5f04)** — 75% 的 AI coding model 在長期維護中會引入 regression。解法不是寫更好的 prompt，而是建立分層的品質把關機制。數據紮實的一篇
