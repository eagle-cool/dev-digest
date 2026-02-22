---
title: "Next.js 16 快取大翻新、React 偵錯神器、Deno Sandbox 上線"
date: 2026-02-22
description: "Next.js 16 把快取從頁面級拉到 fetch 級，React 多了一個直接掛 Fiber tree 的偵錯擴充，Deno 推出 microVM 沙箱跑不信任程式碼。今天的前端圈很忙。"
tags: [frontend, react, nextjs, nodejs, tooling]
---

今天的前端圈有三件事值得坐下來好好看。Next.js 16 終於把快取模型搞清楚了（是的，以前那個真的很混亂），有人寫了一個直接讀 React Fiber tree 的 DevTools 擴充套件來抓效能問題，然後 Deno 推出了 microVM 沙箱——以後跑不信任的程式碼不用再自己搞 Docker 了。

---

## 🔧 今日硬菜

### [Next.js 16 Caching Explained: Revalidation, Tags, Draft Mode & Real Production Patterns](https://dev.to/realacjoshua/nextjs-16-caching-explained-revalidation-tags-draft-mode-real-production-patterns-26dl)

踩過 Next.js 快取坑的人都知道那種痛——ISR 說好 60 秒重新驗證，結果頁面死活不更新，你在 dev mode 測了半天覺得沒問題，一上 production 就爆。Next.js 16 做了一個關鍵的架構轉變：快取的單位從「頁面」變成了「fetch 呼叫」。每個 `fetch()` 可以獨立設定 `revalidate` 時間和 `tags`，然後用 `revalidateTag()` 做精準的按需失效。這不是小改動，這是整個心智模型的重建。搭配 Draft Mode，CMS 預覽流程也終於不用靠通靈了。

**重點：**
- 快取粒度從頁面級降到 fetch 級，每個 API 呼叫可以獨立控制快取策略
- `revalidateTag()` 實現精準的按需失效，不用再整頁重建
- 但是... dev mode 和 production 的快取行為還是不一樣，測試永遠要用 `next build && next start`

### [Open-source React DevTools extension for spotting performance and state issues in real time](https://dev.to/hoainhoblogdev/open-source-react-devtools-extension-for-spotting-performance-and-state-issues-in-real-time-54ib)

React 官方的 DevTools 很好，但它基本上是個「你知道要找什麼才找得到」的工具。這個開源的 Chrome 擴充套件做的事不太一樣——它直接掛上 React Fiber tree，主動幫你掃描常見的效能地雷：直接 mutate state、用 index 當 key、useEffect 沒做 cleanup、元件重複渲染過多。還附帶 CLS 即時監控和記憶體洩漏偵測。聽起來很重，但作者做了節流處理，只在 DevTools 面板開啟時才注入。用 `npx react-debugger` 就能裝。

**重點：**
- 主動掃描而非被動查詢，抓 state mutation、missing key、effect cleanup 等常見問題
- 支援 render 次數追蹤、CLS 監控、記憶體監控、甚至 Redux state tree 瀏覽
- 但是... 這是個新專案，在大型應用上的效能損耗還需要實戰驗證，建議先在 staging 環境試

### [5 Ways Deno Sandbox Changes How You Run Untrusted Code in APIs](https://dev.to/1xapi/5-ways-deno-sandbox-changes-how-you-run-untrusted-code-in-apis-32fh)

還記得 `vm2` 嗎？那個被爆出一堆 CVE 的 Node.js 沙箱？Deno 在 2026 年 2 月推出了 Deno Sandbox——用 microVM 來跑不信任的程式碼。每次執行都開一個獨立的 VM，毫秒級冷啟動，內建 CPU/記憶體/網路資源限制。對於需要讓使用者提交程式碼的場景（playground、webhook transformer、plugin 系統），這比自己搞 Docker + seccomp + syscall filter 省事太多了。搭配同期 GA 的 Deno Deploy，從 edge API 到沙箱執行一條龍。

**重點：**
- microVM 隔離，不是 container 層級，是 VM 層級的安全邊界
- 內建 fork bomb 防護、記憶體上限、CPU 時間預算、網路存取預設關閉
- 但是... 這是 Deno 生態限定，如果你的 stack 是 Node.js，還是得等社群移植或繼續用 Docker

---

## ⚡ 一句話帶過

- **[Minions: Stripe's one-shot, end-to-end coding agents](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)** — Stripe 內部的 AI coding agent 架構公開了，一次性端到端完成任務，不搞多輪對話
- **[Improving Accessibility - Tooltip](https://dev.to/hritickjaiswal/improving-accessibility-tooltip-3ogc)** — React tooltip 的無障礙實作，重點在時間延遲、焦點管理和鍵盤互動三件事
- **[Stop Repeating React Setup: Introducing create-react-forge](https://dev.to/chiragmak10/stop-repeating-react-setup-introducing-create-react-forge-29hd)** — 又一個 React 腳手架工具，這次整合了 Vite/Next.js + Tailwind + 狀態管理 + 測試的一鍵選擇
- **[Adding Content Moderation to a SvelteKit App with OpenAI's Moderation API](https://dev.to/rrosset91/adding-content-moderation-to-a-sveltekit-app-with-openais-moderation-api-1f4e)** — SvelteKit + OpenAI 內容審核 API 的實戰整合，部署在 Cloudflare Pages 上
- **[react-native-root-jail-detect](https://dev.to/rushikeshpandit/published-a-lightweight-library-for-rootjailbreak-detection-react-native-1eam)** — React Native 輕量級 root/越獄偵測庫，金融類 App 的基本配備
- **[Postgres Is Your Friend. ORM Is Not](https://hypha.pub/postgres-is-your-friend-orm-is-not)** — ORM 讓你離 SQL 越來越遠，直到你在 production 踩到效能懸崖才想起來
- **[Building a Visual Regression Engine with Playwright](https://dev.to/nijil71/building-a-visual-regression-engine-in-python-with-playwright-2117)** — 用 Playwright 做視覺回歸測試，跨斷點截圖比對，抓 CSS 改壞的好幫手
- **[I Bet Your Table Code is 200+ Lines](https://dev.to/jacksonkasi/i-bet-your-table-code-is-200-lines-prove-me-wrong-4jfe)** — 表格元件的程式碼膨脹問題，200 行起跳 600 行不意外
- **[Idempotency in APIs](https://dev.to/fazal_mansuri_/idempotency-in-apis-why-your-retry-logic-can-break-everything-and-how-to-fix-it-345k)** — API 冪等性入門，你的 retry 邏輯可能正在幫使用者重複扣款
- **[Next.js + Tauri 2 用 Static Export 裝進桌面端](https://juejin.cn/post/7608393462304505890)** — Next.js 靜態匯出 + Tauri 2 打包成桌面/行動 App 的完整踩坑筆記

---

## 📚 慢慢啃

- **[Building a Production-Grade Table Editor with React and XState](https://dev.to/keyurparalkar/building-a-production-grade-table-editor-with-react-and-xstate-41ke)** — 用狀態機管理表格編輯器的 undo/redo、拖曳排序、分組，讀完你會重新思考複雜 UI 的狀態管理方式
- **[Beyond JSON: Why My Next Project Uses a Custom Binary Protocol](https://dev.to/makalin/beyond-json-why-my-next-project-uses-a-custom-binary-protocol-o2l)** — JSON 是 CPU 殺手這件事大家都知道但都在裝沒事，這篇認真算了帳
- **[从输入 URL 到页面显示的完整技术流程](https://juejin.cn/post/7609190755588620326)** — 經典面試題的完整技術拆解，從 URL 解析到瀏覽器渲染，適合重新校準基礎認知
