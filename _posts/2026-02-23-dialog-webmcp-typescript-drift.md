---
title: "dialog 全攻略、Google WebMCP 搶先看、AI 寫的 TypeScript 債怎麼還"
date: 2026-02-23
description: "HTML dialog 元素終於值得認真對待、Google 推 WebMCP 讓 AI agent 直接呼叫網頁功能、drift CLI 幫你抓出 AI 生成的 TypeScript 技術債"
tags: [frontend, web-platform, typescript, ai, tooling]
---

今天三道硬菜都跟「瀏覽器原生能力」有關。`<dialog>` 終於從「那個沒人用的 HTML 元素」翻身成正宮，Google 丟出 WebMCP 要讓 AI agent 別再傻傻地 parse DOM，然後有人做了個 CLI 專門抓 AI 幫你寫出來的 TypeScript 爛帳。週日的份量剛剛好。

---

## 🔧 今日硬菜

### [Stop Rebuilding Modals: A Deep Dive into the &lt;dialog&gt; Element](https://dev.to/anjab/stop-rebuilding-modals-a-deep-dive-into-the-element-gko)

寫了十年前端的人都經歷過 modal 地獄——自己刻 focus trap、手動鎖 scroll、跟 z-index 打仗、然後 accessibility 永遠在 backlog 最底層。`<dialog>` 這東西 2012 年就有了，但直到 2022–2023 Safari 跟 Firefox 才把 `showModal()`、`::backdrop`、`inert` 都實作到位。現在一個 `dialog.showModal()` 就搞定 focus trapping、背景 inert、backdrop、ESC 關閉、top-layer stacking 和 ARIA 語義。你以前手刻的那些，瀏覽器全包了。

**重點：**
- `showModal()` 才是正宮——它把 dialog 丟進 top layer，自動 trap focus、建立 backdrop、讓背景 `inert`。`show()` 只是讓它顯示，什麼都不管
- 動畫要注意：關閉狀態的 dialog 是 `display: none`，transition 不會觸發。得用 `@starting-style` 或下一幀才加 class
- 跟 Popover API 的分工很清楚：modal 用 `<dialog>`（阻斷式互動），輕量浮動 UI 用 `popover`（light dismiss、不 trap focus）
- 但如果你的 modal 其實是 drawer、side panel、或需要巢狀堆疊的複雜場景，框架的 Modal 可能還是比較順手

### [WebMCP: A Browser-Native Execution Model for AI Agents](https://dev.to/astrodevil/webmcp-a-browser-native-execution-model-for-ai-agents-125n)

Google 在 2 月 13 日丟出 WebMCP Early Preview，核心概念很直接：讓網頁透過 `navigator.modelContext.registerTool()` 註冊結構化的 JavaScript function，AI agent 就能直接呼叫，不用再傻傻地 parse DOM、模擬點擊。這解決了一個根本問題——現有的 AI agent 操作網頁全靠「看畫面猜按鈕在哪」，token 消耗大、延遲高、還超容易壞。WebMCP 讓 tool 在瀏覽器 session 裡面跑，直接繼承 cookie、登入狀態、same-origin 安全邊界。規格正在 W3C Web Machine Learning Community Group 底下發展。

**重點：**
- 兩種註冊方式：宣告式（HTML form 標註）和命令式（JS `registerTool`），後者可以做動態、有狀態的能力暴露
- 跟傳統 MCP 的差別：不走 JSON-RPC server，網頁本身就是 tool provider，執行在同一個 JS 環境
- 安全模型很明確：只有 `registerTool` 註冊的能力才會被 agent 看到，same-origin 限制，不能隨意爬 DOM
- 但還在 Early Preview 階段，只有 Chrome 實驗版能測。離全面可用還早，不過方向對了

### [drift: an open source CLI that detects silent technical debt in AI-generated TypeScript code](https://dev.to/eduardbar/drift-an-open-source-cli-that-detects-silent-technical-debt-in-ai-generated-typescript-code-4ll7)

這個打到痛點了。AI 幫你寫的 code 通常「會動」，但留下一堆 ESLint 不會抓、CI 不會擋的隱性債務：`catch(e) {}` 直接吞錯誤、import 了沒用的東西不清、function 50 行做五件事、到處塞 `any`。drift 用 ts-morph 做 AST 分析，8 條偵測規則，給每個檔案打 0–100 分。作者拿 shadcn/ui（人寫的）跑出 0 分，自己的 vibe code 跑出 40–60 分。差距赤裸裸。

**重點：**
- 8 條規則直指 AI 程式碼的通病：大檔案、大 function、console.log 殘留、unused imports、重複 function 名、`any` 濫用、空 catch、缺 return type
- 可以直接塞進 CI：`npx @eduardbar/drift scan ./src --min-score 60`，超過閾值就擋 merge
- 已知限制：CLI 工具裡的 `console.log` 會被誤判，設定檔排除功能還在 roadmap
- 零 config、零 runtime dependency、MIT license——先跑一次再說

