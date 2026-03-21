---
title: "TypeScript 逆襲 WASM 快三倍、OpenCode 拿下 120K 星、六個 Hook 省掉八成 Token"
date: 2026-03-21
description: "OpenUI 把 Rust WASM parser 改寫成 TypeScript 反而快了 3 倍，開源 AI coding agent OpenCode 衝到 120K 星，還有人用六個 Hook 腳本把 Claude Code 的 token 用量砍掉 80%"
tags: [frontend, typescript, ai, tooling]
---

今天三件事值得好好聊。有人把 Rust WASM 改回 TypeScript 結果快了三倍（對，你沒看錯），一個開源 AI coding agent 悄悄長到 120K GitHub stars，然後有個工程師用六個 hook 腳本把 Claude Code 的 token 消耗砍了八成。前端圈永遠不缺打臉時刻。

---

## 🔧 今日硬菜

### [We rewrote our Rust WASM Parser in TypeScript – and it got 3x Faster](https://www.openui.com/blog/rust-wasm-parser)

OpenUI 團隊做了一件大家覺得「不可能更快」的事——把他們的 Rust WASM parser 整個改寫成 TypeScript。結果？每次呼叫快 2.2 到 4.6 倍。這不是什麼 micro-benchmark 灌水，是真實的 streaming LLM output parsing pipeline，六個階段的完整解析器。

關鍵在於 WASM boundary tax。每次跨越 JS-WASM 邊界，你付的是：字串複製進 WASM linear memory、Rust 序列化成 JSON、再複製回 JS heap、最後 V8 反序列化。Rust 本身跑得再快，這筆「過路費」都吃掉了。他們甚至試過 `serde-wasm-bindgen` 直接傳 JsValue——反而慢了 30%，因為逐欄位建構 JS 物件的邊界穿越次數比 JSON 一次性傳輸更多。

更狠的是他們順手修了 streaming 架構的 O(N²) 問題，用 statement-level incremental caching 把完成的語句快取起來，只重新解析最後一段未完成的 statement。

**重點：**
- WASM 不是萬靈丹——parsing 結構化文字成 JS 物件這種場景，序列化成本 > 運算成本
- `serde-wasm-bindgen` 的「直接傳遞」其實是更多次的微小邊界穿越，比 JSON round-trip 更慢
- 但是... WASM 在 compute-bound + minimal interop 場景（影像處理、加密、物理模擬）依然無可取代，別因為這篇就把所有 WASM 都拆了

### [OpenCode – The open source AI coding agent](https://opencode.ai/)

又一個 AI coding agent？先別翻白眼——這個已經 120K GitHub stars、800+ contributors、月活 500 萬開發者。OpenCode 主打完全開源、隱私優先（不存你的程式碼），支援 75+ LLM provider（包括本地模型），內建 LSP 整合讓 LLM 自動載入對應語言的 language server。

真正有意思的是它的定位：不綁定任何一家 AI 供應商。你可以用 GitHub Copilot 帳號登入、用 ChatGPT Plus/Pro 登入、或接任何 API key。Terminal、Desktop app、IDE extension 三種介面都有。它還推了個叫 Zen 的服務，提供經過 coding agent benchmark 驗證的模型選擇。

**重點：**
- LSP-enabled 是殺手功能——LLM 有了靜態分析資訊，程式碼品質會明顯不同
- 多 session 平行跑 + session 分享連結，團隊協作場景很實用
- 但是... 120K stars 的背後是大量行銷推動，實際 coding 能力跟 Claude Code / Cursor 的差距還需要時間驗證

### [Claude Code used 2.5M tokens on my project. I got it down to 425K with 6 hook scripts.](https://dev.to/cytostack/claude-code-used-25m-tokens-on-my-project-i-got-it-down-to-425k-with-6-hook-scripts-d40)

踩過 Claude Code token 用量爆炸的都知道這痛。這位老兄做了件有趣的事：先 log 了 132 個 session，發現 **71% 的檔案讀取都是重複的**。Claude 在同一個 session 裡讀了 `server.ts` 四次、`package.json` 三次——不是因為檔案改了，是因為它根本不記得自己讀過。

他的解法是六個 Node.js hook 腳本（叫 OpenWolf）：一個 `anatomy.md` 幫每個檔案寫一行描述 + token 估算，讓 Claude 看描述就能決定要不要真的開檔案；一個 `cerebrum.md` 記錄你的偏好和糾正；一個 `buglog.json` 記已修過的 bug。結果從 2.5M 降到 425K tokens，省了約 80%。

**重點：**
- AI agent 的「無記憶」問題比你想的嚴重——七成讀取是浪費
- 用 hook 在 Read/Write 前注入上下文，比塞一大堆 CLAUDE.md 更精準
- 但是... 作者自己承認 hook 可靠性還不穩定、cerebrum.md 的遵從率只有 85-90%，而且 80% 省量是單一大專案的數字，20 個專案平均是 65.8%

