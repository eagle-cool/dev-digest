---
title: "ISR 快取踩坑記、SWR vs Query 怎麼選、npm 供應鏈警報"
date: 2026-04-12
description: "Next.js ISR 多 Pod 快取用 Redis 省下 90% 後端呼叫、React Query 跟 SWR 2026 年實戰比較、npm 供應鏈攻擊連 audit 都擋不住——今天三道硬菜一次上齊。"
tags: [frontend, nextjs, react, nodejs, tooling]
---

今天三道菜都不輕。Next.js ISR 在多 Pod 環境下的快取問題，踩過的人都知道有多痛；SWR 跟 React Query 的老戰場又有人交了實戰報告；然後 npm 供應鏈那邊，`npm audit` 全部 pass 的情況下照樣被打穿。週六早上配咖啡剛好。

---

## 🔧 今日硬菜

### [How We Made Next.js ISR Page Cache Efficient with Redis](https://dev.to/kasonz/how-we-made-nextjs-isr-page-cache-efficient-with-redis-42km)

ISR 在單機跑很美好，但一上 Kubernetes 多 Pod 部署，每個 Pod 各自為政——同一頁面 8 個 Pod 各 regenerate 一次、各寫一份快取、每次 rolling deploy 就是一場對自家後端的小型 DDoS。這篇把整個解法攤開來講：用 Redis 做共享快取、Redis NX lock 做寫入去重、gzip level 1 壓縮省下 77% 記憶體（只多 5ms，完全值得），最狠的是直接注入 `prerender-manifest.json` 來消滅冷啟動風暴。

**重點：**
- 共享 Redis 快取 + 寫入鎖，8 個 Pod 只需要 1 次 regeneration，後端呼叫直接砍 90%
- Gzip level 1 是甜蜜點：77% 壓縮率、<1ms 成本，比 level 6 多省的那 5% 不值得拿延遲換
- 但是⋯⋯ `prerender-manifest.json` 是 Next.js 內部實作細節，不是 public API。每次升版都要驗證，踩到地雷就是無聲退化回冷啟動模式

### [React Query vs SWR in 2026: What I Actually Use and Why](https://dev.to/whoffagents/react-query-vs-swr-in-2026-what-i-actually-use-and-why-3362)

這種比較文滿街都是，但這篇難得的誠實。作者兩套都在 production 用過，結論很直白：SWR 4KB、API 一個下午就記住，80% 場景夠用；React Query 13KB、更囉嗦但 cache seeding、optimistic update、infinite scroll 的控制粒度碾壓。最實用的是作者的 hybrid 模式——Server Component 拿初始資料（不需要任何 library），Client Component 用 SWR + `fallbackData` 做即時更新，零 loading flash。

**重點：**
- Next.js App Router + 簡單讀取 → SWR；複雜 mutation + 手動快取管理 → React Query
- Hybrid 模式才是 2026 年的正解：RSC 做首屏、SWR 做客戶端同步，bundle 最小化
- 但是⋯⋯兩套都選了就別中途換。作者原話：「Pick one and commit before you're two months in」

### [I watched Shai Hulud steal credentials from teams running npm audit](https://dev.to/mckeane_mcbrearty_77fda95/i-watched-shai-hulud-steal-credentials-from-teams-running-npm-audit-heres-the-gap-nobody-talks-1ed9)

這篇不是恐嚇文，是實打實的案例拆解。2025 年 9 月 Shai Hulud worm 打穿 PostHog、Zapier、Postman 等 500+ 個套件，2026 年 3 月 axios 也被北韓駭客搞了——重點是 **npm audit、Snyk、Dependabot 全部 pass**。因為供應鏈攻擊根本沒有 CVE 可查，preinstall hook 讀你的 `~/.npmrc` 然後 POST 出去，從 npm audit 的角度看這叫「正常運作」。作者後來做了一個行為分析掃描器，不靠 CVE 資料庫，直接看 tarball 裡的程式碼在幹嘛。

**重點：**
- npm audit 只查已知漏洞資料庫，供應鏈攻擊在被發現前沒有 advisory，所以永遠 pass
- 行為分析是正確方向：`child_process` + 網路請求 + 讀環境變數 = 高信心竊取模式
- 但是⋯⋯文章後半是在推自家產品 Dependency Guardian。概念對，但要自己評估是否信任另一個第三方工具來保護你的供應鏈

---

## ⚡ 一句話帶過

