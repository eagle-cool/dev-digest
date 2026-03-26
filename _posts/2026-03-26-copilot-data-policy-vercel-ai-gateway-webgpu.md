---
title: "Copilot 要吃你的 code、Vercel AI 帳單透明化、WebGPU 上桌"
date: 2026-03-26
description: "GitHub Copilot 4 月起預設拿你的互動資料訓練模型（記得去關）、Vercel AI Gateway 推出跨 provider 統一帳單 API、WebGPU 與 WebAssembly 解鎖瀏覽器端硬體加速實戰"
tags: [frontend, ai, web-platform, css, tooling]
---

今天最值得聊的一件事：你寫的 code，GitHub 要拿去訓練 Copilot 了。4 月 24 日起，Free、Pro、Pro+ 用戶的互動資料預設開啟訓練，不想被吃的自己去 Settings 關。另外 Vercel 終於讓你看清楚 AI 帳單花在哪了，還有人用 WebGPU 在瀏覽器裡搞硬體加速——未來前端要會寫 shader 了嗎？

---

## 🔧 今日硬菜

### [Updates to GitHub Copilot interaction data usage policy](https://github.blog/news-insights/company-news/updates-to-github-copilot-interaction-data-usage-policy/)

GitHub 正式宣布：從 4 月 24 日起，Copilot Free、Pro、Pro+ 用戶的互動資料——包括你的輸入、輸出、code snippets、游標附近的上下文、檔案結構、甚至你對建議按讚或倒讚——通通預設拿去訓練模型。之前已經 opt out 的人不受影響，但新用戶和沒動過設定的人，你的 code 已經在訓練管道裡了。

說白了，這就是「免費的最貴」的經典案例。GitHub 先用微軟員工的資料訓練，看到 acceptance rate 提升了，現在要把網撒到所有個人用戶。Business 和 Enterprise 不受影響——因為企業客戶的律師不是吃素的。

**重點：**
- 4/24 起預設開啟，去 Settings → Copilot → Privacy 手動關掉
- 收集範圍廣到嚇人：code context、檔名、repo 結構、導航模式全都算在內
- 但是... 資料會共享給 Microsoft 旗下公司，雖然說不會給第三方 AI provider，但「affiliate」的定義有多寬？自己判斷

### [Unified reporting for all AI Gateway usage](https://vercel.com/blog/unified-reporting-for-your-ai-spend)

如果你在 Vercel 上跑 AI 功能，你大概經歷過這種痛苦：OpenAI 一個 dashboard、Anthropic 一個 dashboard、自帶 key 的又是另一個——月底帳單來了才知道燒了多少。Vercel AI Gateway 的新 Custom Reporting API（beta）終於把這些統一了。

這個 API 讓你用一個 endpoint 就能查跨 provider 的費用、token 用量和請求數。支援按 model、provider、user ID、自訂 tag 甚至 credential 類型來拆帳。對做 SaaS 的人來說，這意味著你終於能算出每個客戶、每個功能到底吃了多少 AI 成本。有個案例說靠這個省了 8 萬美金——信不信由你。

**重點：**
- 跨 AI SDK、Chat Completions API、Responses API、Anthropic Messages API 統一報表
- 支援 BYOK（Bring Your Own Key）場景的費用追蹤
- 但是... 目前還是 beta，而且你得先把流量都導到 Vercel AI Gateway 才有用——又多了一層 vendor lock-in

### [Supercharge Your Web Apps: Hardware Acceleration with WebGPU and WebAssembly](https://dev.to/programmingcentral/supercharge-your-web-apps-hardware-acceleration-with-webgpu-and-webassembly-259d)

這篇完整走了一遍 WebGPU + WebAssembly 的實戰流程：從為什麼 JavaScript 單執行緒搞不定 AI 推論，到用 Rust 編譯 WASM 做矩陣運算，再到用 ONNX Runtime Web 配合 WebGPU 在瀏覽器裡跑模型。踩過 WASM 記憶體管理坑的人都知道，這東西入門容易踩坑更容易。

文章把 WebGPU 的 command buffer 模型講得蠻清楚：Device → Shader Module → Compute Pipeline → Buffer → Command Encoder，這套流程理解了，後面要做任何 GPGPU 計算都有底。不過說實話，這更像是一本書的導讀（作者確實在推書），但技術含量是實在的。

**重點：**
- WebGPU 的 WGSL shader 讓你直接操控 GPU 做通用計算，不再只是畫三角形
- Rust → WASM → TypeScript 的工作流搭配 `wasm-pack build --target web` 一條龍
- 但是... WebGPU 瀏覽器支援度還不是 100%，Safari 還在追，production 用之前先看看你的用戶都用什麼瀏覽器

---

