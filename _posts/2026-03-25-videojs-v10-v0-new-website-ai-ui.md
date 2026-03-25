---
title: "Video.js 砍掉 88% 體積重生、v0 收編 new.website、AI 做的 UI 還是別直接上線"
date: 2026-03-25
description: "Video.js v10 四大播放器專案合體瘦身 88%、Vercel 收購 new.website 整合進 v0、AI 生成的 UI 看起來能用但真的不能直接上 production。"
tags: [frontend, tooling, ai, web-platform]
---

今天最值得聊的：一個 16 年老專案把自己砍掉重練，而且找了三個對手一起幹。Vercel 那邊又在買東西了（意外嗎？）。還有人終於把大家心裡想但不敢說的話講出來——AI 生出來的 UI，看起來很美，上線就爆。

---

## 🔧 今日硬菜

### [Video.js v10 Beta: Hello, World (again)](https://videojs.org/blog/videojs-v10-beta-hello-world-again)

這件事值得你停下來認真看。Video.js 的原作者 Steve Heffernan 在離開多年後回來了，而且不只是回來——他把 Video.js、Plyr、Vidstack、Media Chrome 四個專案的人拉在一起，從零重寫。這四個專案加起來 75,000 GitHub stars，每月處理數十億次影片播放。

重寫的成果相當驚人：預設 bundle 體積砍了 88%。React 版的 video player 只剩 18 kB gzipped，一個背景影片播放器甚至壓到 3.5 kB。他們搞了一個叫 SPF（Streaming Processor Framework）的串流架構，把以前肥滋滋的 HLS 引擎拆成可組合的模組，最簡的 ABR 串流只要 12.1 kB gzipped。

架構上完全拆分成三層：State（用 Zustand store slices）、UI（無樣式原始元件，靈感來自 Radix）、Media（可抽換的媒體層）。每一層都是可選的、可 tree-shake 的。HTML 端用 Web Components（`<video-player>`），React 端用 hooks + providers，全面支援 TypeScript 和 Tailwind。

**重點：**
- 預設 bundle 砍 88%，React video player 只要 18 kB gzipped
- 四大播放器專案史無前例的合併重寫，Plyr 作者操刀 UI 設計
- 但是... 正式版要到 2026 年中，現在是 beta，migration guide 還在寫。急著換的先等等

### [new.website joins forces with v0](https://vercel.com/blog/new-website-joins-forces-with-v0)

Vercel 又收購了。這次是 new.website，一個主打「不用寫太多 prompt 就能建出完整網站」的平台。被收進 v0 之後，他們的三個核心能力會直接整合進去：內建表單和資料庫（送出就存，不用另外設定）、SEO meta tag 設定直接在 UI 操作、內容管理和網站本身分離。

這代表什麼？v0 正在從「幫你生 UI」進化成「幫你生完整產品」。表單、資料庫、SEO、CMS——這些以前要另外接的東西，未來在 v0 裡面可能原生就有。對獨立開發者和小團隊來說，又少了幾個要自己串的服務。

**重點：**
- new.website 的表單、資料庫、SEO、CMS 能力整合進 v0
- v0 從 UI 生成器往完整產品平台演進
- 但是... Vercel 生態系越來越封閉，vendor lock-in 的老問題又浮出來了

### [Why AI-generated UI still isn't production-ready](https://dev.to/frontuna_system_0e112840e/why-ai-generated-ui-still-isnt-production-ready-2ig9)

終於有人把這件事寫出來了。AI 工具生 UI 的速度確實快到不行，但問題是：它看起來對，實際上不能用。

三個核心問題。第一，accessibility 永遠是做半套——alt text 給你塞了，但語意結構和真正的螢幕閱讀器體驗？沒有。第二，layout 在 happy path 看起來完美，但互動邏輯經不起考驗，元件不可複用，edge case 一碰就碎。第三，也是最痛的：你以為省了開發時間，結果把時間全花在「修 AI 生出來的東西」上面。

作者的結論我很認同：AI 應該幫你做到 70-80%，但不應該在過程中製造更多清理工作。這個「基礎品質層」目前還不存在。踩過 Copilot 生成程式碼然後花三小時 debug 的人，看到這篇一定很有共鳴。

**重點：**
- AI 生的 UI 在 accessibility、元件結構、edge case 處理上全面不及格
- 省下的生成時間都花在修補上，淨效益存疑
- 但是... 這不代表不該用，而是要知道它的極限在哪裡——70% 的加速器，不是 100% 的替代品

---

## ⚡ 一句話帶過

