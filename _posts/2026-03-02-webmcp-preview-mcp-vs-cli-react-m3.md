---
title: "WebMCP 搶先體驗、MCP 該不該死、React 穿上 Material 3 新衣"
date: 2026-03-02
description: "Chrome 推出 WebMCP 讓網站變 agent-ready，一篇 216 讚 HN 文章宣判 MCP 死刑說 CLI 才是王道，Material 3 Expressive 終於有人包成 React 元件了"
tags: [frontend, react, web-platform, ai, tooling]
---

今天前端圈被 MCP 這三個字母洗版了。Chrome 團隊發了 WebMCP 搶先體驗，要讓你的網站直接跟 AI agent 對話；同一天 Hacker News 上一篇「MCP 該死，CLI 萬歲」的文章衝到 216 讚。兩邊打起來了，我們看戲就好。另外有人終於把 Material 3 Expressive 包成 React 元件庫，這件事等很久了。

---

## 🔧 今日硬菜

### [WebMCP is available for early preview](https://developer.chrome.com/blog/webmcp-epp)

Chrome DevRel 正式推出 WebMCP 的早期預覽。核心概念很簡單：讓網站透過標準化的方式向 AI agent 暴露「可以做什麼」。分成兩條路——Declarative API 直接用 HTML form 定義動作，Imperative API 則走 JavaScript 處理複雜互動。

這東西解決的是 AI agent 跟網頁互動時的「盲人摸象」問題。現在的 agent 要操作網站，基本上就是在 DOM 裡亂摸，猜按鈕在哪、表單怎麼填。WebMCP 等於幫網站掛了一塊招牌：「我能幫你訂機票，參數長這樣。」

**重點：**
- 兩層 API 設計：Declarative（HTML form）處理標準動作，Imperative（JS）處理複雜流程
- 應用場景瞄準客服工單、電商結帳、旅遊訂票這些高價值互動
- 但是... 目前只是 early preview，要加入 Chrome 的 EPP 才能玩，離正式標準還有很長的路

### [When does MCP make sense vs CLI?](https://ejholmes.github.io/2026/02/28/mcp-is-dead-long-live-the-cli.html)

這篇在 HN 拿了 216 讚、150 則留言，標題直接喊「MCP is dead, long live the CLI」。作者的論點很明確：LLM 本來就很會用 CLI，它們是吃 man page 和 Stack Overflow 長大的，根本不需要另一層協議來「幫忙」。

踩過 MCP 坑的人讀這篇會一直點頭。初始化會掛、認證要重複做、權限只有全有全無——這些都是真實的日常痛點。而 CLI 天生就能 pipe、能 compose、能用同一套 auth。作者舉了一個 Terraform plan 的例子：用 CLI + jq 幾行搞定，用 MCP 你要嘛把整個 plan 塞進 context window（燒錢），要嘛自己寫 filter（更累）。

**重點：**
- LLM 用 CLI 的能力已經很強，額外的協議層增加了複雜度卻沒帶來對等的價值
- CLI 的可組合性（pipe, jq, grep）是 MCP 目前無法匹敵的殺手級優勢
- 但是... 作者自己也承認，沒有 CLI 替代品的場景下 MCP 還是有存在意義——問題是大多數場景不是這種

### [I built a React library for Material 3 "Expressive" (with motion)](https://dev.to/prudhvi_raj/i-built-a-react-library-for-material-3-expressive-with-motion-demos-docs-mb1)

Google 的 Material 3 Expressive 方向喊了一段時間，但 React 生態一直缺個能直接用的封裝。這位開發者把 Google 的 Material Web Components 包了一層 React wrapper，加上了 Expressive 風格的樣式和動畫，還附了 Storybook demo 和完整 API 文件。

老實說，Web Components 和 React 之間的橋接一直是個痛點——事件系統不同、SSR 要特別處理、型別要自己補。這個庫試圖把這些髒活包掉，讓你用起來像一般的 React 元件。有 Storybook 和 docs-first 的開發流程，起碼態度是對的。

**重點：**
- 底層是 Google Material Web Components，上層包成 typed React props/events
- 額外加了 React-first 的 date/time picker，不只是 1:1 wrapper
- 但是... 非 Google 官方、SSR 需要 client-only boundary、生態採用度還是未知數

