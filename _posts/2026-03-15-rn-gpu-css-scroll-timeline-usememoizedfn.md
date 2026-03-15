---
title: "React Native 零拷貝 GPU 管線、CSS scroll-timeline 全攻略、useCallback 該退休了"
date: 2026-03-15
description: "有人在 React Native 上跑 WebGPU compute shader 還跑到 120fps、CSS scroll-timeline 終於可以不靠 JS 做滾動動畫、ahooks useMemoizedFn 讓 useCallback 的依賴地獄成為過去式"
tags: [frontend, react, css, web-platform, ai, tooling]
---

今天三道硬菜都很硬。有人在 React Native 上把 WebGPU compute shader 跑到 4K 120fps（對，你沒看錯），CSS 終於可以純靠自己做滾動動畫不求 JavaScript，然後 ahooks 的 `useMemoizedFn` 讓我重新思考為什麼我們還在寫 `useCallback` 的依賴陣列。另外 MCP 協議的存亡之爭也很精彩——95 個 HN 留言的那種精彩。

---

## 🔧 今日硬菜

### [Zero-Copy GPU Compute on Camera Frames in React Native — What Actually Worked](https://dev.to/kbrandwijk/zero-copy-gpu-compute-on-camera-frames-in-react-native-what-actually-worked-512j)

這篇不是 demo，是真的踩坑記。作者要在 React Native 上跑 WebGPU compute shader 處理即時相機畫面，目標是 60fps 以上不燒手機。聽起來瘋狂？結果他做到了 4K 120fps，GPU frame time 才 4ms。

核心思路：iOS 的 `CVPixelBuffer` 底層是 `IOSurface`，本身就在 GPU 記憶體上。Dawn 的 `SharedTextureMemory` 可以直接把 IOSurface import 成 WebGPU texture——零拷貝。整條管線從相機到 compute shader 到 Skia Graphite 渲染，沒有任何一個 byte 離開過 GPU。

最精彩的是他踩了 13 個坑的過程。先試 naive 方案每幀拷貝 8MB 像素到 JS——三秒後 OOM。再試從 JS 端寫 texture——Dawn 的 JS binding `writeTexture()` 是半成品，silently 回傳全零。最後才領悟：既然 JS 端寫不進去，那就讓 native side 全包，JS 只拿 opaque handle。用 Reanimated 的 `useFrameCallback` 在 UI thread 跑 worklet 拉最新的 SkImage，完全繞過 JS thread 和 React reconciliation。

**重點：**
- IOSurface → Dawn SharedTextureMemory → Compute Shader → Skia Graphite，全程 GPU，零拷貝
- 支援 multi-pass shader chaining（邊緣偵測 + 色彩映射一幀搞定）和 GPU buffer readback（直方圖、特徵偵測）
- 但是... Skia Graphite 的 Dawn JS binding 一堆 stub，build system 整合花的時間比寫管線本身還長。React Native 的 GPU 故事才剛開始，路還沒鋪好

### [CSS 滚动驱动动画（scroll-timeline）：无 JS 实现滚动特效](https://juejin.cn/post/7616666565057003535)

以前做滾動動畫，你得監聽 `scroll` 事件、手動算進度、節流防手抖，寫一堆 JavaScript。現在 CSS `scroll-timeline` 讓你用三行 CSS 搞定——動畫進度直接綁定滾動位置，GPU 加速，不佔主執行緒。

`scroll()` 函數綁定到滾動容器的滾動進度，`view()` 函數在元素進出 viewport 時觸發。配合 `animation-range` 可以精確控制動畫在 `entry`、`exit`、`contain`、`cover` 哪個階段執行。進度條、視差滾動、淡入淡出、圖片縮放——以前每個都要一坨 JS，現在都是純 CSS。

文章把 `@property` 搭配 `scroll-timeline` 做數字滾動計數的技巧也值得一看，還有用 `clip-path` 做文字逐字顯示的玩法。

**重點：**
- `animation-timeline: scroll()` 綁定滾動進度，`view()` 綁定元素可見性，配合 `animation-range` 控制觸發區間
- 性能完勝 JS 方案：動畫在 compositor thread 跑，不觸發 layout/paint
- 但是... Safari 和 Firefox 還不支援。Chrome/Edge 115+ 可用，生產環境請備好 `@supports` 降級和 polyfill

### [ahooks useMemoizedFn：解决 useCallback 的依赖地狱](https://juejin.cn/post/7616660062761910306)

每個寫 React 的人都被 `useCallback` 的依賴陣列折磨過。你的函數依賴五個 state，依賴陣列寫五個變數，任何一個變化函數引用就變了，`memo` 過的子組件照樣重渲染。依賴地獄，名副其實。

ahooks 的 `useMemoizedFn` 解法很優雅：用 `useRef` 存最新函數，返回一個引用永遠不變的包裝函數。呼叫時透過 `fnRef.current` 拿到最新閉包——不需要依賴陣列，引用永遠穩定，`memo` 子組件真的不會重渲染。

原理就十行程式碼，但解決了一個困擾 React 社群多年的問題。React 官方提案的 `useEvent` 思路幾乎一樣，但至今沒有正式發布。ahooks 先把路走通了。