- **[Building a Figma to GitHub token pipeline that actually works](https://dev.to/alexandersstudi/building-a-figma-to-github-token-pipeline-that-actually-works-3o0o)** — 設計師改個品牌色前端要手動同步？這篇用 Figma webhook + GitHub Actions 做 design token 自動化，終於不用再追著設計師要色碼了
- **[JavaScript Error Handling: Moving Beyond Generic Catch-Alls](https://dev.to/pavkode/javascript-error-handling-moving-beyond-generic-catch-alls-for-efficient-debugging-and-resolution-fn)** — 別再 `catch(e) { console.log(e) }` 了，JS 的 error type 比你想的多，這篇分門別類講清楚
- **[How Our Designer Ships Front-End Changes Without Writing Code](https://dev.to/hsin_yenchung_5562e7d5c9/how-our-designer-ships-front-end-changes-without-writing-code-5936)** — 設計師在 Slack 打一句話，10 分鐘後 PR 帶 preview 就出來了。DX 做到這程度值得偷師
- **[What is a metaframework](https://dev.to/fyodorio/what-is-a-metaframework-4cef)** — 如果你一直分不清 framework 跟 metaframework 的差別，這篇是目前最清楚的概念整理
- **[How to Build a Scalable SaaS Platform with Next.js and PostgreSQL](https://dev.to/waseemahmad/how-to-build-a-scalable-saas-platform-with-nextjs-and-postgresql-48mh)** — 46 個專案的經驗濃縮，從 auth 到 Stripe 到多租戶架構，Next.js SaaS 的架構決策清單
- **[Building a Robust Real-Time Transcription Pipeline in Next.js](https://dev.to/nareshipme/building-a-robust-real-time-transcription-pipeline-in-nextjs-stt-streaming-and-error-recovery-2io5)** — Next.js 裡做即時語音轉文字，chunking、並發、error recovery 都有碰到，比想像中複雜
- **[I Got Tired of Copying the Same MERN Boilerplate, So I Built a CLI](https://dev.to/shivamm2606/i-got-tired-of-copying-the-same-mern-boilerplate-so-i-built-a-cli-2b88)** — 每次起新專案都在複製貼上同一套 MERN 設定？嗯，所以又有人做了 CLI。問題不新，但痛點真實
- **[How I use Claude Code to write tests for untested code](https://dev.to/subprime2010/how-i-use-claude-code-to-write-tests-for-untested-code-a-complete-workflow-32oh)** — 那個零測試的 production module，你知道我在說哪個。這篇示範用 AI 補測試的完整流程
- **[I scanned the most famous AI coding repos on GitHub. Here's what I found](https://dev.to/vibedoctor_io/i-scanned-the-most-famous-ai-coding-repos-on-github-heres-what-i-found-469l)** — 用靜態分析掃 AI 生成的程式碼，結果不意外：hallucinated imports、React hooks memory leak、N+1 query
- **[Cursor 3 Just Turned My AI Agent Workflows Into Something Actually Scalable](https://dev.to/cristiansarmiento/cursor-3-just-turned-my-ai-agent-workflows-into-something-actually-scalable-1j47)** — Cursor 3 的 Background Agent 讓平行跑多個 AI agent 變得可行，從玩具變成真正的工作流

---

## 📚 慢慢啃

- **[Why Queues Don't Fix Overload (And What To Do Instead)](https://dev.to/pmbanugo/why-queues-dont-fix-overload-and-what-to-do-instead-14on)** — 你以為加個 queue 就能擋住流量洪峰？這篇從物理定律講起，解釋為什麼 unbounded queue 本身就是 bug，以及 backpressure 的正確姿勢。做 Node.js 後端的必讀
- **[Building a Five-Layer Quality Gate for Agent-Written Code](https://dev.to/kagin007/building-a-five-layer-quality-gate-for-agent-written-code-3e0k)** — 一個 solo dev 用 Claude Code 平行開發的完整品質管控流程：五層 gate 從 lint 到 integration test。AI 寫的 code 怎麼確保品質？這篇有體系
- **[Blame-aware code review — why your AI reviewer should only flag what you changed](https://dev.to/radpdx/blame-aware-code-review-why-your-ai-reviewer-should-only-flag-what-you-changed-54he)** — AI code reviewer 吐 40 個 finding，28 個是你碰都沒碰的舊 code。blame-aware review 只看你改的部分，這才是合理的 scope
- **[MCP vs CLI for Browser Automation: I Benchmarked Both](https://dev.to/achiya-automation/mcp-vs-cli-for-browser-automation-i-benchmarked-both-and-the-results-surprised-me-4cog)** — Safari 原生 MCP automation vs CLI-Anything，實測對比 token 消耗、速度、可靠性。結論不是你想的那樣
