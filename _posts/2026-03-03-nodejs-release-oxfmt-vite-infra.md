---
title: "Node.js 砍掉奇數版、oxfmt 快 Prettier 45 倍、Vite 已成基建"
date: 2026-03-03
description: "Node.js 宣布廢除沿用十年的奇偶版本規則，oxfmt 以 Rust 之姿挑戰 Prettier 地位，Vite 從開發工具進化為前端生態基礎設施。另有 React 反模式、CSS 原生主題切換等實用內容。"
tags: [frontend, nodejs, tooling, react, css, ai]
---

今天三件大事。Node.js 終於想通了，把那個從來沒人搞懂的奇偶版本規則給砍了。尤雨溪的 VoidZero 團隊又出了個格式化工具 oxfmt，號稱比 Prettier 快 45 倍（Rust 黨又贏了）。然後有篇 Vite 回顧文把這幾年的演進講得挺透，值得回頭看看我們到底是怎麼走到今天的。

---

## 🔧 今日硬菜

### [Node.js 宣布重大調整，運行十年的規則要改了！](https://juejin.cn/post/7612492005219614783)

十年了，Node.js 終於承認那個「偶數穩定、奇數實驗」的發布策略是個笑話。官方數據很誠實：絕大多數開發者和企業根本不碰奇數版本，大家就是乾等 LTS。既然沒人用，何必浪費社區志工的命去維護？

從 Node.js 27（2027 年 4 月）開始，一年只出一個主版本，每個版本都直接轉 LTS。新增 Alpha 階段（10 月到隔年 3 月）讓你提前跑 CI，4 月正式發布後 API 定型。Node 26 是舊制度的最後一版。

**重點：**
- 每年 4 月一個主版本，10 月轉 LTS，不再區分奇偶——閉眼升級就對了
- LTS 維護期 29 個月不變，V8 採用週期不變，品質標準不變
- 但是... 這代表 breaking changes 集中在一次大版本爆發，Alpha 階段的 CI 測試變得更關鍵，別偷懶不跑

### [oxfmt：尤雨溪力推的新格式化工具，速度比 Prettier 快 45 倍](https://juejin.cn/post/7612574523557756928)

前端工具鏈的 Rust 化已經不是趨勢，是既成事實。oxfmt 是 Oxc 生態（VoidZero 團隊用 Rust 打造的 JS/TS 全鏈路工具鏈）的格式化器，本質上就是 Prettier 的 Rust 重寫版。100% Prettier 相容、支援 `--migrate-prettier` 一鍵遷移、import 自動排序、VS Code 插件直接用。

踩過 Prettier 在大型 monorepo 裡慢到崩潰的人都知道，格式化速度不是小事——它直接影響 pre-commit hook 的體驗和 CI 的等待時間。45 倍不是噱頭，是 Rust 編譯型語言對 JS 解釋型的碾壓。

**重點：**
- Oxc 生態補完最後一塊拼圖：Rolldown（打包）+ oxlint（lint）+ oxfmt（格式化），Rust 全家桶成型
- 遷移成本極低：`oxfmt --migrate prettier` 搞定，VS Code 換個 formatter 設定就好
- 但是... Beta 階段，邊緣 case 的格式化結果可能跟 Prettier 有微妙差異，團隊大規模切換前建議先跑 diff 驗證

### [Vite 發展現狀與回顧：從「極致開發體驗」到生態基礎設施](https://juejin.cn/post/7612545160433582089)

這篇把 Vite 從誕生到現在的脈絡梳理得很清楚。重點不在教你怎麼用 Vite（2026 年了你應該會了），而是講清楚一件事：Vite 已經不只是「Webpack 的替代品」，它是 SvelteKit、SolidStart、Qwik 等框架的底層 runtime，是 Vitest、VitePress 的共享核心，是前端工具生態的基礎設施層。

從瀏覽器原生 ESM 的機會出發，到依賴預構建解決 CJS 相容問題，再到 Rollup 保證生產構建品質——這個「開發用 ESM、生產用 Rollup」的雙形態設計確實優雅。文章也坦誠提到了局限：超大型 monorepo 的依賴預構建偶爾翻車、SSR 場景仍需框架層面補完。

**重點：**
- Vite 的定位已從「dev tool」升級為「前端基建平台」，理解這個轉變對技術選型很重要
- Rolldown（Rust 版 Rollup）正在開發中，未來生產構建也會 Rust 化，速度再上一層
- 但是... 老專案從 Webpack 遷移不是「裝個 Vite 就完事」，依賴預構建的相容性坑需要逐個解決

