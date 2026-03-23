---
title: "TC39 Signals 要統一響應式、五個正在消失的前端技能、Node.js 任務佇列踩坑指南"
date: 2026-03-23
description: "TC39 Signals 提案深度解析、前端技能淘汰清單比你想的更殘酷、Node.js 背景任務用記憶體存你是在賭命——今日三篇硬菜加十則快訊。"
tags: [frontend, typescript, nodejs, tooling, ai]
---

今天的重頭戲是 TC39 Signals 提案。如果你寫前端超過三年，你一定經歷過 React Hooks 的依賴陣列地獄、Vue 和 React 狀態不能互通的無奈。Signals 想從語言層級解決這件事。另外兩篇也值得看：一篇告訴你哪些前端技能正在被淘汰（不是 jQuery），一篇告訴你為什麼 Node.js 的 fire-and-forget 在 production 是慢性自殺。

---

## 🔧 今日硬菜

### [2026年最值得关注的JavaScript新特性：Signals，响应式编程的下一个十年](https://juejin.cn/post/7619658215228325903)

TC39 正在討論把 Signals 納入 JavaScript 語言標準，目前還在 Stage 1，但這件事的影響力不亞於當年 Promise 進標準。簡單講，Signal 就是一個「會自動通知訂閱者」的狀態容器——你讀它的時候自動訂閱，改它的時候自動通知。三個核心 API：`Signal.State`（可變狀態）、`Signal.Computed`（派生計算）、`Signal.subtle.Watcher`（觀察者）。

這篇文章把 Signals 的內部運作講得很清楚：依賴追蹤靠全域執行上下文，更新傳播用 Push-Pull 混合模型避免菱形依賴的 glitch 問題。最讓人興奮的是跨框架狀態共享——同一份 Signal store 可以同時在 React 和 Vue 裡用，不用再寫 adapter 寫到懷疑人生。

**重點：**
- 自動依賴追蹤，徹底告別 `useEffect` 的依賴陣列地獄
- 細粒度更新是預設行為，不是 React 那種整個元件重新跑
- 但是... Stage 1 離 Stage 4 還很遠，API 隨時可能改。現在想體驗可以用 `@preact/signals-core` 當 polyfill，但別在 production 用

### [The Frontend Skills That Are Actually Dying (Not the Ones You Think)](https://dev.to/web_dev-usman/the-frontend-skills-that-are-actually-dying-not-the-ones-you-think-2pnj)

每個論壇都有人在喊「jQuery 已死」「PHP 已死」，但真正在消失的技能清單比這殘酷得多，而且跟你的履歷直接相關。作者列了五個：手刻 CSS（被 Tailwind + Design Tokens 取代）、跨瀏覽器 CSS hack（IE 已死 + Interop 計畫）、從零手刻 UI 元件（shadcn/ui、Radix 做得比你好）、jQuery 特定模式（`$.ajax` 被 Fetch API 取代）、手動 Webpack/Babel 設定（Vite 預設就夠了）。

踩過這些坑的人都知道，這些曾經是「真正的技能」。但工具鏈的進化把整個工作類別吃掉了，不是你不努力，是這個工種不存在了。

**重點：**
- jQuery 本身沒死（75% 網站還在用），但「jQuery 專家」這個職位死了
- 手刻元件寫在履歷上如果沒有上下文，現在讀起來是 legacy signal
- 但是... 底層原理還是要懂。Tailwind 壞掉的時候，會修 CSS 的人才是英雄

### [The Silent Job Loss: Why Your Node.js SaaS Needs a Persistent Task Queue](https://dev.to/siddhant_jain_18/the-silent-job-loss-why-your-nodejs-saas-needs-a-persistent-task-queue-5cih)

經典場景：使用者付了錢，Stripe webhook 觸發，你 fire-and-forget 一個非同步任務去產生報告。30 秒後你部署了 hotfix。報告永遠沒產生，使用者被扣了錢，三天後你才從客服工單知道。這不是假設情境，這是每個用記憶體存任務的 Node.js 後端的預設行為。

文章從原子性 claim 問題講到 Postgres 的 `FOR UPDATE SKIP LOCKED`、Redis Lua script 的原子操作、指數退避加 jitter 避免 Thundering Herd、dead-letter log 的結構化輸出，最後用完整的 crash recovery 測試收尾。567 個測試、93.13% 覆蓋率，這不是紙上談兵。

**重點：**
- `FOR UPDATE SKIP LOCKED` 是 Postgres 任務佇列的標準解法，Redis 用 Lua script 達到同等保證
- 指數退避一定要加 jitter，不然所有 worker 同時 retry 就是 DDoS 自己
- 但是... 這篇最後在推銷自家的 KeelStack Engine。核心概念紮實，但你可能更想用 BullMQ 或 Quirrel

