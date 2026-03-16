---
title: "DevTools MCP 連你瀏覽器、React 最大 XSS、新聞網頁肥到 49MB"
date: 2026-03-16
description: "Chrome DevTools MCP 支援自動連接瀏覽器除錯 session，React Server Components 史上最大 XSS 漏洞拆解，紐約時報首頁 49MB 的荒謬審計報告"
tags: [frontend, react, tooling, web-platform]
---

今天三個重頭戲：Chrome 官方讓你的 AI coding agent 直接接管瀏覽器除錯 session（終於不用複製貼上 console 錯誤了），有人把 React Server Components 的 XSS 漏洞拆得乾乾淨淨——`JSON.stringify` 不轉義 `<script>` 這種低級錯誤居然在 React 身上發生，然後紐約時報首頁 49MB、422 個請求，比 Windows 95 還大。

---

## 🔧 今日硬菜

### [Chrome DevTools MCP: Debug Your Browser Session](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)

Chrome 官方出手了。DevTools MCP server 現在支援 `--autoConnect`，讓你的 AI coding agent 直接連上正在使用的瀏覽器 session。不用另開 Chrome、不用重新登入——你在 Elements panel 選個元素，直接叫 agent 去查問題；Network panel 看到一個爛 request，讓 agent 去追。這是手動除錯跟 AI 輔助除錯之間的無縫切換，踩過「agent 看不到我的頁面狀態」這個坑的人都知道這有多重要。

**重點：**
- Chrome M144（目前 Beta）新增 remote debugging 連線請求機制，透過 `chrome://inspect#remote-debugging` 啟用
- 每次連線都會彈 dialog 要用戶授權，不是靜默的——安全性有顧到
- 但是... 目前只暴露 Elements 和 Network panel 資料給 agent，其他 panel 還沒開放，離「AI 全權接管 DevTools」還有一段路

### [深入探究 React 史上最大安全漏洞](https://juejin.cn/post/7617271860845936681)

這篇把 CVE-2023-36053 從頭到尾拆了一遍——React 16.0.0 到 18.2.0、Next.js 13.4.0 之前全中。問題出在 Server Components 序列化時用 `JSON.stringify` 沒轉義 `<`、`>`、`/`，導致惡意的 HTML 直接注入客戶端執行。用 `dangerouslySetInnerHTML` 搭配未消毒的用戶輸入？恭喜你，一個 `<img onerror>` 就能偷走所有 cookie。修復其實很簡單：把 `<` 換成 `\u003c`，但這種基礎中的基礎居然在 React 核心裡躺了這麼久，想想就後怕。

**重點：**
- Server Components 返回的 JSON payload 沒做 HTML entity 轉義，瀏覽器直接把 `<script>` 當真的執行
- 修復方式：序列化時強制將 `<`、`>`、`/` 替換為 Unicode escape sequence
- 但是... 就算 React 修了，你還是得在應用層做 DOMPurify + CSP + HttpOnly cookie。框架幫你擋子彈，但別把防彈衣當唯一防線

### [The 49MB Web Page](https://thatshubham.com/blog/news-audit)

紐約時報首頁：422 個 network request、49MB、兩分鐘才載完。比 Windows 95（28 張磁碟片）還大。作者做了一次完整的 UX 審計，發現頁面載入的前幾秒鐘你的瀏覽器在幹嘛？跑 programmatic ad auction——Rubicon、Amazon 的競價腳本在你的 CPU 上瘋狂運算，而你連標題都還沒看到。更諷刺的是，Google 的搜尋引擎懲罰 CLS 和 intrusive interstitials，但 Google 的廣告系統正是製造這些問題的幫兇。

**重點：**
- 新聞網站實際內容只佔螢幕 11-15%，其餘全是廣告、modal、cookie banner、newsletter 彈窗
- `IntersectionObserver` 控制 sticky video、`ResizeObserver` 管理 ad slot collapse 都有現成解法，但沒人用
- 但是... 這不是工程師的鍋，是商業模式的鍋。短期 CPM 收益 vs 長期用戶留存，出版業選了前者