## ⚡ 一句話帶過

- **[Why Native CSS Nesting Matters](https://dev.to/tu_codigocotidiano_f173d/why-native-css-nesting-matters-less-repetition-more-real-structure-in-your-css-2f8l)** — 原生 CSS Nesting 全面到位，SCSS 的棺材板又被釘了一根釘子
- **[Butterfly CSS vs. Tailwind CSS: Partners or Rivals?](https://dev.to/butterflycss/butterfly-css-vs-tailwind-css-partners-or-rivals-422b)** — 又一個要挑戰 Tailwind 的框架，帶內建動畫支援，但生態系差距不是一點半點
- **[组件测试策略：测试 Props、事件和插槽](https://juejin.cn/post/7621074045393338422)** — Props、Events、Slots 三把刀，元件測試的系統化思路整理得不錯
- **[Vue3 單元測試實戰：從組合式函數到組件](https://juejin.cn/post/7621014914937405482)** — Vue 3 測試從 Composable 寫到元件，終於有人把這條路走完了
- **[When Meta's AI Agent Hallucinates a SEV1 Incident](https://dev.to/olivier-coreprose/when-meta-s-ai-agent-hallucinates-a-sev1-incident-fallout-and-fix-5124)** — AI Agent 自己幻覺出一個 P1 事故，然後團隊真的去救火了，比恐怖片還恐怖
- **[Claude Code's Deny List Bypass](https://dev.to/gentic_news/claude-codes-deny-list-bypass-how-to-protect-your-codebase-from-compound-commands-3dk2)** — Claude Code 的 deny list 只看複合指令第一個 token，安全模型有洞，用的人注意
- **[Top 5 Free Tailwind Component Libraries in 2026](https://dev.to/chnkc41/top-5-free-tailwind-component-libraries-in-2026-49f9)** — 不想花錢買 UI 套件的看過來，五個免費 Tailwind 元件庫實測比較
- **[Vercel vs Hetzner in 2026](https://dev.to/devtoolpicks/vercel-vs-hetzner-in-2026-which-is-actually-worth-it-for-solo-developers-2og8)** — 獨立開發者的永恆難題：Vercel 的便利 vs Hetzner 的便宜，答案還是「看你時間值多少錢」
- **[Mantine SelectStepper 2.0.0](https://dev.to/undolog/mantine-selectstepper-200-swipe-scroll-resize-repeat-6ch)** — Mantine 生態又多一個好用元件，支援觸控手勢和響應式 props
- **[dirham v1.3.0 — A Universal Web Component](https://dev.to/pooyagolchian/dirham-v130-a-universal-web-component-for-the-uae-dirham-symbol-5gfh)** — 為了一個 Unicode 18.0 的貨幣符號做了一整個 Web Component，這就是工程師的浪漫
- **[Build a Real-Time Social Media App with Next.js](https://dev.to/astrodevil/build-a-real-time-social-media-app-with-insforge-minimax-and-nextjs-3jb8)** — Next.js + MiniMax AI 做即時社群平台，前端 × AI 整合的完整範例

---

## 📚 慢慢啃

- **[Code Your Own Virtual DOM in 100 Lines of JavaScript](https://dev.to/luizgarcia/code-your-own-virtual-dom-in-100-lines-of-javascript-a7h)** — 百行 JS 手刻 Virtual DOM，讀完你會真正理解 React 的 diffing 為什麼那樣設計，比看十篇「React 原理」都值
- **[The software industry is ready to grow](https://dev.to/ben/the-software-industry-is-ready-to-grow-4ie4)** — DEV.to 創辦人的長文觀察：AI 工具不是在取代開發者，而是在擴大整個產業的蛋糕，值得花 15 分鐘想想自己在哪個位置
- **[LLM Non-Determinism: What Providers Guarantee, and How to Build Around It](https://dev.to/jhagerer/llm-non-determinism-what-providers-guarantee-and-how-to-build-around-it-3502)** — 你以為設了 temperature=0 輸出就穩定？這篇告訴你各家 provider 的保證範圍有多窄，以及怎麼在架構層面應對
- **[90% of Claude-linked output going to GitHub repos with <2 stars](https://www.claudescode.dev/?window=since_launch)** — 數據顯示 Claude Code 產出的 90% 程式碼流向低星 repo，vibe coding 到底在造什麼？值得想想 AI 輔助開發的真實樣貌
- **[I Built a Visual Branching Code History Plugin for IntelliJ IDEA](https://dev.to/dexter_b95966b3d296e/i-built-a-visual-branching-code-history-plugin-for-intellij-idea-4fh6)** — 把程式碼修改歷史做成視覺化分支樹，Git 看不到的細粒度變更這裡看得到，DX 小品但想法有趣
