---
title: "WebGPU 影片編輯器、Cloudflare 吞併 Pages、AI Agent 的 Optimistic UI"
date: 2026-03-22
description: "瀏覽器裡跑 WebGPU + WASM 的專業影片編輯器、Cloudflare 正式把 Pages 併入 Workers 的遷移實錄、以及怎麼用 Optimistic UI 讓你的 AI Agent 不再轉圈圈。"
tags: [frontend, web-platform, tooling, ai]
---

今天三道硬菜的共同主題：瀏覽器能做的事越來越誇張了。有人用 WebGPU 在瀏覽器裡搞了專業級影片剪輯、Cloudflare 確認 Pages 正式被 Workers 吸收（早說了吧），然後有篇文章認真聊了怎麼讓 AI Agent 的 UI 不要一直轉那個令人焦慮的 loading spinner。

---

## 🔧 今日硬菜

### [Professional video editing, right in the browser with WebGPU and WASM](https://tooscut.app/)

Tooscut 這個專案直接把專業影片編輯搬進瀏覽器，底層是 WebGPU 做 compositing、Rust 編譯成 WASM 跑運算。多軌時間軸、關鍵幀動畫、即時濾鏡（亮度、對比、飽和度、模糊）全部 GPU 運算，還支援 bezier easing curve。最狠的是用 File System Access API 做到完全本地運行——你的影片素材不會離開你的機器。

這種專案兩年前根本不可能做到。WebGPU 終於讓瀏覽器的 GPU 運算不再是玩具，配合 WASM 處理密集運算，Native app 的護城河又矮了一截。如果你還在覺得「Web 做不了重活」，是時候更新認知了。

**重點：**
- WebGPU + Rust/WASM 的組合拳，讓瀏覽器端的影片處理達到接近原生效能
- File System Access API 實現完全本地化，零伺服器依賴——隱私和離線都搞定了
- 但是... WebGPU 目前的瀏覽器支援率還沒到 100%，Safari 還在追趕，要做跨瀏覽器的話得準備 fallback

### [Cloudflare Pages vs Workers in 2026: Migration Guide](https://dev.to/rickcogley/cloudflare-pages-vs-workers-in-2026-migration-guide-ka7)

Cloudflare 的 Workers 技術負責人 Kenton Varda 講得很明白：「我們正在把 Pages 的功能全部搬進 Workers。」雖然官方沒說 deprecated，但訊號夠清楚了——Secrets Store、Workflows、Containers 全部是 Workers only。作者今年初把所有 Pages 專案遷移完畢，整理了完整的相容性矩陣和踩坑紀錄。

踩過 Pages 部署坑的人都知道，Pages 一直是那種「夠用但處處受限」的東西。Cron Triggers 不行、Queue Consumers 不行、Durable Objects 要繞路。現在 Cloudflare 等於承認了：Pages 就是 Workers 的子集，早該統一。如果你還有專案跑在 Pages 上，現在遷移是最好的時機——晚了只會更痛。

**重點：**
- Workers 現在能做 Pages 所有的事，還多了 Cron Triggers、Queue Consumers、Email Workers 等一堆 Pages 做不到的
- 遷移的主要痛點在 wrangler.toml 設定和 build 流程的差異，文章有詳細對照表
- 但是... 遷移不是按個按鈕的事，尤其是有用到 Pages Functions 的專案，需要重新理解 Workers 的 routing 模型

### [Stop Waiting: How to Build "Instant" AI Agents with Optimistic UI](https://dev.to/programmingcentral/stop-waiting-how-to-build-instant-ai-agents-with-optimistic-ui-3agp)

你花幾週打造了一個 LangGraph.js agent，結果使用者輸入完就盯著 loading spinner 發呆。這篇文章把 Optimistic UI 的老概念拿來解決 AI Agent 的延遲問題：不等 server 回應就先更新 UI，假設成功、失敗再 rollback。文章給了完整的 TypeScript 實作，包含狀態快照、progressive disclosure（讓使用者看到 agent 正在做什麼），以及 rollback 機制。

這概念本身不新——社群媒體的「按讚」就是這樣做的。但套在多步驟的 AI agent 上，複雜度直接翻好幾倍：你得管理一連串的中間狀態，處理並行 tool execution 的 UI 呈現，還得防止使用者連按觸發的 race condition。文章把這些 edge case 都講到了，算是把 Optimistic UI 從「知道」拉到「能用」的好教材。

