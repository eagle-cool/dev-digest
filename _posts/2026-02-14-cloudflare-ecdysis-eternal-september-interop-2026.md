---
title: "Cloudflare 開源 Rust 零停機重啟、開源永恆九月危機、Interop 2026 啟動"
date: 2026-02-14
description: "Cloudflare 開源五年實戰的 Rust 零停機重啟函式庫 ecdysis，GitHub 正面回應 AI 時代開源維護者的困境，Interop 2026 帶來 20 項跨瀏覽器互通性改進。"
tags: [systems, opensource, frontend]
---

今天最大的亮點是 Cloudflare 把內部用了五年的 Rust 零停機重啟神器開源了，同時 GitHub 終於正式回應 AI 生成的低品質貢獻正在壓垮開源維護者的問題。Web 開發者這邊也有好消息——Interop 2026 宣布了 20 項跨瀏覽器互通性的重點改進項目，CSS 和 Web API 的一致性將在今年大幅提升。

---

## 🔥 今日焦點

### [Shedding old code with ecdysis: graceful restarts for Rust services at Cloudflare](https://blog.cloudflare.com/ecdysis-rust-graceful-restarts/)

Cloudflare 正式開源了 [ecdysis](https://github.com/cloudflare/ecdysis)——一個在生產環境運行五年、用於實現 Rust 服務零停機重啟的函式庫。這不是又一個玩具專案，而是每天保護數十億請求不被中斷的核心基礎設施。ecdysis 採用經典的 fork-then-exec 模式，父程序在子程序初始化完成前持續服務，透過 named pipe 傳遞 socket 檔案描述符，確保升級過程中沒有任何連線被拒絕或中斷。

**重點：**
- 採用 fork + execve 繼承 socket 的方式，消除重啟期間的服務中斷窗口
- 原生支援 Tokio 非同步框架和 systemd 整合，API 設計簡潔
- 在 Cloudflare 330+ 資料中心、120+ 國家的生產環境中驗證了五年，每次重啟避免數十萬請求丟失

### [Welcome to the Eternal September of open source](https://github.blog/open-source/maintainers/welcome-to-the-eternal-september-of-open-source-heres-what-we-plan-to-do-for-maintainers/)

GitHub 用了一個精準的比喻——「永恆的九月」——來描述 AI 時代開源面臨的困境。生成式 AI 讓提交程式碼的成本趨近於零，但審查的成本沒有下降。curl 因為 AI 生成的垃圾安全報告而終止了 bug bounty 計畫，Ghostty 改為邀請制貢獻模式。GitHub 的回應不只是空談：他們已經推出了 repo 層級的 PR 權限控制、釘選留言、減少 +1 噪音的橫幅等功能，並且正在探索基於條件的貢獻門檻和自動分類工具。

**重點：**
- AI 讓「貢獻」成本趨近零，但維護者的審查負擔反而激增，信任基礎正在被侵蝕
- GitHub 已推出 PR 權限控制、留言釘選等即時緩解工具，PR 刪除功能即將上線
- 社群也在自救：Mitchell Hashimoto 的 Vouch 專案實作了明確的信任管理機制

### [Announcing Interop 2026](https://webkit.org/blog/17818/announcing-interop-2026/)

Interop 2026 正式公布，這是 Apple、Google、Microsoft、Mozilla 和 Igalia 連續第五年合作推動瀏覽器互通性的計畫。今年涵蓋 20 個焦點領域，其中 15 個是全新項目。對前端開發者來說最令人興奮的包括：CSS `contrast-color()` 終於要全瀏覽器支援、`attr()` 擴展到所有 CSS 屬性、Container Style Queries、Scroll-driven Animations、跨文件 View Transitions、以及 JSPI 讓 WebAssembly 更容易移植同步應用程式。

**重點：**
- 20 個焦點領域涵蓋 CSS、Web API、效能、無障礙等面向，是歷年最全面的一次
- `contrast-color()`、進階 `attr()`、Container Style Queries 等實用功能將達成跨瀏覽器一致
- Navigation API 新增 `precommitHandler`，View Transitions 擴展至跨文件場景

---

## ⚡ 快訊

- **[Automate repository tasks with GitHub Agentic Workflows](https://github.blog/ai-and-ml/automate-repository-tasks-with-github-agentic-workflows/)** — GitHub 推出 Agentic Workflows，讓 AI 代理自動處理 issue 分類、CI 失敗調查、文件更新等倉庫維護任務
- **[WebKit features for Safari 26.3](https://webkit.org/blog/17798/webkit-features-for-safari-26-3/)** — Safari 26.3 釋出，改進內容傳輸最佳化、SPA 導航控制，並修復 anchor positioning 問題
- **[crates.io: an update to the malicious crate notification policy](https://blog.rust-lang.org/2026/02/13/crates.io-malicious-crate-update/)** — crates.io 不再為每個惡意套件發布部落格文章，因多數案例無實際使用證據，噪音大於訊號
- **[New deployments with vulnerable next-mdx-remote are now blocked](https://vercel.com/changelog/new-deployments-with-vulnerable-versions-of-next-mdx-remote-are-now-blocked-by-default)** — Vercel 自動封鎖含 CVE-2026-0969 漏洞版本 next-mdx-remote 的部署，強烈建議升級
- **[Announcing Rust 1.93.1](https://blog.rust-lang.org/2026/02/12/Rust-1.93.1/)** — Rust 1.93.1 修補版本釋出，包含穩定性修正
- **[Vercel Flags is now in public beta](https://vercel.com/changelog/vercel-flags-is-now-in-public-beta)** — Vercel 內建 feature flag 管理工具進入公測，支援目標規則、用戶分群和環境控制
- **[Stale-if-error cache-control directive now supported](https://vercel.com/changelog/stale-if-error-cache-control-header-is-now-supported)** — Vercel CDN 支援 stale-if-error 指令，origin 故障時可繼續提供快取回應
- **[Advanced egress firewall filtering for Vercel Sandbox](https://vercel.com/changelog/advanced-egress-firewall-filtering-for-vercel-sandbox)** — Vercel Sandbox 新增 SNI 過濾和 CIDR 封鎖的出站網路策略控制

---

## 🔗 延伸閱讀

- **[The Death of Traditional Testing: Agentic Development Broke a 50-Year-Old Field, JiTTesting Can Revive It](https://engineering.fb.com/2026/02/11/developer-tools/the-death-of-traditional-testing-agentic-development-jit-testing-revival/)** — Meta 工程團隊深度探討 AI 輔助開發如何衝擊傳統測試方法論，提出 Just-in-Time Testing 的新範式，值得每位寫測試的工程師一讀
- **[Introducing Markdown for Agents](https://blog.cloudflare.com/markdown-for-agents/)** — Cloudflare 分析 AI 爬蟲和代理如何改變內容發現方式，探討如何用結構化 Markdown 取代傳統 SEO 策略
- **[Browserbase joins the Vercel Agent Marketplace](https://vercel.com/changelog/browserbase-joins-the-vercel-agent-marketplace)** — Browserbase 加入 Vercel Marketplace，讓 AI 代理透過 CDP 直接操作遠端瀏覽器，免去自建基礎設施的麻煩
