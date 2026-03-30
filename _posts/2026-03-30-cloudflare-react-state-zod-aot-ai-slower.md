---
title: "Cloudflare 偷看你的 React State、Zod AOT 快 60 倍、AI 讓你變慢 19%"
date: 2026-03-30
description: "Cloudflare Turnstile 逆向工程揭露它會讀取 React Router 內部狀態來驗證你不是機器人；Zod AOT 用 Vite plugin 在建置時編譯 schema 達到 60 倍加速；METR 研究指出 AI 工具讓開發者自認快 24% 但實際慢 19%。"
tags: [frontend, react, typescript, ai, tooling]
---

今天三道硬菜都很有料。有人逆向了 Cloudflare Turnstile 發現它會讀你的 `__reactRouterContext`（對，React Router 的內部狀態），有個 Vite plugin 讓 Zod 驗證快了 60 倍而且零程式碼改動，然後 METR 的研究告訴你——你以為 AI 讓你快了 24%，但計時器說你慢了 19%。週日早上的咖啡配這些剛剛好。

---

## 🔧 今日硬菜

### [ChatGPT Won't Let You Type Until Cloudflare Reads Your React State](https://www.buchodi.com/chatgpt-wont-let-you-type-until-cloudflare-reads-your-react-state-i-decrypted-the-program-that-does-it/)

有人解密了 377 個 Cloudflare Turnstile 程式，發現它不只是做瀏覽器指紋辨識——它會讀取 ChatGPT 的 React 應用內部狀態。`__reactRouterContext`、`loaderData`、`clientBootstrap`，這三個 React Router v6+ 的內部結構全被拿來當驗證依據。邏輯很直白：一個沒有真正執行 React SPA 的 headless browser，根本不會有這些屬性。這是把 bot 偵測從瀏覽器層拉到了應用層。

整個加密鏈其實沒那麼神秘——外層用 `p` token 做 XOR，內層的 key 就是一個 float literal 直接寫在 bytecode 裡。377 個樣本，每個都收集完全相同的 55 個屬性，搭配 28 個 VM opcodes 的自訂虛擬機。除了指紋之外，還有行為生物辨識（鍵盤時間、滑鼠軌跡、滾動模式）和 proof-of-work。

**重點：**
- Turnstile 會讀 React Router 內部狀態（`__reactRouterContext`、`loaderData`），確認你的瀏覽器真的跑了完整的 SPA
- 加密是 XOR，key 就在同一個 payload 裡——防的是隨便看看的人，不是認真分析的人
- 但是... 這代表 Cloudflare 對你跑的前端應用有相當程度的可見性，隱私邊界是政策決定，不是密碼學保證

### [Zod vs Typia vs AJV — I Built a Build Plugin That Makes Zod 60x Faster With Zero Code Changes](https://dev.to/wakita181009/zod-vs-typia-vs-ajv-i-built-a-vite-plugin-that-makes-zod-60x-faster-with-zero-code-changes-1poc)

Zod 好用是好用，但效能一直是它的罩門。這個 Zod AOT 的新版本加了 `autoDiscover` 模式——一個 Vite plugin 在建置時自動找到你的 Zod schema，編譯成最佳化的驗證器，零程式碼改動。`vite.config.ts` 加兩行就結束了。

技術上最精彩的是 Two-Phase Validation：對合法輸入，整個 schema 被壓成一條 boolean expression chain，零記憶體配置、零 error object。只有驗證失敗時才走完整的錯誤收集路徑。在 production 環境中 95%+ 的 request 都是合法的，所以 Fast Path 幾乎是唯一會跑的程式碼。benchmark 結果：大型物件驗證比 Zod v4 快 54 倍，Set/Map 快 16-22 倍，接近 Typia 的水準但保留完整的 Zod API。

**重點：**
- `autoDiscover` 模式：Vite plugin 自動偵測並 AOT 編譯所有 exported Zod schema，支援 tRPC、Hono、React Hook Form
- Two-Phase Validation：合法輸入走 Fast Path（單一 boolean chain），無效輸入才做完整錯誤收集
- 但是... 有 `.transform()`、`.refine()` 的 schema 無法完全 AOT 編譯，只能部分最佳化。遞迴結構也還是用 `z.lazy()` fallback，效能仍輸 Typia 1.9 倍

### [Developers Think AI Makes Them 24% Faster. It Actually Makes Them 19% Slower.](https://dev.to/ziva/developers-think-ai-makes-them-24-faster-it-actually-makes-them-19-slower-27h)

METR 研究的重點不是「AI 讓你變慢」——而是你根本無法判斷 AI 到底有沒有幫到你。開發者自認快了 24%，實際測量慢了 19%，中間有 43 個百分點的認知落差。時間都花在哪？review AI 的建議、debug AI 生成的程式碼、重新 prompt、在自己的思路和 AI 的建議之間來回切換。這些活動「感覺」很有生產力，但實際上是純開銷。

搭配 CodeRabbit 的資料更有意思：AI 生成的程式碼有 1.7 倍的 issue、1.75 倍的邏輯錯誤、8 倍的過度 I/O 操作。Stack Overflow 2025 調查也顯示，對 AI 準確性的信任度從 40% 降到 29%。開發者用得越多，信任度越低——但還是繼續用。