---

## ⚡ 一句話帶過

- **[How I built a Next.js app with 1,500+ localized routes and perfect Technical SEO](https://dev.to/hunterx13/how-i-built-a-nextjs-app-with-1500-localized-routes-and-perfect-technical-seo-3g5l)** — 1500 條 i18n 路由全靠 Next.js App Router 動態生成，SEO 滿分。想做多語系的可以抄作業
- **[Top 5 in Frontend and AI this week](https://dev.to/shrutikapoor08/top-5-in-frontend-and-ai-this-week-ai-is-making-us-dumber-useeffect-gets-banned-9ca)** — Modern CSS 的隨機數、anchor positioning 已經能取代整個 JS 函式庫了，還有 TanStack 新出的 Hotkeys 庫
- **[I built a 1.7kb VDOM-less framework, it went "viral", and Reddit banned me](https://dev.to/murillobrand/i-built-a-17kb-vdom-less-framework-it-went-viral-and-reddit-banned-me-5gmp)** — 1.7KB 的 fine-grained reactive framework Sigwork，沒有 Virtual DOM。又一個挑戰者，三年後見
- **[Promises, Batching, AbortController, and How the Web Actually Works](https://dev.to/jalajb/promises-batching-abortcontroller-and-how-the-web-actually-works-53hi)** — `.then()` 和 `.finally()` 的進階用法、React 的 batching 秘密、HTTP 心智模型，基礎功溫習
- **[ESLint + Prettier + Husky + lint-staged：建立自动化的高效前端工作流](https://juejin.cn/post/7619219565654065215)** — 掘金上一篇很完整的前端代碼規範自動化教學，團隊剛建好 repo 的直接拿去用
- **[The bespoke software revolution? I'm not buying it](https://world.hey.com/jason/the-bespoke-software-revolution-i-m-not-buying-it-4bfad9ec)** — Jason Fried 對「AI 讓每個人都能有客製軟體」的潑冷水。HN 83 分，留言區比文章精彩
- **[The SOLID Principles are Universal](https://dev.to/remojansen/the-solid-principles-are-universal-1c9m)** — InversifyJS 作者聊 SOLID 在 JS/TS 的適用性，觀點比一般八股文有深度
- **[I got tired of downloading Playwright artifacts from CI](https://dev.to/sentinelqa/i-got-tired-of-downloading-playwright-artifacts-from-ci-so-i-changed-the-workflow-6gf)** — 把 Playwright 的 CI 測試報告流程從「下載 artifacts 慢慢看」改成即時可查，有共鳴
- **[PrinceJS vs Hono vs Express vs Elysia: Benchmarking the Fastest Bun Frameworks](https://dev.to/thelittleprince1218/princejs-vs-hono-vs-express-vs-elysia-benchmarking-the-fastest-bun-frameworks-in-2025-c73)** — Bun 框架效能對決，一個 13 歲奈及利亞少年寫的框架 PrinceJS 居然跑進前三
- **[网络请求在 Vite 层的代理与 Mock](https://juejin.cn/post/7619219565654032447)** — Vite 的 proxy 和 mock 設定從入門到實戰，被 CORS 搞到懷疑人生的新手必讀

---

## 📚 慢慢啃

- **[Promises, Data Flow, and the memo + useCallback Combo That Actually Makes React Fast](https://dev.to/jalajb/promises-data-flow-and-the-memo-usecallback-combo-that-actually-makes-react-fast-13j4)** — 把 `memo` + `useCallback` 的組合拳講透了，不是教你用 API，是教你為什麼這個組合能阻止不必要的 re-render
- **[The Supabase Gotchas Nobody Warns You About](https://dev.to/cynthizo/the-supabase-gotchas-nobody-warns-you-about-until-you-hit-them-2a7g)** — Supabase DX 一流但坑也不少，這篇整理了實際踩過的雷，準備用 Supabase 開新專案的先讀
- **[How I Cut My AI Coding Agent's Token Usage by 120x With a Code Knowledge Graph](https://dev.to/deusdata/how-i-cut-my-ai-coding-agents-token-usage-by-120x-with-a-code-knowledge-graph-4a3d)** — 用 code knowledge graph 取代 AI agent 的逐檔搜尋，概念跟今天硬菜的 OpenWolf 異曲同工但走更激進的路線
- **[I built a SaaS in a week with Next.js, Supabase, and Stripe](https://dev.to/alberto_towns_b3e46310d1c/i-built-a-saas-in-a-week-with-nextjs-supabase-and-stripe-heres-what-i-learned-jbg)** — Next.js + Supabase + Stripe 的一週 SaaS 實戰記錄，對想 ship side project 的人是很實際的參考
