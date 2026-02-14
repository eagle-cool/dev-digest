---
title: "Go 1.26 發布、年度十大 Web 攻擊技術、DDoS 破 31 Tbps 紀錄"
date: 2026-02-14
description: "Go 1.26 帶來 Green Tea GC 和泛型自引用；PortSwigger 年度十大 Web 攻擊揭曉，側信道攻擊成主流；Cloudflare 報告 DDoS 翻倍突破 31.4 Tbps 紀錄"
tags: [systems, security, frontend, devops]
---

今天三件大事。Go 1.26 終於把 Green Tea GC 從實驗搬進預設了（效能黨可以慶祝），PortSwigger 的年度十大 Web 攻擊技術出爐——今年側信道攻擊占了一堆席位，然後 Cloudflare 告訴你 2025 年 DDoS 攻擊翻了一倍，最大那發打到 31.4 Tbps。情人節看這些比巧克力實在。

---

## 🔧 今日硬菜

### [Go 1.26 is released](https://go.dev/blog/go1.26)

Go 1.26 是一次紮實的大版本。最讓人眼前一亮的是 Green Tea GC 正式轉正——之前在 1.25 還是實驗性質，現在預設啟用。cgo 呼叫的 overhead 降了約 30%，對那些 Go/C 混合的 codebase 來說是實打實的效能提升。語言層面，`new()` 終於可以帶初始值了（`new(int64(300))` 取代兩行的 `x := ...; ptr := &x`），泛型也支援自引用型別參數。`go fix` 整個用 analysis framework 重寫了，附帶一堆 modernizer 幫你自動升級舊寫法。

**重點：**
- Green Tea GC 預設啟用 + cgo overhead 降 30% + slice 更多情況分配在 stack 上
- 實驗性功能：SIMD package、`runtime/secret`（安全擦除敏感資料）、goroutine leak profiling
- 但是... 三個新 crypto package（`crypto/hpke`、`crypto/mlkem`）暗示 Go 在後量子密碼學佈局，升級前先確認你的 CI 能跑過

### [Top 10 web hacking techniques of 2025](https://portswigger.net/research/top-10-web-hacking-techniques-of-2025)

PortSwigger 第 19 屆年度評選，由社群提名 + 專家評審選出。今年最大的趨勢是側信道攻擊（side-channel）成為核心攻擊手法——十個裡面有兩個是 XS-Leak。冠軍是 Vladislav Korchagin 的 error-based blind SSTI 技術，把 SQL injection 時代的 error-based 思路搬到 template injection，配上 polyglot 偵測和開源工具，可能開啟 SSTI 的新紀元。亞軍 ORM Leak 則把 ORM 層的資訊洩漏從框架特定漏洞提升為通用方法論——SQL injection 漸漸少了，但資料照樣能 dump。

**重點：**
- #1 Error-based blind SSTI：老技術新戰場，polyglot 偵測 + 開源工具鏈
- #3 SSRF via HTTP redirect loops：簡單、優雅、致命——把 blind SSRF 變成 visible
- 但是... James Kettle 觀察到提名數從去年 121 降到 63，「可能因為大家都被 AI 分心了」——資安研究的注意力正在被稀釋

### [2025 Q4 DDoS threat report: A record-setting 31.4 Tbps attack](https://blog.cloudflare.com/ddos-threat-report-2025-q4/)

數字會說話：2025 年 DDoS 攻擊總量 4,710 萬次，比 2024 翻倍，比 2023 暴增 236%。平均每小時自動擋下 5,376 次攻擊。最誇張的是 Aisuru-Kimwolf 殭屍網路——由 100 到 400 萬台被感染的 Android TV 組成——在聖誕前夕發動「The Night Before Christmas」攻勢，HTTP DDoS 峰值打到 2.05 億 rps。那個 31.4 Tbps 的紀錄？只持續了 35 秒。攻擊來源方面，孟加拉取代印尼成為最大來源，香港被攻擊排名跳了 12 名到第二，英國更是狂飆 36 名到第六。

**重點：**
- Aisuru-Kimwolf 殭屍網路主力是 Android TV，估計 100-400 萬台——你家的智慧電視可能正在打別人
- 網路層攻擊三倍增長，雲端平台（DigitalOcean、Azure、Tencent）成主要攻擊跳板
- 但是... Cloudflare 說這些都是「自動擋下的」，報告本質上也是產品廣告——不過數據本身確實值得警惕

---

## ⚡ 一句話帶過

- **[Improve global upload performance with R2 Local Uploads](https://blog.cloudflare.com/r2-local-uploads/)** — R2 上傳現在先寫到離你最近的節點再非同步複製，延遲直接砍掉大半，做全球化應用的可以關注
- **[Release Notes for Safari Technology Preview 237](https://webkit.org/blog/17842/release-notes-for-safari-technology-preview-237/)** — WebKit 又在默默推進，寫前端的養成定期看 TP release notes 的習慣不會虧
- **[Introducing new token formats and secret scanning](https://vercel.com/changelog/new-token-formats-and-secret-scanning)** — 把 Vercel API key 不小心推到 public repo？現在會自動撤銷。早該有了
- **[The Vercel OSS Bug Bounty program is now available](https://vercel.com/blog/the-vercel-oss-bug-bounty-program-is-now-available)** — Next.js、Turborepo 等開源專案現在有正式的 bug bounty，找到洞有錢拿
- **[Introducing Geist Pixel](https://vercel.com/blog/introducing-geist-pixel)** — Geist 字體家族新成員，像素風格的 bitmap typeface，做 retro UI 的設計師會愛
- **[Workflow 4.1 Beta: Event-sourced architecture](https://vercel.com/changelog/workflow-event-sourcing)** — Vercel Workflow 底層改用 event sourcing，狀態重建靠 replay event log——踩過分散式狀態坑的都知道這方向是對的
- **[GitHub availability report: January 2026](https://github.blog/news-insights/company-news/github-availability-report-january-2026/)** — 一月兩次事故，其中 Copilot 掛了 46 分鐘錯誤率飆到 100%。依賴 AI 寫 code 的那天你在幹嘛？
- **[Zero-configuration support for Koa](https://vercel.com/changelog/zero-configuration-support-for-koa)** — Koa 終於能零設定部署到 Vercel 了，雖然 2026 年還在用 Koa 的人可能不多
- **[PostHog joins the Vercel Marketplace](https://vercel.com/changelog/posthog-joins-the-vercel-marketplace)** — PostHog 原生整合 Vercel，feature flags + A/B testing 一站搞定

---

## 📚 慢慢啃

- **[Interop 2025: A year of convergence](https://webkit.org/blog/17808/interop-2025-review/)** — 四年了，Interop 計畫終於把各大瀏覽器的相容性推到令人滿意的程度。這篇年度回顧告訴你哪些 CSS/Web API 現在可以放心用，哪些還要再等等
- **[No Display? No Problem: Cross-Device Passkey Authentication for XR Devices](https://engineering.fb.com/2026/02/04/security/cross-device-passkey-authentication-for-xr-devices-meta-quest/)** — Meta 工程團隊解決了一個有趣的認證問題：VR 頭盔沒辦法掃 QR code，怎麼做跨裝置 passkey？這篇的 FIDO2 實作細節值得搞身份認證的人細讀
- **[Making agent-friendly pages with content negotiation](https://vercel.com/blog/making-agent-friendly-pages-with-content-negotiation)** — 用 HTTP Accept header 讓同一個 URL 對人類送 HTML、對程式送乾淨的結構化文字。Content negotiation 不是新概念，但這個應用場景值得前端架構師想一想
