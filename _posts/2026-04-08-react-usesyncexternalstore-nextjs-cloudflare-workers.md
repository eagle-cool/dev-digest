---
title: "React 被低估的 Hook、Next.js 搬上 Cloudflare、五個該停的壞習慣"
date: 2026-04-08
description: "深入解析 useSyncExternalStore 為何是 React 18 最被忽略的武器，Next.js monorepo 從 Firebase 搬到 Cloudflare Workers 的實戰踩坑，以及五個讓你 React 專案越寫越爛的壞習慣。"
tags: [frontend, react, nextjs, typescript, tooling]
---

今天的硬菜很 React。一個被 90% 開發者忽略的 Hook 其實是狀態管理的核心原語，有人把 Next.js monorepo 從 Firebase 搬到 Cloudflare Workers 月費只要 $5，然後一篇很實在的「我不再做這五件事之後程式碼變好了」值得每個寫 React 的人看一遍。

---

## 🔧 今日硬菜

### [useSyncExternalStore — The React Hook You Didn't Know You Needed](https://dev.to/mehta0007/usesyncexternalstore-the-react-hook-you-didnt-know-you-needed-34mp)

這篇把 `useSyncExternalStore` 從「library 作者才會用的冷門 Hook」拉到「你現在就該學會的基礎原語」，寫得很到位。核心問題是 React 18 的 Concurrent Mode 會讓兩個元件在不同時間點讀取外部狀態，造成 UI 撕裂（tearing）——A 元件讀到舊值、B 元件讀到新值，畫面不一致。這個 Hook 就是 React 團隊給的正式解法。

文章從 browser API 訂閱（`visibilitychange`、`navigator.onLine`）講到自己用 40 行蓋一個 mini state manager，基本上就是 Zustand 的核心原理。最實用的部分是拿 `useState` + `useEffect` 的傳統寫法對比，清楚指出三個問題：初始值可能過期、Concurrent Mode 下會撕裂、SSR 直接爆炸。

**重點：**
- `subscribe` 函數要穩定引用（定義在元件外面或用 `useCallback`），否則每次 render 都重新訂閱，直接無窮迴圈
- `getSnapshot` 不能每次都回傳新物件——React 用 `Object.is` 比較，新 reference = 無窮 re-render
- 但是⋯如果你只是包裝 Zustand 或 Redux，這些 library 內部已經幫你做了，你不需要自己碰這個 Hook

### [Deploying a Next.js Monorepo to Cloudflare Workers: Lessons from the Trenches](https://dev.to/lewiskori/deploying-a-nextjs-monorepo-to-cloudflare-workers-lessons-from-the-trenches-1ok8)

從 Firebase Hosting 搬到 Cloudflare Workers，三個 Next.js 16 app + Nx 22 + pnpm monorepo，月費從不可預測降到 $5。這篇最有價值的不是「Cloudflare 好棒棒」，而是那些沒人跟你說的坑。

`compatibility_date` 設太舊的話 `process.env` 是空的——你的 Zod schema parse 會在 runtime 炸掉但 error message 根本看不出原因。Middleware 裡不能用 `next/headers` 的 `cookies()`——本地 `next dev` 跑得好好的，部署上去才爆。Free plan 的 3 MiB gzip 限制，塞個 Sentry server-side 就超標。這些都是只有實際踩過才會知道的東西。

最值得注意的是 Next.js 16.2 的 Adapter API 已經 stable 了，Vercel 自己的 adapter 也是用同樣的 public API。這代表 `@opennextjs/cloudflare` 未來不用再逆向工程 build output，而是有正式合約可以依賴。

**重點：**
- `@opennextjs/cloudflare` 是讓 Next.js 跑在 Workers 上的關鍵 adapter，設定意外地簡單
- Wrangler 的 `env` 設定不會繼承上層的 `vars` 和 `services`，staging 環境要自己全部重寫一遍
- 但是⋯preview deployment 共用 production 環境變數這點還是很痛，得靠 wrangler environments 手動解決

### [What I Stopped Doing in React Projects (and Why My Code Got Better)](https://dev.to/harkanday/what-i-stopped-doing-in-react-projects-and-why-my-code-got-better-ifj)

這篇的核心觀點很簡單但很少人做到：程式碼變差不是因為做太少，而是做太多。作者在 production 跑了半年的行銷平台，整理出五個他停止做的事情——用 Redux 管 server state（改用 SWR，少了 85% 程式碼）、自己在前端實作 token refresh（300 行 race condition 地獄換成 10 行「401 就登出」）、到處寫 `user.role === 'admin'`（改成 `hasPermission(resource, action)`，新增角色零改動）、寫 snapshot test（改成行為驅動的 integration test）、把 business logic 塞進 Hook（抽到純 TypeScript service 層）。

每一點都不是新觀點，但放在同一篇裡、配上 before/after 對比和具體數字，說服力很夠。特別是 snapshot test 那段——「改了 CSS class 就掛、但 delete 按鈕不呼叫 service 卻不會掛」，這就是為什麼 snapshot test 是假安全感。