---

## ⚡ 一句話帶過

- **[栗子前端技术周刊第 121 期 - Vitest 4.1、Nuxt 4.4、Next.js 16.2...](https://juejin.cn/post/7619589568713916422)** — Vitest 支援 Vite 8 了、Nuxt 基於 Vue Router v5、Next.js 16.2 繼續修補，版本號跑得比你的 side project 還快
- **[Why I Ditched Next.js and Rebuilt My Site with Astro](https://dev.to/waseemrajashaik/why-i-ditched-nextjs-and-rebuilt-my-site-with-astro-2moh)** — 從 Next.js 15 + React 19 + framer-motion 砍到只剩 Astro 6，最好的程式碼就是不送到瀏覽器的程式碼
- **[Uh oh... Cloudflare just turned evil](https://dev.to/fabianfrankwerner/uh-oh-cloudflare-just-turned-evil-42pc)** — 花了十年擋爬蟲的公司現在自己出了一個爬蟲工具，這劇情轉折比 Netflix 還精彩
- **[Recreating Windows XP in React: Why Devs Keep Building OS Clones](https://dev.to/alanwest/recreating-windows-xp-in-react-why-devs-keep-building-os-clones-1jip)** — React + TypeScript 像素級復刻 Windows XP，沒有實用價值但讓人忍不住按 Star
- **[VSCode doesn't save your open tabs when you switch Git branches. I made a fix.](https://dev.to/harug/vscode-doesnt-save-your-open-tabs-and-positions-when-you-switch-git-branches-i-made-a-fix-19c9)** — 切分支丟 tab 這個痛點終於有人受不了了，開源 extension 直接解決
- **[Claude Code Secretly Hoards 140+ Config Files Behind Your Back](https://dev.to/ithiria894/claude-code-secretly-hoards-140-config-files-behind-your-back-heres-how-to-take-control-2dlb)** — 用了一週 Claude Code 發現 `~/.claude/` 裡躺了 140 個檔案，AI 比你還會囤積
- **[搞懂 Cursor 后，我一行代码都不敲了《实战篇》](https://juejin.cn/post/7619674716338536454)** — Cursor 系列最終篇，講團隊怎麼一起用 AI 寫 code，Rules 和 Skills 的協作實踐
- **[Introducing MoltenDB: A Local-First, Pure Rust Database for the Browser and Server](https://dev.to/maximilian_both_50a58f2dc/introducing-moltendb-a-local-first-pure-rust-database-for-the-browser-and-server-5ael)** — Rust 寫的、跑在瀏覽器裡的 local-first 資料庫，Alpha 階段但方向對了
- **[Inception Mercury 2 is live on AI Gateway](https://vercel.com/changelog/mercury-2-on-ai-gateway) / [Grok 4.20 on AI Gateway](https://vercel.com/changelog/grok-4-20-on-ai-gateway)** — Vercel AI Gateway 又加了兩個模型，Mercury 2 主打低延遲 agentic loop，Grok 4.20 三個變體一次到位
- **[Building an Autonomous Coding Assistant: A LangGraph.js Capstone Guide](https://dev.to/programmingcentral/building-an-autonomous-coding-assistant-a-langgraphjs-capstone-guide-1c47)** — 用 LangGraph.js 從零打造能讀 codebase、跑終端機的自主 coding agent，教學級完整度

---

## 📚 慢慢啃

- **[Teaching Claude to QA a mobile app](https://christophermeiklejohn.com/ai/zabriskie/development/android/ios/2026/03/22/teaching-claude-to-qa-a-mobile-app.html)** — 讓 Claude 自動 QA 行動 App 的完整實踐紀錄，從截圖辨識到操作流程，讀完你會知道 AI 測試的天花板在哪裡（HN 61 分不是白來的）
- **[Code has logic. It does not have meaning.](https://dev.to/krzysztofdudek/code-has-logic-it-does-not-have-meaning-11kl)** — 為什麼加入新專案三週了還是看不懂 code？因為沒人寫下「為什麼」。這篇講的是 repo 需要語意記憶而不是更大的 context window
- **[Beyond the Chatbot: Engineering a Hybrid AI Math Tutor for the Future](https://dev.to/devadhithiya/beyond-the-chatbot-engineering-a-hybrid-ai-math-tutor-for-the-future-86g)** — React + Vite 架構的 AI 數學教學系統，離線能力、隱私保護、prompt injection 防護全部考慮到了，教育場景的前端工程比你想的複雜