- **[Dark Mode UI Components for SaaS: Button, Input, Card, Badge (Tailwind CSS v4)](https://dev.to/huangyongshan46a11y/dark-mode-ui-components-for-saas-button-input-card-badge-tailwind-css-v4-1749)** — Tailwind CSS v4 的 dark mode 元件實作，四個最常用的 SaaS 元件一次搞定，實用但不驚艷
- **[Next.js 16 SaaS Database Schema with Prisma](https://dev.to/huangyongshan46a11y/nextjs-16-saas-database-schema-with-prisma-users-subscriptions-and-ai-chat-18gd)** — Next.js 16 + Prisma 的 SaaS schema 設計，Users、Subscriptions、AI Chat 三張表的關係搞清楚就少走很多彎路
- **[手把手搭一套前端监控采集 SDK](https://juejin.cn/post/7620728823730962438)** — 用 Next.js + NestJS + Tiptap 搭前端監控 SDK，從埋點到資料收集的完整實作
- **[Zero-config Cesium.js in Vite — introducing vite-plugin-cesium-engine](https://dev.to/jfayot/zero-config-cesiumjs-in-vite-introducing-vite-plugin-cesium-engine-1daa)** — Cesium.js 配 Vite 的痛苦（複製 WASM、設 base URL、注入 CSS），這個 plugin 全部自動化了
- **[How to Implement Google OAuth 2.0 in Next.js with NestJS](https://dev.to/marwanzaky/how-to-implement-google-oauth-20-in-nextjs-with-nestjs-3pnh)** — Next.js + NestJS 的 Google OAuth 實作教學，從 Google Console 到前後端串接一步步來
- **[I Built an API That Generates OG Images in 50ms — No Puppeteer Needed](https://dev.to/narender_singh_6c6e271c67/i-built-an-api-that-generates-og-images-in-50ms-no-puppeteer-needed-336p)** — 不靠 Puppeteer 的 OG image 生成，50ms 回應，省掉 200MB Chrome binary 的肥胖
- **[Enterprise teams can now set their default build machine](https://vercel.com/changelog/enterprise-teams-can-now-set-their-default-build-machine)** — Vercel Enterprise 可以統一設定預設 build machine 了，新建專案自動套用，管大團隊的人會感謝
- **[I scanned 100 AI-generated apps for security vulnerabilities](https://dev.to/tgoldi/i-scanned-100-ai-generated-apps-for-security-vulnerabilities-heres-what-i-found-1l5o)** — 掃了 100 個 AI 生成的 app，結果不意外——安全漏洞遍地開花，Cursor、Bolt、v0 通通中標
- **[A Streamer Built a Social Network With AI for $40. It Was Hacked in Hours.](https://dev.to/solobillions/a-streamer-built-a-social-network-with-ai-for-40-it-was-hacked-in-hours-1hng)** — 義大利實況主花 40 歐用 AI 蓋了社群網站，幾小時就被駭穿，vibe coding 的最佳反面教材
- **[Context7 MCP Server — Real-Time Library Docs for AI Coding Agents](https://dev.to/grove_chatforest/context7-mcp-server-real-time-library-docs-for-ai-coding-agents-40k3)** — 2026 年最火的 MCP server（50K stars），解決 AI coding 工具幻覺 API 的痛點，即時拉真實文件

---

## 📚 慢慢啃

- **[Is anybody else bored of talking about AI?](https://blog.jakesaunders.dev/is-anybody-else-bored-of-talking-about-ai/)** — HN 上 473 分、338 則留言的長文，講的是很多工程師心裡的真實感受：AI 話題疲勞。值得慢慢看留言區，裡面比文章本身精彩
- **[From Crutches to Bijection: How I Wrote a Sudoku Generator in JS](https://dev.to/__arehspb/from-crutches-to-bijection-how-i-wrote-a-sudoku-generator-in-js-1j47)** — 泰國的數學老師用 JavaScript 寫數獨生成器，從暴力法一路優化到雙射映射，數學和程式碼的優雅交會
- **[How I built a practical agent skill that turns rough READMEs into polished project docs](https://dev.to/debs_obrien/how-i-built-a-practical-agent-skill-that-turns-rough-readmes-into-polished-project-docs-2mef)** — Claude Code agent skill 的實戰教學，從簡單範例到能把粗糙 README 變成精美文件的完整流程
- **[How I Built 7 AI Agents That Bring Discipline to VS Code Copilot](https://dev.to/bitfrog/the-copilot-workflow-handoffs-not-chaos-1598)** — 把 VS Code Copilot 從一個什麼都做的通才拆成七個專精 agent 各司其職，workflow 設計值得參考