**重點：**
- `useMemoizedFn` = 引用永遠不變 + 永遠拿到最新 state/props，不需要依賴陣列
- 配合 `useEffect` 特別好用：空依賴就好，不用擔心函數引用變化觸發重新執行
- 但是... 它不是 React 官方 API，團隊需要統一規範。而且如果函數不傳給子組件也不當 effect 依賴，其實不需要優化

---

## ⚡ 一句話帶過

- **[MCP is dead; long live MCP](https://chrlschn.dev/blog/2026/03/mcp-is-dead-long-live-mcp/)** — 95 個 HN 留言的 MCP 協議批判文，結論是 MCP 想當 AI 的 USB-C 但目前更像 mini-USB：堪用但離標準還遠
- **[Tokis: A Performance-First, Token-Native UI Library for Building Modern Design Systems](https://dev.to/prerak/tokis-a-performance-first-token-native-ui-library-for-building-modern-design-systems-2cad)** — 又一個 design token first 的 UI 庫，切入點是「styling drift」問題，值得觀察但先別急著換
- **[WebMCP and WebAI: Exploring native AI tools in Chrome](https://dev.to/kevin-uehara/webmcp-and-webai-exploring-native-ai-tools-in-chrome-4ocf)** — Chrome 內建 AI API 實測，`navigator.ai` 的時代可能真的要來了
- **[用了大半年 Claude Code，我总结了 16 个实用技巧](https://juejin.cn/post/7616666752521732096)** — 前端工程師的 Claude Code 實戰心得，「給 AI 驗證方式」這條建議確實是最高槓桿
- **[WICK-A11Y v3.0.1: Cypress v16 Ready](https://dev.to/sebastianclavijo/wick-a11y-v301-cypress-v16-ready-upgrade-without-fear-fully-backward-compatible-77m)** — Cypress v16 要改環境變數機制（安全因素），這個 a11y 測試套件已經先準備好了
- **[5 Things AI Can't Do, Even in Redux Toolkit](https://dev.to/devunionx/5-things-ai-cant-do-even-in-redux-toolkit-43pn)** — Store 架構設計、正規化策略、middleware 排序——AI 能幫你寫 slice 但架構決策還是你的事
- **[I built a browser-only Git diff viewer using File System Access API](https://dev.to/chigichan24/i-built-a-browser-only-git-diff-viewer-using-file-system-access-api-no-server-needed-282g)** — 用 File System Access API 在瀏覽器裡直接讀本機 repo 看 diff，給 AI coding agent 做 review 用的，思路不錯
- **[I Built an MCP Server That Lets Designers Change CSS From a Notion Table](https://dev.to/vmvenkatesh78/i-built-an-mcp-server-that-lets-designers-change-css-from-a-notion-table-2hj9)** — 設計師改 Notion 表格 → AI agent 讀取 → 自動生成 design token JSON 並跑 build，design-dev 協作的有趣嘗試
- **[Claudetop – htop for Claude Code sessions](https://github.com/liorwn/claudetop)** — 即時監控你的 Claude Code session 花了多少錢，用 AI 寫 code 的成本終於可視化了
- **[WebMCP: Turn Your Web App Into an AI-Ready Tool Server — No Backend Required](https://dev.to/nicoavanzdev/webmcp-turn-your-web-app-into-an-ai-ready-tool-server-no-backend-required-2pk4)** — W3C 標準讓網頁直接暴露結構化 API 給 AI agent 呼叫，不用再靠 DOM scraping 那套

---

## 📚 慢慢啃

- **[I Analyzed Dozens of AI Agent Rules Files. Most Are Making Your Agent Worse.](https://dev.to/alexefimenko/i-analyzed-a-lot-of-ai-agent-rules-files-most-are-making-your-agent-worse-2fl)** — ETH Zurich 研究發現 LLM 生成的 rules file 讓任務成功率掉 3% 但成本漲 20%，人寫的好一點但成本也漲。讀完會重新思考你的 `.cursorrules` 到底在幫忙還是幫倒忙
- **[Vibe Coding Gets You 70% There. Here's What Happens to the Other 30%.](https://dev.to/totalvaluegroup/vibe-coding-gets-you-70-thereai-programming-webdev-productivity-heres-what-happens-to-the-1lco)** — Vibe coding 的蜜月期過後的冷靜反思：AI 能快速搭起 70% 的東西，但剩下 30% 的除錯、架構和邊界情況才是真正吃經驗的地方
- **[It's time to move your docs in the repo](https://www.dein.fr/posts/2026-03-13-its-time-to-move-your-docs-in-the-repo)** — 84 個 HN upvotes 的老生常談但依然重要：文件放在 repo 裡才能跟著 code review、版控、CI 一起走。AI 時代這點更重要——你的 agent 讀得到 repo 裡的 docs 但讀不到你的 Confluence
- **[Bring your own phosphor: thirteen problems Claude Code couldn't solve without me](https://dev.to/jord0cmd/bring-your-own-phosphor-thirteen-problems-claude-code-couldnt-solve-without-me-5fid)** — 一個老派工程師用 Claude Code 重建復古終端機介面的故事，13 個 AI 搞不定需要人類經驗介入的瞬間。文筆很好，讀起來像偵探小說