---

## ⚡ 一句話帶過

- **[Building a Cost-Efficient Generative UI Architecture in React Native](https://dev.to/serifcolakel/building-a-cost-efficient-generative-ui-architecture-in-react-native-2d58)** — 用 tool calling + metadata 驅動 React Native 的 Generative UI，不直接讓 LLM 吐 JSX，成本控制思路值得偷
- **[React useTransition：让 UI 更新更丝滑的并发特性](https://juejin.cn/post/7617271860845953065)** — useTransition 的完整教學，搞懂 `startTransition` 跟 `isPending` 到底在幹嘛
- **[Your Vibecoded Prototype Took 30 Minutes. Shipping It Will Take 100 Hours.](https://dev.to/adioof/your-vibecoded-prototype-took-30-minutes-shipping-it-will-take-100-hours-4bkf)** — Vibe coding 30 分鐘出原型，然後花 99 小時讓它不丟人。沒錯，聽起來很熟悉
- **[Picking a Package Manager for Your CI/CD Pipeline](https://dev.to/ymerzouka/picking-a-package-manager-for-your-cicd-pipeline-59lc)** — npm vs pnpm vs yarn 在 CI/CD 環境的實測比較，結論不意外：pnpm 贏面大
- **[End-to-End Testing with Playwright: Complete Guide with Page Object Model](https://dev.to/satish_reddybudati_42652/end-to-end-testing-with-playwright-complete-guide-with-page-object-model-3nai)** — Playwright + Page Object Model 的完整實作指南，測試架構該怎麼長就怎麼長
- **[I Built 22 Dev Tools with Zero Backend](https://dev.to/tatelyman/i-built-22-dev-tools-with-zero-backend-architecture-and-lessons-38im)** — 22 個純前端開發工具，零後端零追蹤。JSON formatter 這種輪子又被造了一遍，但至少這次不追蹤你
- **[What is NestJS and Why You Should Use It in 2026](https://dev.to/ash_dubai/what-is-nestjs-and-why-you-should-use-it-in-2026-22o2)** — NestJS 入門導覽，適合剛從 Express 散裝路由畢業想要點結構的人
- **[mini-css-extract-plugin：生产环境 CSS 提取的最佳方案](https://juejin.cn/post/7617269065950838820)** — 還在用 Webpack 的同學，CSS 提取的標準做法複習一下不虧
- **[globaly-i18n: Lightweight i18n Library for JavaScript](https://dev.to/rounaksharma19/i-built-a-lightweight-i18n-library-for-javascript-meet-globaly-i18n-eo2)** — 又一個 i18n 函式庫，支援 namespace、lazy loading、type-safe，功能倒是蠻完整
- **[LLMs can be exhausting](https://tomjohnell.com/llms-can-be-absolutely-exhausting/)** — 用 LLM 寫 code 的心理疲勞感，HN 上 33 則留言全是「我也是」

---

## 📚 慢慢啃

- **[Frontend System Design: Virtualization & Handling Large Data Sets](https://dev.to/zeeshanali0704/frontend-system-design-virtualization-handling-large-data-sets-29nf)** — 虛擬列表、無限滾動、Canvas grid 的完整架構指南。下次面試被問「如何渲染十萬筆資料」不會再傻住
- **[CSS Utilities and Generators: Build Better UI Faster](https://dev.to/padawanabhi/css-utilities-and-generators-build-better-ui-faster-2cej)** — gradient picker、shadow builder、animation generator 的工具大全，收藏起來下次切版省時間
- **[E2E Test Automation Strategy for Frontend Upgrades (Angular, React, Vue.js)](https://dev.to/arkreddysfo/e2e-test-automation-strategy-for-frontend-upgrades-angular-react-vuejs-4b68)** — 前端框架升級（React 18→19、Angular 15→16）的 E2E 測試策略，系統性地降低升級翻車風險
