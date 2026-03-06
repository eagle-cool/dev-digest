---
title: "Angular 21 Signal Forms 驗證三段式、動態 HTML 壓縮省 27 倍、RN 遷移 Expo 零停機"
date: 2026-03-06
description: "Angular Signal Forms 新增 ignoreValidators 精準控制非同步驗證提交、RFC 9842 Delta 壓縮讓動態 HTML 傳輸縮小 27 倍、Shim Proxy 模式讓 React Native 零中斷遷移 Expo CNG"
tags: [frontend, react, web-platform, typescript]
---

今天三道硬菜各有意思。Angular 終於把表單非同步驗證的提交控制做到位了（等了多久），有人把 RFC 9842 壓縮字典拿來對動態 HTML 開刀，結果是 78KB 壓到 52 bytes——沒寫錯，是 bytes。然後 React Native 陣營出了一個 Strangler Fig 式遷移方案，讓你一邊出版一邊換引擎。

---

## 🔧 今日硬菜

### [Angular 21 Signal Forms: ignoreValidators Explained](https://dev.to/brianmtreese/angular-21-signal-forms-ignorevalidators-explained-3gpb)

Angular Signal Forms 加了一個看似簡單但極其實用的選項：`ignoreValidators`。解決的問題很具體——使用者按下送出時，非同步驗證器（比如查使用者名稱有沒有被用過）還在跑，這時候表單該不該送出去？以前 Angular 的答案是「管你的，先送了再說」，這在生產環境根本不能接受。

現在有三段式控制：`pending`（預設，不等非同步就送）、`none`（全部驗證跑完才能送，最安全）、`all`（無視所有驗證狀態，適合草稿自動存檔）。搭配 `onInvalid` callback 還能自動 focus 到第一個出錯的欄位——這才是表單 UX 該有的樣子。

**重點：**
- 三種模式對應三種場景：快速回應、嚴格驗證、草稿保存
- `none` 模式搭配 `onInvalid` + `errorSummary()` 是生產環境首選組合
- 但是... Signal Forms 整套 API 還在早期階段，現有的 Reactive Forms 專案要遷移得三思

### [Beyond Static Resources: Delta Compression for Dynamic HTML](https://dev.to/carlosmateom/beyond-static-resources-delta-compression-for-dynamic-html-3hn4)

這篇是今天的技術炸彈。RFC 9842 Compression Dictionary Transport 讓瀏覽器拿「上一次的回應」當壓縮字典來壓「這一次的回應」。聽起來理所當然？但 HTTP 規範一直沒支援這件事——因為動態頁面的 `Cache-Control: no-cache` 會讓字典立刻被清掉。

作者提出的 Dictionary TTL 擴展把字典存活時間跟 HTTP cache 解耦。實測數據驚人：同 session 跨查詢導航，標準 Brotli 壓到 23KB，Delta 壓縮只要 1-2.7KB——**小 10 到 27 倍**。更誇張的是 tab 復原場景，78KB 的頁面壓到 52 bytes。沒錯，五十二個位元組。因為只有 CSRF token 和 CSP nonce 變了，其他全部「從字典複製」。

**重點：**
- Chromium 已有實驗性 flag `chrome://flags/#enable-compression-dictionary-ttl`
- 對 SSR 框架（Next.js、Nuxt）的動態頁面特別有感，session 穩定的內容佔比通常超過 70%
- 但是... 伺服器端得替每個 active session 存一份字典，10 萬併發 ≈ 10GB RAM，基礎設施成本不能不算

### [The Shim Proxy Pattern: A Zero-Disruption Path from React Native Legacy to Expo CNG](https://dev.to/thenoroboto/the-shim-proxy-pattern-a-zero-disruption-path-from-react-native-legacy-to-expo-cng-ai1)

踩過 React Native 升級地獄的人都知道那種痛——`ios/` 和 `android/` 資料夾裡堆了三年的手動設定，原本懂 native 層的人早就離職了，然後你被要求升級到 New Architecture。這篇提出的 Shim Proxy 模式用 Strangler Fig 架構思路來解：Metro bundler 的 `extraNodeModules` 攔截 import，把舊套件靜默替換成 Expo 相容的 shim，`blockList` 隔離版本衝突。`legacy/` 繼續出版上線，`expo-bridge/` 在旁邊慢慢驗證。

最聰明的是讓 `ios/` 和 `android/` 變成 `npx expo prebuild` 的一次性產物——不進 git，不累積技術債。升級 React Native 版本從「多人數週苦戰」變成「跑一個指令」。