---

## ⚡ 一句話帶過

- **[I Fixed 110 Failing E2E Tests in 2 Hours Without Writing a Single Line of Test Code](https://dev.to/nikolarss0n/i-fixed-110-failing-e2e-tests-in-2-hours-without-writing-a-single-line-of-test-code-2mfd)** — 用 AI 修 Playwright 測試而不是手刻，聽起來很香但前提是你得先搞懂為什麼壞
- **[Why Your Frontend Integration Tests Keep Failing Randomly](https://dev.to/dipuoec/why-your-frontend-integration-tests-keep-failing-randomly-and-what-to-do-about-it-46m8)** — Flaky test 十有八九是你測試依賴了外部狀態，不是玄學
- **[From Static Assets to Dynamic Synthesis: Mastering DALL-E 3 and Vercel AI SDK in Next.js](https://dev.to/programmingcentral/from-static-assets-to-dynamic-synthesis-mastering-dall-e-3-and-vercel-ai-sdk-in-nextjs-20je)** — Next.js + AI SDK 做即時圖片生成，Generative UI 的實戰入門
- **[Redis in NestJS: The RedisX Solution You Didn't Know You Needed](https://dev.to/sur-ser/redis-in-nestjs-the-redisx-solution-you-didnt-know-you-needed-1c7f)** — NestJS 的 Redis 全家桶套件：caching、locking、rate limiting 一包搞定
- **[React Router: Loaders, Actions & Form](https://dev.to/edriso/react-router-loaders-actions-form-2bbe)** — 還在用 useEffect + useState 抓資料的可以看看 loader pattern 怎麼把這件事變乾淨
- **[How to Make Any React App Multilingual - Static UI + Dynamic Data](https://dev.to/manjhss/how-to-make-any-react-app-multilingual-static-ui-dynamic-data-1d0p)** — 多語系不只是翻按鈕文字，API 回來的動態資料才是真正的坑
- **[How to Prevent Accidental Password Leaks in Your Node.js APIs](https://dev.to/freerave/how-to-prevent-accidental-password-leaks-in-your-nodejs-apis-24k7)** — 別再手動 `delete user.password` 了，用 schema 層面解決比較不會漏
- **[Stop Writing JSON Fixtures. Use a Mock Server Instead.](https://dev.to/dipuoec/stop-writing-json-fixtures-use-a-mock-server-instead-2oph)** — 那個 `/fixtures` 資料夾裡一半的 JSON 已經跟 API 對不上了，你知道的
- **[Dancing Pixels: Audio-Reactive 3D Web Experience with React Three Fiber](https://dev.to/up_min_sparcs/dancing-pixels-building-an-immersive-audio-reactive-3d-web-experience-with-react-three-fiber-2ln6)** — React Three Fiber 做音訊視覺化，技術選型很有參考價值
- **[How to Convert SVG to React Components: Complete Guide](https://dev.to/arenasbob2024cell/how-to-convert-svg-to-react-components-complete-guide-34jc)** — SVG 轉 React component 的正確姿勢，動態 props 控制顏色和大小才是重點

---

## 📚 慢慢啃

- **[Everything I've learned about .cursorrules after mass testing them](https://dev.to/nedcodes/everything-i-learned-about-cursorrules-after-mass-testing-them-for-2-months-31km)** — 花兩個月實測 Cursor rules 系統的各種 edge case，如果你還在用 `.cursorrules` 而不是新的 rule 格式，這篇值得讀
- **[Why Your "Clean Code" is Actually Unmaintainable Rubbish](https://dev.to/oyminirole/why-your-clean-code-is-actually-unmaintainable-rubbish-eoc)** — 讀程式碼的時間是寫的 7 倍以上，所以可讀性比你以為的重要得多。這篇從認知負擔的角度切入，不是又一篇 Clean Code 教條文
- **[Rust for WebAssembly: How I Built Near-Native Performance Web Apps](https://dev.to/realacjoshua/rust-for-webassembly-how-i-built-near-native-performance-web-apps-2f1g)** — 從踩坑到上線的 Rust + Wasm 實戰紀錄，那些 bundler 報錯和 wasm target 設定的細節才是你真正需要的
- **[Building a Lightning-Fast Data Platform: How We Tackled Core Web Vitals](https://dev.to/ladlablogger/building-a-lightning-fast-data-platform-how-we-tackled-core-web-vitals-on-a-heavy-content-site-3fgg)** — 資料密集型網站的效能最佳化實戰，動態內容和載入速度的拉扯永遠是前端最經典的戰場
