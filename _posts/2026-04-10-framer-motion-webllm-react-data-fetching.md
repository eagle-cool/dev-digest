---
title: "踢掉 Framer Motion 省 27% bundle、瀏覽器直接跑 LLM、React 資料抓取進化論"
date: 2026-04-10
description: "一個 Next.js 網站把 Framer Motion 換成 CSS transition 省了 27% bundle，WebLLM 讓你在瀏覽器跑 Llama-3 完全離線，還有 React 資料抓取從 class component 到 Server Component 的完整演進史。"
tags: [frontend, react, nextjs, css, ai, nodejs]
---

今天的重點就三件事：有人把 Framer Motion 從 Next.js 專案踢掉，bundle 直接少了 27%（而且網站看起來一模一樣）；WebLLM 讓你在瀏覽器裡跑 Llama-3，一個 byte 都不用送出去；然後有篇文章把 React 資料抓取從 2013 年講到現在，看完你會覺得自己老了。

---

## 🔧 今日硬菜

### [I Evicted Framer Motion From a Client Site and Cut the Bundle by 27%](https://dev.to/sumorai/i-evicted-framer-motion-from-a-client-site-and-cut-the-bundle-by-27-2mep)

一個南佛州的小型 Web Agency 在做 Next.js 客戶網站的效能審計時，發現行動端 Lighthouse 分數卡在 70 幾分。打開 bundle analyzer 一看——Framer Motion 是整個專案最大的 dependency，40KB+ 的 JavaScript。用在哪？六個動畫：幾個 fade-in、一個 mobile nav 滑出、幾個 hover 效果。

他的做法簡單粗暴：全部換成 CSS `transition` + `IntersectionObserver`。fade-in 用 `.visible` class 切換 opacity 和 transform，mobile nav 用 `translateX` 搞定，hover 效果——拜託，CSS `:hover` 從 2010 年就有了。結果？bundle 小 27%，行動端 Lighthouse 從 72 跳到 89，TBT 少了 180ms，網站看起來完全一樣。

踩過這個坑的都知道：Framer Motion 的問題不只是 bundle size，而是它在每個 animation frame 都跑 JavaScript，吃的是 main thread。CSS transition 走的是 compositor thread，在 GPU 上跑，零主線程開銷。

**重點：**
- CSS `transition` + `IntersectionObserver` 能取代大部分商業網站的動畫需求，零 JS 執行成本
- Framer Motion 真正不可取代的是 `layout` 動畫、`AnimatePresence` exit 動畫、和複雜的拖拉手勢
- 但是——如果你的網站只需要 fade-in 和 hover，你正在用賽車引擎驅動一台腳踏車

### [Private AI is Here: Building a 100% Offline Mental Health Tracker with WebLLM and React](https://dev.to/beck_moulton/private-ai-is-here-building-a-100-offline-mental-health-tracker-with-webllm-and-react-fp9)

這篇示範了一個完全離線的 AI 日記應用：用 WebLLM 在瀏覽器裡跑 Llama-3-8B，透過 WebGPU 硬體加速，資料存 IndexedDB，一個 byte 都不送到伺服器。架構是 React + WebLLM engine + TVM.js，模型權重第一次下載後快取在 Cache API 裡，之後秒開。

技術上最有意思的是 prompting 部分——他讓模型扮演 CBT 治療師，對日記做結構化分析（情緒分數、認知扭曲、反思建議），用 `response_format: { type: "json_object" }` 強制輸出 JSON。整個架構完全可以套用到任何需要隱私的前端 AI 場景。

**重點：**
- WebLLM + WebGPU 讓瀏覽器變成完整的 AI 推理引擎，Llama-3-8B 在本地跑得動
- 模型權重用 Cache API 快取，第二次載入幾乎即時
- 但是——8B 模型的下載量不小，低階手機的 WebGPU 支援和記憶體都是問題，離「production-ready」還有一段距離

### [The Senior Dev Approach to Data Fetching in React](https://dev.to/emann/the-senior-dev-approach-to-data-fetching-in-react-171g)

這篇把 React 資料抓取的演進史攤開來講：從最早的 class component + `componentDidMount` + Higher-Order Component 模式，到 2019 年 Hooks 出來後用 `useEffect` 簡化，再到 Tanner Linsley 覺得 `useEffect` 拿來做資料抓取根本是災難而創造了 React Query。

文章最精彩的地方是點出 `useEffect` 做資料抓取的致命問題：你以為只要加幾行 code，但實際上你得處理 loading state、error state、race condition、快取、cache invalidation、stale request、網路斷線 fallback⋯⋯加完之後你的「簡單」fetch 變成 50 行。React Query 把這些全部封裝好，這才是資深工程師的選擇。