**重點：**
- Metro `extraNodeModules` + `blockList` 組合拳是核心，shim 只需暴露相同 API
- 前提是業務邏輯必須跟 native shell 分離，否則沒有共用層可以建構
- 但是... Config Plugins 本身也是維護表面，每次 Expo SDK 大版本更新都可能默默壞掉

---

## ⚡ 一句話帶過

- **[Next.js Rebuilt, NumPy in TypeScript, Six AI Predictions](https://dev.to/urbanisierung/nextjs-rebuilt-numpy-in-typescript-six-ai-predictions-3ob0)** — 本週前端圈摘要，Next.js 重構消息最值得追，TypeScript 版 NumPy 是個有趣的嘗試
- **[Supercharge Your Webpack 5 Builds with Rust](https://dev.to/ndtao2020/supercharge-your-webpack-5-builds-with-rust-568g)** — OXC 編譯器塞進 Webpack 5，還沒遷到 Vite 的團隊可以先拿這個止痛
- **[AI-Assisted Engineering: The Productivity Paradox Nobody Warns You About](https://dev.to/matthiasbruns/ai-assisted-engineering-the-productivity-paradox-nobody-warns-you-about-4oon)** — METR 實驗顯示 AI 輔助編程反而慢 19%，終於有人拿隨機對照數據打臉「效率提升 50%」的行銷話術
- **[MCP Tool Overload: Why More Tools Make Your Agent Worse](https://dev.to/nebulagg/mcp-tool-overload-why-more-tools-make-your-agent-worse-5a49)** — 給 AI Agent 塞 50 個工具結果它選擇困難，少即是多的老道理又被驗證一次
- **[SQLite Performance Tips for Web Applications](https://dev.to/ahmet_gedik778845/sqlite-performance-tips-for-web-applications-29o3)** — WAL 模式、PRAGMA 調校、索引策略，SQLite 當 Web 後端的人必收
- **[ReactJS Anti Pattern: Passing Setters Down the Components Tree](https://dev.to/kkr0423/reactjs-anti-pattern-passing-setters-down-the-components-tree-2d0m)** — 別再把 `setState` 直接傳給子元件了，這篇講了為什麼以及怎麼改
- **[Adding Giscus Comments to Next.js Blog Pages](https://dev.to/estebankt123/adding-giscus-comments-to-nextjs-blog-pages-5e1b)** — Next.js 部落格接 Giscus 留言，GitHub Discussions 當後端，零成本零資料庫
- **[Stop Using Grey Text](https://catskull.net/stop-using-grey-text.html)** — 你的淺灰色次要文字可能正在折磨一半的使用者，可讀性問題比你想的嚴重
- **[Why Remote MCP Servers Break When Tools Touch Files](https://dev.to/deathsaber/why-remote-mcp-servers-break-when-tools-touch-files-2em)** — MCP 遠端部署踩坑實錄：本地檔案系統假設在遠端環境直接崩壞
- **[GPT-5.4 Complete Guide: What's New, API Access, and How to Use It](https://dev.to/ashinno/gpt-54-complete-guide-whats-new-api-access-and-how-to-use-it-55gk)** — GPT-5.4 工具呼叫 token 省 47%，對 AI 整合前端的開發者來說是實質利多

---

## 📚 慢慢啃

- **[浏览器渲染管线深度拆解：从 Parse HTML 到 Composite Layers 的每一帧发生了什么](https://juejin.cn/post/7613652045741785115)** — 從 HTML 解析到合成層的完整渲染管線拆解，搞前端效能優化的話這篇能幫你建立完整的心智模型
- **[React Native 性能优化指南](https://juejin.cn/post/7613639135041667122)** — New Architecture、Fabric、Hermes、TurboModules 全家桶的效能優化策略，配合今天硬菜的 Expo 遷移文一起讀效果更佳
- **[前端异常捕获：从"页面崩了"到"精准定位"的实战架构](https://juejin.cn/post/7613605454283472947)** — 同步、非同步、資源載入錯誤全方位捕獲，加上 SourceMap 還原和自動告警，監控體系建設的完整指南
- **[组合式函数 Composables 的设计模式：如何写出可复用的 Vue3 Hooks](https://juejin.cn/post/7613808397809238042)** — Vue 3 Composables 該怎麼切、邊界在哪、什麼時候該忍住不抽——寫 React hooks 的人讀了也會有共鳴
