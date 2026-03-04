---
title: "WebMCP SDK 實戰、Electron 打贏 Native 的真正原因、React 萬級瀑布流"
date: 2026-03-04
description: "Chrome WebMCP 從概念到 SDK 實戰指南、Tonsky 解析 Electron 為何贏了 Native、DreamMasonry 用 Float64Array 實現萬級 React 瀑布流虛擬化"
tags: [frontend, react, web-platform, ai]
---

今天前端圈三件事最值得聊。Chrome WebMCP 從「概念預覽」到「這是你的 SDK，開工吧」只花了一週，Tonsky 出來解釋為什麼 Claude 是 Electron app（劇透：不是因為 AI 還不夠好），然後有個傢伙受不了現有的 React masonry library，花兩週自己寫了一個——結果還真比較快。

---

## 🔧 今日硬菜

### [Chrome Just Made Every Website an AI Agent API — Here's How to Build For It](https://dev.to/up2itnow0822/chrome-just-made-every-website-an-ai-agent-api-heres-how-to-build-for-it-1f4h)

上週才[聊過 WebMCP 的概念預覽](https://eagle-cool.github.io/dev-digest/posts/webmcp-preview-mcp-vs-cli-react-m3/)，這週就有人把完整的 SDK 做出來了。Chrome 146 的 WebMCP 讓網站用兩種方式暴露能力給 AI agent：HTML meta tag（宣告式）和 JavaScript API（命令式）。`webmcp-sdk` 已經上 npm，附帶 React bindings、安全模組和測試工具。這代表前端開發者現在要開始認真想「我的網站對 AI agent 長什麼樣」這個問題了——不是未來式，是現在進行式。

**重點：**
- 宣告式用 `<meta name="mcp:tool">` 描述靜態能力，零 JS 依賴；命令式用 `navigator.mcp.registerTool()` 處理動態功能
- `@webmcp/react` 提供 `<WebMCPProvider>` 和 `<DeclareTool>` 元件，React 開發者可以直接用
- 但是... 安全問題很現實——`@webmcp/security` 是必裝的，不然你的 admin 功能可能被 agent 戳到

### [Claude is an Electron App because we've lost native](https://tonsky.me/blog/fall-of-native/)

Tonsky（對，就是那個寫 Clojure 和 Skija 的 Tonsky）寫了一篇很到位的分析。起因是有人問：「Claude 花了 $20K 用 agent swarm 寫 Rust 編譯器，但桌面版卻是 Electron app？」大家的第一反應是「LLM 還不夠好所以寫不了 native」，Tonsky 說不對——是 native 本身已經沒什麼好提供的了。API 難用、UI 一致性早就名存實亡（看看 Apple 的 Liquid Glass），效能優勢是理論上存在但實際上沒人在乎。結論很殘忍但很誠實：問題不是 Electron，是整個業界的品質標準在下降。

**重點：**
- Native API 的開發體驗比 Web 差，OS 廠商反而在趕走 native 開發者
- Apple 的 UI 一致性已經是「看心情設計」的水準——連圓角半徑都對不齊
- 但是... Tonsky 自己也說 Web 不是解方，Slack 不需要載入 80MB 來顯示三則訊息——這是「選擇擺爛」，跟技術棧無關

### [Why I Built Another Masonry Library for React (And Why It's Faster)](https://dev.to/adioof/why-i-built-another-masonry-library-for-react-and-why-its-faster-3195)

又一個「受不了現有方案所以自己寫」的故事，但這次有料。作者用了 `react-masonry-css`（不支援虛擬化，2000 item 就卡）和 Masonic（支援虛擬化但不支援自訂 scroll container，2022 年後就沒更新），都踩到痛點後決定自幹。`DreamMasonry` 用 `Float64Array` 追蹤 column heights，萬級 item 只渲染約 50 個 DOM node，infinite scroll 和 custom scroll container 都內建。13KB，零外部依賴。實測數據也很漂亮：10K items 65ms render time、58 FPS。

**重點：**
- 虛擬化預設開啟，用 `requestAnimationFrame` 計算可見範圍，只在範圍變更時才觸發 React re-render
- 暴露 `useGrid`、`usePositioner`、`useInfiniteScroll` 三個 hooks，可以拆開單獨用
- 但是... 200 item 以下的小場景用 `react-masonry-css` 就夠了，作者自己也說別過度工程

---

## ⚡ 一句話帶過

- **[GPT-5.3 Instant](https://openai.com/index/gpt-5-3-instant/)** — OpenAI 又出新型號了，HN 275 分炸鍋，但你的 production prompt 大概又要重新 tune 一遍
- **[Gemini 3.1 Flash-Lite: Built for intelligence at scale](https://dev.to/googleai/gemini-31-flash-lite-built-for-intelligence-at-scale-3i8e)** — Google 最便宜的 Gemini 3 系列模型，專打高量低成本場景，前端 AI 整合的成本門檻又降了
- **[Stop Writing .subscribe() in Angular — Use toSignal Instead](https://dev.to/paszekdev/stop-writing-subscribe-in-angular-components-use-tosignal-instead-bom)** — Angular 的 Signals 遷移實戰，還在寫 `takeUntilDestroyed` 搭配 `tap` 的人該看看了
- **[I Built a Voice Assistant That Runs Entirely in Your Browser](https://dev.to/cppshane/i-built-a-voice-assistant-that-runs-entirely-in-your-browser-4jfd)** — WebAssembly + AI，語音助理全跑在瀏覽器裡，Web 平台能做的事又多了一件
- **[design-auditor: Lighthouse for your design system](https://dev.to/__aa5b04f75e3a/i-built-a-cli-that-catches-design-inconsistencies-like-lighthouse-but-for-your-design-system-nc7)** — 你的專案有 54 種藍色和 22 種 line-height？這個 CLI 幫你全部抓出來
- **[用 scroll-padding 實現優雅的錨點跳轉](https://juejin.cn/post/7612859059739328562)** — 還在用 `:before` 偽元素 hack 錨點被 fixed header 遮住的問題？`scroll-padding-top` 才是原生解法
- **[原生 JS 側邊欄縮放：從 DOM 監聽到底層優化](https://juejin.cn/post/7612929019807170587)** — 不靠任何 library 實作可拖拽側邊欄，mousedown/move/up 三元組邏輯寫得很扎實
- **[GitHub Is Having Issues](https://www.githubstatus.com/incidents/n07yy1bk6kc4)** — GitHub 又掛了（HN 201 分），檔案載不出來、repo 建不了，日常
- **[Skills Are Overhyped: Converted Vercel's Into Cursor Rules](https://dev.to/alonsarias/skills-are-overhyped-i-converted-vercels-into-cursor-rules-and-got-real-performance-wins-1fc9)** — 把 Vercel 的 AI skills 轉成 Cursor rules 後效能反而更好，AI 工具的「最佳實踐」還在摸索期
- **[VS Code Spec Management with WYSIWYG Markdown Editor](https://dev.to/thlandgraf/i-built-a-spec-management-extension-with-a-wysiwyg-markdown-editor-in-a-vs-code-webview-lessons-h5d)** — 在 VS Code webview 裡搞 WYSIWYG Markdown 編輯器，踩坑心得值得做 extension 的人參考

---

## 📚 慢慢啃

- **[An Interactive Intro to CRDTs](https://jakelazaroff.com/words/an-interactive-intro-to-crdts/)** — 2023 年的經典最近又被翻出來（HN 87 分），用互動式範例解釋 CRDT 的核心概念，想做即時協作功能的必讀
- **[How Zero-Knowledge File Sharing Works: AES-256-GCM in the Browser](https://dev.to/fileshot_9818357dbe6cc693/how-zero-knowledge-file-sharing-works-aes-256-gcm-in-the-browser-49mo)** — 用 Web Crypto API 在瀏覽器端做零知識加密，server 永遠碰不到明文，讀完你會重新思考前端的安全邊界
- **[How we rebuilt search architecture for GitHub Enterprise Server](https://github.blog/engineering/architecture-optimization/how-we-rebuilt-the-search-architecture-for-high-availability-in-github-enterprise-server/)** — GitHub 官方 blog 的搜尋架構重構實錄，搜尋體驗是前端 UX 的核心，底層怎麼撐起來的值得了解
- **[Deep Dive: SSE vs HTTP Streamable — What's the Difference?](https://dev.to/zrcic/deep-dive-sse-vs-http-streamable-whats-the-difference-1829)** — MCP transport 選型深度比較，做 AI 整合前先搞懂 SSE 和 HTTP Streamable 在連線管理和狀態處理上的根本差異