**重點：**
- React 資料抓取經歷了 HOC → Hooks → React Query → Server Components 四個世代
- `useEffect` 做資料抓取不是不行，是你得自己重新發明 React Query 已經解決的所有問題
- 但是——每個時代的方案都有其適用場景，Server Components 也不是銀彈，CSR 場景依然需要 React Query

---

## ⚡ 一句話帶過

- **[Agentic Infrastructure](https://vercel.com/blog/agentic-infrastructure)** — Vercel 發了一篇長文講「基礎設施應該從應用程式本身推導出來」，看完你會發現他們想讓 AI agent 直接操作部署流程，野心不小
- **[前端架構演進：基於AST的常量模組自動化遷移實踐](https://juejin.cn/post/7626589093779472434)** — 用 AST codemod 把集中式常量導出遷移成模組化，這種髒活有人幫你寫好工具就是爽
- **[Deploying SvelteKit to Cloudflare Workers for Free](https://dev.to/iampavel/deploying-sveltekit-to-cloudflare-workers-for-free-354d)** — SvelteKit + CF Workers 免費方案的完整指南，SSR + API routes + edge caching 一次搞定
- **[Anthropic Accidentally Published 513K Lines of Claude Code Source on npm](https://dev.to/guyruvio/anthropic-accidentally-published-513k-lines-of-claude-code-source-on-npm-what-developers-need-to-16ha)** — 59.8MB 的 source map 混進 npm 包裡，1,906 個檔案的完整架構全曝光，這就是為什麼你要檢查 `.npmignore`
- **[Cypress AI Skills: Teaching Your AI Assistant to Write Better Tests](https://dev.to/raju_dandigam/cypress-ai-skills-teaching-your-ai-assistant-to-write-better-tests-1dib)** — 讓 AI 寫 Cypress 測試的正確姿勢：不是丟 prompt 就好，得教它你的 selector 策略和 page object 模式
- **[Angular & the "Disappearing this"](https://dev.to/agraczyk/angular-the-disappearing-this-why-arrow-functions-are-more-than-just-syntax-sugar-3925)** — 在 Angular 裡用箭頭函式不只是語法糖，是 `this` binding 的生死問題，基礎但值得重溫
- **[layercache: Stop Paying Redis Latency on Every Hot Read](https://dev.to/flyingsquirrel0419/layercache-stop-paying-redis-latency-on-every-hot-read-m8l)** — Node.js 的 L1/L2 快取層套件，in-process memory 擋在 Redis 前面，高 QPS 場景省掉大量 round-trip
- **[Things I Wish I Knew Before Building My First React Native App](https://dev.to/abdulmalik_muhammad/things-i-wish-i-knew-before-building-my-first-react-native-app-40a5)** — 「我已經會 React 了，能差多少？」差很多，Bridge 不是免費的，這篇省你一週踩坑
- **[BunnyCDN has been silently losing our production files for 15 months](https://old.reddit.com/r/webdev/comments/1sglytg/bunnycdn_has_been_silently_losing_our_production/)** — 生產環境的檔案無聲無息消失了 15 個月才發現，CDN 不是放上去就沒事了
- **[Instant 1.0, a backend for AI-coded apps](https://www.instantdb.com/essays/architecture)** — InstantDB 1.0 發布，定位是「給 AI 寫的 app 用的後端」，即時同步 + 權限控制，架構文寫得很紮實

---

## 📚 慢慢啃

- **[How JavaScript Really Executes Code: Execution Context and Scope Chain Explained](https://dev.to/malloc72p/how-javascript-really-executes-code-execution-context-and-scope-chain-explained-4c9f)** — 基於 ES5 規範從頭拆解 Execution Context、Variable Object、Scope Chain，讀完你會真正理解 hoisting 和 closure 為什麼是那樣運作的
- **[Docker for TypeScript Developers Building AI Agents in 2026](https://dev.to/raju_dandigam/docker-for-typescript-developers-building-ai-agents-in-2026-1k3l)** — 如果你是前端工程師開始碰 AI agent 開發，這篇幫你補上 Docker 容器化的基礎，從 Dockerfile 寫法到多服務編排都有涵蓋
- **[The Terminal Renaissance: Designing Beautiful TUIs in the Age of AI](https://dev.to/hyperb1iss/the-terminal-renaissance-designing-beautiful-tuis-in-the-age-of-ai-24do)** — Claude Code 每天貢獻 13.5 萬個 GitHub commit，69% 的開發者終端機常駐，TUI 設計正在文藝復興——這篇講的是為什麼以及怎麼做
- **[MCP Prompts and Resources: The Primitives You're Not Using](https://dev.to/aws-heroes/mcp-prompts-and-resources-the-primitives-youre-not-using-3oo1)** — MCP 不只有 Tools，還有 Prompts 和 Resources 兩個被忽略的原語——搞懂它們能讓你的 AI agent 少走很多彎路