**重點：**
- 把 fetch 邏輯放在純 TypeScript service 層，Hook 只負責「協調」——需求變更時改 service 就好，零元件改動
- 權限從 role-based 改成 resource-action 模型，前端只消費後端給的權限表
- 但是⋯「401 就登出」這招不適用於所有場景，長時間操作的應用還是得做 token refresh

---

## ⚡ 一句話帶過

- **[TypeScript Tricks I Actually Use Day to Day](https://dev.to/abdulmalik_muhammad/typescript-tricks-i-actually-use-day-to-day-3a1l)** — Discriminated unions、template literal types、satisfies 關鍵字，都是真的天天在用的東西，不是面試才背的
- **[use-local-llm: React Hooks for AI That Actually Work Locally](https://dev.to/pooyagolchian/use-local-llm-react-hooks-for-ai-that-actually-work-locally-28mi)** — 把本地跑的 LLM 包成 React Hook，不用 server、不走 OpenAI，streaming 和 abort 都有處理
- **[Agent-Driven E2E Testing with Cypress: A Practical Guide to Harness Engineering with Cursor Subagents](https://dev.to/cypress/agent-driven-e2e-testing-with-cypress-a-practical-guide-to-harness-engineering-with-cursor-5fob)** — Cypress 官方出的 AI agent 測試指南，把散落在人腦裡的 domain knowledge 餵給 agent 寫 E2E
- **[Ionify vs Vite: What Actually Happens Inside Your Build Tool](https://dev.to/khaledmsalem/ionify-vs-vite-what-actually-happens-inside-your-build-tool-2end)** — 挑戰 Vite「每次從 entry point 開始建置」的預設，提出增量建置的思路，概念有趣但生態系支援是問號
- **[Zephyr Events – A 2KB TypeScript Event Emitter That's Race-Condition Safe](https://dev.to/bogdan_bododumitrescu_/zephyr-events-a-2kb-typescript-event-emitter-thats-race-condition-safe-4ng2)** — 修了 EventEmitter3 和 mitt 都有的 bug：handler 在 emit 中呼叫 off() 會跳過下一個 handler
- **[Why Cursor Keeps Setting CORS to * (And How to Fix It)](https://dev.to/chandan_karn_fb750e731394/why-cursor-keeps-setting-cors-to-and-how-to-fix-it-i4n)** — AI 生成的 Express 後端幾乎都是 `origin: '*'`，因為訓練資料就長這樣，五分鐘修好的事別拖
- **[Your Accessibility Score Is Lying to You](https://dev.to/chris_devto/your-accessibility-score-is-lying-to-you-5fh2)** — axe-core 和 Lighthouse 只是拼字檢查等級的 a11y 工具，真正的無障礙測試還是得靠 screen reader 手動驗
- **[NodeJS Authentication Library](https://dev.to/noorix1/nodejs-authentication-library-5fdd)** — nauth-toolkit，支援 NestJS/Express/Fastify + React/Angular 前端，自架在你自己的 PostgreSQL 上，免月費
- **[vue-multiple-themes v4: Dynamic Multi-Theme Support for Vue 2 & 3](https://dev.to/pooyagolchian/vue-multiple-themes-v4-dynamic-multi-theme-support-for-vue-2-3-p8g)** — 不只 dark/light，支援季節主題、品牌色、a11y 高對比，Vue 生態少有的完整主題方案
- **[dotenv-audit v1.1.0 — now with .env.example auto-sync](https://dev.to/akashguptasky/dotenv-audit-v110-now-with-envexample-auto-sync-18ei)** — 自動掃你程式碼裡用到但 .env 沒定義的變數，新增 sync 指令自動更新 .env.example

---

## 📚 慢慢啃

- **[WebAssembly 3.0 with .NET: The Future of High-Performance Web Apps in 2026](https://dev.to/vikrant_bagal_afae3e25ca7/webassembly-30-with-net-the-future-of-high-performance-web-apps-in-2026-3mg7)** — Wasm 從「能不能用」進化到「該用在哪」的階段，這篇整理了 Blazor 生態的成熟度和 Wasm 在 edge computing 的新可能
- **[Screen Reader Testing in 5 Minutes: A Developer's Quick Start Guide](https://dev.to/agentkit/screen-reader-testing-in-5-minutes-a-developers-quick-start-guide-27l7)** — 第一次開 VoiceOver 就想 force quit 的人必讀，五分鐘入門讓你至少知道自己的網站在 screen reader 裡長什麼樣
- **[Why I Built a SvelteKit + FastAPI SaaS Boilerplate (and Open-Sourced the Starter)](https://dev.to/quartalis/why-i-built-a-sveltekit-fastapi-saas-boilerplate-and-open-sourced-the-starter-4ne1)** — 市面上 SaaS boilerplate 全是 Next.js，這位老兄用 SvelteKit 5 + FastAPI + PostgreSQL 蓋了一個，技術選型的思路值得參考
- **[Why Angular Still Fits Enterprise Frontend Architecture Better Than Most Trends](https://dev.to/maninderpreet_singh/why-angular-still-fits-enterprise-frontend-architecture-better-than-most-trends-4g6e)** — 不是來打框架戰的，認真分析了為什麼大型組織需要 Angular 那套「不自由但一致」的結構，適合在選框架前冷靜讀一遍