---

## ⚡ 一句話帶過

- **[栗子前端技术周刊第 118 期 - Oxfmt Beta、Angular GitHub stars、React 基金会](https://juejin.cn/post/7611820139810848822)** — Oxfmt 號稱比 Prettier 快 30 倍、Angular 破 10 萬星、React 基金會正式成立，三件大事一次打包
- **[权限陷阱：为什么你的"点击复制"在某些浏览器或 iframe 里会失效？](https://juejin.cn/post/7611851387791179814)** — localhost 複製好好的，上線就啞火？Clipboard API 的 Secure Context 和 Permissions Policy 兩個坑，踩過的都懂
- **[Clipboard API 深度实战：如何同时存入纯文本和富文本两种格式？](https://juejin.cn/post/7611851387791163430)** — ClipboardItem 搭配多種 MIME type，一次複製、到處適配，該學的現代 API
- **[Clean Architecture in the Age of AI: Preventing Architectural Liquefaction](https://dev.to/uxter/clean-architecture-in-the-age-of-ai-preventing-architectural-liquefaction-5d8d)** — AI 生成的 code 在局部很好，但架構邊界正在被慢慢溶解，「Architectural Liquefaction」這個比喻精準到位
- **[Vercel Rejects Deploys from AI Sub-Agents. Here's Why — and the Fix.](https://dev.to/agent_paaru/vercel-rejects-deploys-from-ai-sub-agents-heres-why-and-the-fix-272c)** — Vercel 會驗證 git commit author，AI sub-agent 的 commit 直接被無聲拒絕，坑踩得很有代表性
- **[10 Cool CodePen Demos (February 2026)](https://dev.to/alvaromontoro/10-cool-codepen-demos-february-2026-59nf)** — 二月份最酷的 CodePen 作品精選，純 CSS 的創意永遠看不膩
- **[Add visual regression testing to your CI/CD without managing infrastructure](https://dev.to/custodiaadmin/add-visual-regression-testing-to-your-cicd-without-managing-infrastructure-a9m)** — Safari 上按鈕跑版、三個客戶棄單才發現——visual regression testing 該排進 pipeline 了
- **[January in Servo: preloads, better forms, details styling, and more](https://servo.org/blog/2026/02/28/january-in-servo/)** — Servo 瀏覽器引擎一月進度：resource preload、form 改善、details 元素樣式支援，穩步前進中
- **[I Built a Simulated Kernel Driven Operating System in the Browser](https://dev.to/mukund_149/i-built-a-simulated-kernel-driven-operating-system-in-the-browser-2d0k)** — 不是又一個可以拖視窗的「Web OS」，而是真的有 kernel 架構和 process queue 的瀏覽器 OS 模擬
- **[React Native VS Flutter: Which is future-proof & Best?](https://dev.to/techrajeshnandi/react-native-vs-flutter-which-is-future-proof-best-1ca6)** — 又是這個萬年話題，不過這次作者用數據和實際案例來論述，Flutter 粉可能不太開心

---

## 📚 慢慢啃

- **[从 0 到 1 实现一个 useState](https://juejin.cn/post/7611861210603028486)** — 從零手寫 useState，拆解資料持久化和觸發重新渲染的核心機制，想搞懂 React hooks 底層的可以花個下午讀
- **[Building a Production-Grade Table Editor with React and XState](https://dev.to/keyurparalkar/building-a-production-grade-table-editor-with-react-and-xstate-adding-rows-columns-efb)** — 用 XState 狀態機驅動 table editor，schema-driven 架構讓新增行列變得可預測，適合想認真學 state machine 在 UI 中應用的人
- **[From Static Timeline to Fully Interactive Scheduler: Drag & Drop in My React Native Library](https://dev.to/kozerkarol/from-static-timeline-to-fully-interactive-scheduler-drag-drop-in-my-react-native-library-4jkl)** — React Native 時間軸排程元件加入 drag & drop 和 resize，做過行事曆類 app 的知道這有多難
- **[Coding Agents Are Actually Good at This One Thing](https://dev.to/mattstratton/coding-agents-are-actually-good-at-this-one-thing-5dej)** — 不吹不黑地聊 AI coding agent 真正擅長的事，比起「取代工程師」的恐嚇文有營養多了