**重點：**
- 核心模式：假設成功、即時更新 UI、用狀態快照做 rollback 安全網
- 對 LangGraph.js 的 multi-step 流程，用 SSE/WebSocket 推送中間狀態，讓 UI 顯示「正在搜尋...」「正在整合...」等進度
- 但是... 文章的範例偏概念展示，真正要上 production 還得處理 Vercel/Lambda timeout、schema validation（LLM 可能回傳亂七八糟的 JSON），這些他提到了但沒深入

---

## ⚡ 一句話帶過

- **[Vue3动态组件Component的深度解析与应用](https://juejin.cn/post/7619552802596159539)** — Vue 3 的 `<component :is>` 從基礎到 KeepAlive 快取策略，掘金出品，中規中矩但基本功紮實
- **[TypeScript Utility Types Complete Guide](https://dev.to/muhammadlutf1/typescript-utility-types-complete-guide-agl)** — Partial、Pick、Omit、Record 全家桶複習，適合傳給剛入坑 TypeScript 的同事
- **[The Review Gap](https://dev.to/thesythesis/the-review-gap-16ff)** — AI 產生的 PR 等待人類 review 的時間是人寫 PR 的 4.6 倍——瓶頸從寫 code 搬到了讀 code
- **[Cellular Automata Explorer in WebAssembly](https://dev.to/jsamwrites/i-built-a-cellular-automata-explorer-in-webassembly-here-are-21-visual-experiments-376o)** — 用 WASM 跑 cellular automata 的視覺實驗，21 天 21 個作品，技術含量不高但創意滿分
- **[OpenTelemetry just standardized LLM tracing](https://dev.to/vola-trebla/opentelemetry-just-standardized-llm-tracing-heres-what-it-actually-looks-like-in-code-2e5f)** — OTel 終於定義了 LLM 的 span 命名和 attribute 標準，不用再每家 vendor 一套格式了
- **[Push Notifications in React Native + Node.js](https://dev.to/dottt/push-notifications-in-react-native-nodejs-the-complete-setup-guide-part-1-2673)** — FCM 從零到 production 的 setup guide，React Native 開發者的必經之路
- **[How I Run a Team of AI Coding Agents in Parallel](https://dev.to/battyterm/how-i-run-a-team-of-ai-coding-agents-in-parallel-1bmn)** — 同時跑五個 Claude Code 的實戰心得，混亂但誠實
- **[No Semicolons Needed](https://terts.dev/blog/no-semicolons-needed/)** — 又有人在 HN 上吵分號了，23 則留言，經典永不退流行
- **[Stop Writing Environment Variable Validation From Scratch](https://dev.to/husainkorasawala/stop-writing-environment-variable-validation-from-scratch-po9)** — 用 Zod 或類似工具驗證 env vars 而不是手寫 if-throw，小技巧但省很多 debug 時間
- **[Why I Ditched Modern Frameworks for 100/100 Lighthouse](https://dev.to/barkodkarekod/why-i-ditched-modern-frameworks-for-a-high-performance-multilingual-utility-100100-lighthouse-4eoa)** — 用 Vanilla JS 拿到 Lighthouse 滿分，框架黨看了沉默、原生黨看了流淚

---

## 📚 慢慢啃

- **[Accessibility-First Software Engineering](https://dev.to/topuzas/accessibility-first-software-engineering-building-inclusive-systems-from-the-ground-up-38ol)** — 不是又一篇「加 alt text」的入門文，而是從架構層面談 accessibility context propagation 和多模態 API 設計，值得週末沉思
- **[5 Dangerous Lies Behind Viral AI Coding Demos](https://dev.to/deepak_mishra_35863517037/5-dangerous-lies-behind-viral-ai-coding-demos-that-break-in-production-160m)** — 拆解那些「五分鐘從零到一」的 AI coding demo 背後藏了什麼，看完對「AI 取代工程師」的焦慮會降低不少
- **[Stop Installing CLI Tools: 7 Data Format Conversions in the Browser](https://dev.to/damon_bb9e4bba1285afe2fcd/stop-installing-cli-tools-7-data-format-conversions-you-can-do-in-the-browser-37bj)** — IP 轉 hex、JSON escape、文字轉 CSV，全部在瀏覽器端完成不上傳——又一個「瀏覽器能做更多」的實證
- **[How to Write Effective Agent Skills for Claude](https://dev.to/alextdev/how-to-write-effective-agent-skills-for-claude-2opb)** — 如果你用 Claude Code 但每次都在重複下同樣的指令，這篇教你怎麼把常用流程封裝成可重用的 Agent Skill
