---
title: "CSS 跑 DOOM、AI 寫的 RTL 全是 bug、Node.js 斷路器實戰"
date: 2026-03-29
description: "有人用純 CSS 渲染了 DOOM，AI coding agent 生成的 RTL 介面全是坑，還有 Node.js Circuit Breaker 的生產級指南。今天前端圈很精彩。"
tags: [frontend, css, nodejs, ai, web-platform]
---

今天值得聊的就三件事。有人用 CSS transform 把 DOOM 搬進瀏覽器了（對，每個牆壁都是 `<div>`），AI 生成的 RTL 介面被抓出六大類 bug，然後有篇 Node.js 斷路器的文章寫得比大多數公司的 runbook 還詳細。

---

## 🔧 今日硬菜

### [CSS is DOOMed](https://nielsleenheer.com/articles/2026/css-is-doomed-rendering-doom-in-3d-with-css/)

Niels Leenheer 做了一件所有前端工程師想過但沒人真的去做的事：用純 CSS 渲染 DOOM。每面牆、每個地板、每個 imp 都是一個 `<div>`，透過 CSS 3D transform 定位在三維空間裡。JavaScript 負責遊戲邏輯，但所有的渲染——包括三角函數計算——全部交給 CSS。他用了 `hypot()` 算牆寬、`atan2()` 算旋轉角度，甚至用 paused animation + negative delay 的 type grinding hack 來做 CSS-only 的 frustum culling。這不是玩具 demo，這是對現代 CSS 能力的一次壓力測試。

**重點：**
- 牆壁座標用 CSS custom properties 傳入，寬度和角度全由 CSS `hypot()` / `atan2()` 即時計算
- 實驗性的純 CSS culling 用 paused animation trick 繞過了 CSS 還沒有 `if()` 的限制
- 但是... 撞到一堆瀏覽器 bug——Safari 的 View Transitions 會壓平 `preserve-3d`，Chrome 的 compositor 在大量 3D 元素下不穩定，`var()` 引用 `background-image` 會觸發瘋狂的 re-rasterization

### [AI Coding Agents Are Great, but They Suck at RTL. Here's How I Fixed It](https://dev.to/idanlevi1/ai-coding-agents-are-great-but-they-suck-at-rtl-heres-how-i-fixed-it-2g0g)

做 Hebrew 產品的工程師終於受不了了。每次叫 AI 生成元件，出來的 CSS 全是 `margin-left` 而不是 `margin-inline-start`、Tailwind 用 `ml-4` 而不是 `ms-4`、數字在 RTL 排版裡亂跳、圖示方向反了。他整理出六大類 AI 在 RTL 上的系統性錯誤，然後做了 RTLify——一個 CLI 工具，一行 `npx rtlify-ai init` 就能在你的 AI editor 裡注入一份 RTL 規則 markdown。不是 runtime dependency，就是一份規則檔讓 AI 讀。踩過 CSS logical properties 坑的人都知道這有多痛。

**重點：**
- 六大 RTL bug 模式：physical CSS、Tailwind 方向類別、bidi 文字（`<bdi>` tag）、icon 翻轉、currency/date 格式、React Native style
- 工具原理很簡單：寫一份 `.rtlify-rules.md` + 在 editor config 加指標，AI 每次對話都會讀
- 但是... 這本質上是在幫 AI 補訓練資料的偏差——LTR 語料遠多於 RTL，工具治標不治本，但至少你的 PR 不會再被 QA 打回來

### [Node.js Circuit Breaker Pattern in Production: Prevent Cascading Failures with Opossum](https://dev.to/axiom_agent/nodejs-circuit-breaker-pattern-in-production-prevent-cascading-failures-with-opossum-odg)

凌晨三點，payment service 開始 timeout，每個 request 掛 30 秒然後失敗，promise queue 爆了，connection pool 耗盡，整個站跟著掛。這篇用 Opossum 從頭講了 Circuit Breaker 的三態（closed/open/half-open）、fallback 策略設計原則、Prometheus metrics 整合、Grafana alert 配置，甚至包含 Bulkhead pattern 的 capacity 設計。是一篇你可以直接拿去改的生產級指南。

**重點：**
- `timeout` 必須小於你的 HTTP server request timeout，否則 circuit 永遠不會 trip
- Fallback 設計四原則：不要 throw、標記 stale、每次 fallback 都要 log、保持便宜
- 但是... 文章最後承認是 AI agent 寫的。內容品質不錯，但如果你在意這個，現在你知道了

---

## ⚡ 一句話帶過