---

## ⚡ 一句話帶過

- **[告別 Prop Drilling：Context API 的陷阱、Reducer 模式與原子化狀態庫原理](https://juejin.cn/post/7612481256647950351)** — 把 Context 性能問題和 Zustand/Jotai 原理講得挺透，踩過 re-render 坑的人看了會點頭
- **[React 反模式排查手冊：從性能殺手到邏輯陷阱](https://juejin.cn/post/7612657123727835176)** — JSX 裡建物件和函式是最常見的隱形效能殺手，這篇整理得很實用
- **[CSS 變數 + 主題切換：從 CSS-in-JS 回歸原生方案](https://juejin.cn/post/7612524046837645348)** — 600+ 組件項目從 styled-components 遷到 CSS 原生變數，效能大幅提升，方向是對的
- **[defineModel 是進步還是邊界陷阱？](https://juejin.cn/post/7612908108270649370)** — Vue 3.4 的 defineModel 語法糖背後的狀態哲學辯論，寫得有深度
- **[Vue3 中 emit 能 await 嗎？事件機制裡的非同步陷阱](https://juejin.cn/post/7612557781271199798)** — 不能，但很多人以為可以。這篇把原因講清楚了
- **[從微信小程式 data-id 到 React 列表效能優化](https://juejin.cn/post/7612492005219598399)** — 少用閉包、多用 data-* 屬性，memo 和虛擬化才能真正生效
- **[Giggles — A Batteries-Included React Framework for TUIs](https://github.com/zion-off/giggles)** — 用 React 寫終端 UI 的新框架，靈感來自 Ink 和 Bubbletea，有趣但不確定有多少人需要
- **[用這 9 個瀏覽器 API 把頁面從「卡成 PPT」救回到 90+](https://juejin.cn/post/7612479947865784363)** — IntersectionObserver、requestIdleCallback 等原生 API 的效能優化實戰，基礎但實用
- **[OpenTiny NEXT-SDK：用 MCP 協議讓 AI Agent 操控前端界面](https://juejin.cn/post/7612562930848710682)** — 前端 × AI 的新方向，讓 AI 直接操作 UI 元素，概念有意思但生態還很早期
- **[msw-fetch-mock: Undici-Style Fetch Mocking for MSW](https://dev.to/recca0120/msw-fetch-mock-undici-style-fetch-mocking-for-msw-5e2j)** — 統一前端和 Node.js 測試中的 HTTP mock 策略，寫測試的人可以看看
- **[Build a Voice Agent in JavaScript with Vercel AI SDK](https://dev.to/mkp_bijit/build-a-voice-agent-in-javascript-with-vercel-ai-sdk-1dc3)** — 用 Vercel AI SDK 做語音 Agent 的完整流程，AI × 前端整合的實戰參考
- **[別再用 scoped 了！Vue 項目中真正安全的 CSS 封裝方案](https://juejin.cn/post/7612557781271085110)** — scoped 的局限性比你想的多，CSS Modules 才是正解

---

## 📚 慢慢啃

- **[組合式函數的設計模式：如何寫出真正可複用的 Vue3 Composables](https://juejin.cn/post/7612492005219172415)** — 從「能複用」到「複用得好」的思維框架，寫 Vue3 的人讀完會重構自己的 composables
- **[手把手帶你了解 Webpack HMR 的基本原理](https://juejin.cn/post/7612529289542271022)** — 雖然大家都在用 Vite 了，但搞懂 HMR 從 WebSocket 通知到模組替換的完整流程，對理解任何 dev server 都有幫助
- **[從平面到空間：用 React Three Fiber 構建 3D 產品網格](https://juejin.cn/post/7612557781270921270)** — R3F + GLSL 著色器的實戰教程，想在前端玩 3D 的好入門材料
- **[別再亂寫 16px 了！CSS 單位體系已經進入「計算時代」](https://juejin.cn/post/7612657123742187535)** — calc/clamp 搭配 CSS 單位建立響應式設計系統，從工程視角把單位體系講透
- **[從「傻等」到「流淌」：我在 AI 項目中實現串流輸出的血淚史](https://juejin.cn/post/7612641614825521202)** — SSE、EventSource、打字機效果的正確做法，做 AI 前端整合的必讀