**重點：**
- AI 在你「不熟」的程式碼上最有價值（探索新 library、理解陌生 codebase），在你熟悉的 codebase 上幾乎是純負擔
- 最高 ROI 的用法是「理解」而非「生成」——「解釋這個函式」比「幫我寫這個函式」更值錢
- 但是... 這研究是針對「資深開發者在自己的 repo 上工作」這個特定場景，新手或不熟悉 codebase 的情境下結果可能不同

---

## ⚡ 一句話帶過

- **[Hands-On with GitHub Copilot's New 'Autopilot' Mode in VS Code](https://dev.to/ulisses99/hands-on-with-github-copilots-new-autopilot-mode-in-vs-code-1790)** — Copilot 終於有全自動 agent mode 了，Preview 階段，能不能打過 Claude Code 再說
- **[I Scanned 5 Popular JavaScript Sites for SEO Issues — Here's What I Found](https://dev.to/jsvisible/i-scanned-5-popular-javascript-sites-for-seo-issues-heres-what-i-found-22ob)** — react.dev、vercel.com、stripe.com/docs 全都有 SEO 問題，大公司也不例外
- **[Your Supabase RLS Is Probably Wrong: A Security Guide for Vibe Coders](https://dev.to/solobillions/your-supabase-rls-is-probably-wrong-a-security-guide-for-vibe-coders-3l4e)** — 掃了幾十個 vibe-coded app，80% 的 RLS 寫錯了。功能正常 ≠ 安全正常
- **[most AI-generated tests are worse than no tests](https://dev.to/nalalou/most-ai-generated-tests-are-worse-than-no-tests-10no)** — `expect(result).toBeDefined()` 不叫測試，叫自我安慰。12/12 pass 的假安全感最要命
- **[Neovim 0.12.0](https://github.com/neovim/neovim/releases/tag/v0.12.0)** — 大版本更新，HN 上 251 讚。Vim 黨的信仰又被續了一年
- **[bQuery.js — The jQuery for the Modern Web Platform](https://dev.to/josunlp/bqueryjs-the-jquery-for-the-modern-web-platform-10o7)** — 有人做了「現代版 jQuery」，模組化零建置。致敬經典，但 2026 年的市場空間值得觀察
- **[What Cursor's 8GB Storage Bloat Teaches Us About Claude Code's Clean Architecture](https://dev.to/gentic_news/what-cursors-8gb-storage-bloat-teaches-us-about-claude-codes-clean-architecture-5173)** — Cursor 本地儲存散落 8GB，Claude Code 就一個 `.jsonl`。架構選擇的蝴蝶效應
- **[Zustand Has a Free State Manager That Replaces Redux in 10 Lines](https://dev.to/0012303/zustand-has-a-free-state-manager-that-replaces-redux-in-10-lines-15mj)** — Redux 要 200 行 boilerplate 的事 Zustand 10 行搞定，這不是新聞但值得重複講
- **[10 Things You Can Do With Vercel MCP Server](https://dev.to/rupa_tiwari_dd308948d710f/10-things-you-can-do-with-vercel-mcp-server-1c9j)** — Vercel 官方 MCP server 讓 AI 工具直接操作你的 Vercel 帳號，OAuth 認證，不是自架 MCP
- **[Architecting a Decoupled Stack: Next.js 15 and Django REST API](https://dev.to/hubert_zonyra/architecting-a-decoupled-stack-nextjs-15-and-django-rest-api-20b7)** — Next.js 15 + Django 的前後端分離架構，適合需要 Python 後端但想要 React 前端的團隊

---

## 📚 慢慢啃

- **[Stop Fighting Zustand Context: Practical Store Scoping Patterns for React](https://dev.to/alexey79/stop-fighting-zustand-context-practical-store-scoping-patterns-for-react-3c71)** — Zustand 上手簡單，但 app 長大後 store scoping 會變成痛點。這篇深入講解怎麼用 Context 做 store 隔離，寫 Zustand 的人遲早會需要
- **[Angular Signals Have Changed Angular Forever — Here's the Complete Guide](https://dev.to/0012303/angular-signals-have-changed-angular-forever-heres-the-complete-guide-2dn4)** — Angular 終於有 fine-grained reactivity 了，Signal 取代 Zone.js 讓 change detection 變得可預測。不管你用不用 Angular，理解 Signals 的設計思路對 React/Vue 開發者也有啟發
- **[Deno Fresh Has a Free API — Zero-JS Web Framework with Island Hydration](https://dev.to/0012303/deno-fresh-has-a-free-api-zero-js-web-framework-with-island-hydration-16jm)** — 預設零 JS、Islands 架構只 hydrate 互動元件。如果你對 Astro 的 Islands 概念感興趣，這是 Deno 生態的對應方案
- **[The "Vibe Coding" Wall of Shame](https://crackr.dev/vibe-coding-failures)** — HN 上 92 讚的 vibe coding 失敗案例合集。笑完之後想想，這些坑你的團隊是不是也踩過