- **[Stop Writing postMessage Manually For Workers — I Built a Decorator for That](https://dev.to/yashwant_kumarnt_b9b12b/stop-writing-postmessage-manually-for-workers-i-built-a-decorator-for-that-4h1d)** — Web Workers 的 `postMessage` 寫到崩潰？decorator pattern 包一層，Angular 和 React 都能用
- **[fixing two bugs stacked on top of each other in ProseMirror](https://dev.to/nalalou/why-bold-bleeds-when-you-join-blocks-in-prosemirror-1ob7)** — ProseMirror 裡 bold mark 在 block join 時會「漏出去」，debug 過程比 bug 本身精彩
- **[How We Cut Our AI API Bill by 78%](https://dev.to/ashu_578bf1ca5f6b3c112df8/how-we-built-a-context-engine-that-makes-ai-code-assistants-see-your-entire-codebase-3076)** — 自建 context engine 替代 Cursor 預設的 cosine similarity，token 用量砍七成
- **[Your AI Doesn't Need Screenshots. It Needs DevTools.](https://dev.to/emad_omar_5311e0e328be24c/your-ai-doesnt-need-screenshots-it-needs-devtools-3lg1)** — AI debug 網頁還在截圖？console、Network tab、response payload 才是正解
- **[Every MCP Browser Tool Uses Chromium. That's a Problem.](https://dev.to/achiya-automation/every-mcp-browser-tool-uses-chromium-thats-a-problem-4kpp)** — 13 個 MCP browser server，全部綁 Chromium。瀏覽器的多樣性在 AI 工具鏈裡直接歸零
- **[I got tired of rebuilding multi-step form logic from scratch, so I built FormFlow](https://dev.to/shngffrddev/i-got-tired-of-rebuilding-multi-step-form-logic-from-scratch-so-i-built-formflow-1mo)** — 第五次寫 `useReducer` + `localStorage` 的 multi-step form 後，終於抽成 library 了
- **[Code Mode: Batching MCP Tool Calls in a WASM Sandbox to Cut LLM Token Usage by 30-80%](https://dev.to/chrisremo85/code-mode-batching-mcp-tool-calls-in-a-wasm-sandbox-to-cut-llm-token-usage-by-30-80-18g7)** — 用 WASM sandbox 把多個 MCP tool call 打包成一次，省 token 的思路不錯
- **[Candy Logger v2 is here — a browser logger with a real UI](https://dev.to/shehari007/candy-logger-v2-is-here-a-browser-logger-with-a-real-ui-bl2)** — 瀏覽器 logger 終於不只是 `console.log` 了，浮動面板 + 結構化 table view
- **[AI Answers Can Come with Silent Tech Debt](https://dev.to/gtzilla/ai-answers-can-come-with-silent-tech-debt-4ibd)** — vibe coding 的隱性成本：AI 幫你做了選擇，但你不知道它選了什麼
- **[Why I Chose Remotion + FFmpeg for Server-Side Video Rendering](https://dev.to/dwelvin_morgan_38be4ff3ba/why-i-chose-remotion-ffmpeg-for-server-side-video-rendering-4c1g)** — 用 React 寫影片模板、server-side 渲染，Remotion 的正確打開方式

---

## 📚 慢慢啃

- **[The Mistakes Didn't Change. The Speed Did.](https://dev.to/felixortizdev/the-mistakes-didnt-change-the-speed-did-13i8)** — AI agent 生成的程式碼裡，大多數 PR 至少有一個漏洞。問題不是新的，但速度是。值得所有在用 AI 寫 code 的人停下來想一想
- **[Building a single-pass rich-text DSL parser (no regex)](https://dev.to/hoshinoyumeka/building-a-single-pass-rich-text-dsl-parser-no-regex-5bge)** — 受不了 Markdown 的語法衝突，從頭設計一套 DSL parser。single-pass、no regex、可巢狀，parser 設計的好教材
- **[Technical Debt: When to Fix, When to Ship](https://dev.to/juststevemcd/technical-debt-when-to-fix-when-to-ship-20pn)** — 不是「要不要還債」而是「你有沒有搞懂你欠的是什麼」。把 tech debt 當決策問題而不是道德問題的思路很清醒
- **[8 JavaScript Mistakes I See in Every Code Review (And How to Fix Them)](https://dev.to/lucasmdevdev/8-javascript-mistakes-i-see-in-every-code-review-and-how-to-fix-them-1dla)** — `==` vs `===`、for-in 遍歷 array、忘記 cleanup effect... 基礎但你團隊裡一定有人在犯
