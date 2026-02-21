---
title: "關掉 Dependabot、通報漏洞被告、Android 開放性告急"
date: 2026-02-21
description: "Filippo Valsorda 直言 Dependabot 是噪音製造機、資安研究員通報漏洞反遭律師威脅、F-Droid 警告 Google 鎖死 Android 的計畫仍在推進。另有 Prisma vs Drizzle 冷啟動實測、CERN 重建 1989 年瀏覽器等。"
tags: [security, opensource, devops, frontend, programming]
---

今天的主題是「做對的事被懲罰」三連發。Go 生態的密碼學大佬叫你把 Dependabot 關了，一個潛水教練通報漏洞結果被律師追殺，然後 F-Droid 提醒大家 Google 鎖死 Android 的計畫根本沒取消。開心嗎？

---

## 🔧 今日硬菜

### [Turn Dependabot Off](https://words.filippo.io/dependabot/)

Filippo Valsorda——前 Go Security Team 負責人、filippo.io/edwards25519 的維護者——直接開砲：Dependabot 是個噪音製造機，讓你以為自己在做安全維護，其實只是在浪費時間。

事情是這樣的：他發了個 edwards25519 的安全修復，影響範圍是 `(*Point).MultiScalarMult` 這個幾乎沒人用的方法。結果 Dependabot 對幾千個根本不受影響的 repo 開了 PR，附帶一個莫名其妙的 CVSS 分數和 73% 的「相容性」警告——實際上 diff 就一行。更離譜的是，連不匯入受影響 package 的 repo 都收到了警報。

他的替代方案很務實：用 `govulncheck` 做靜態分析過濾，它能追蹤到 symbol 層級判斷你的程式碼是否真的呼叫了有漏洞的函式。另外用 CI 跑 `go get -u -t ./...` 測試最新依賴版本，而不是盲目升級——這樣惡意套件進到依賴鏈也只會跑在 CI 沙箱裡，不會直接進 production。

**重點：**
- `govulncheck` 比 Dependabot 精準得多，支援 package 甚至 symbol 層級的可達性分析
- 每天在 CI 跑最新依賴的測試，比自動開 PR 升級更安全也更實際
- 但是... 這套方案目前主要適用於 Go 生態，npm/PyPI 等生態的工具鏈還沒這麼成熟

### [I found a Vulnerability. They found a Lawyer](https://dixken.de/blog/i-found-a-vulnerability-they-found-a-lawyer)

一個潛水教練兼 Linux 平台工程師，在哥斯大黎加的船上發現他的潛水保險公司會員入口有個離譜的漏洞：遞增的數字 user ID 加上所有帳號共用的預設密碼，沒有強制改密、沒有 rate limiting、沒有 MFA。連未成年學員的完整個資——姓名、地址、電話、出生日期——都能被任何人存取。

他按標準流程走了 responsible disclosure：先通報馬爾他的 CSIRT，再聯繫公司。結果回信的不是 IT 團隊，是公司 DPO 的律師事務所。先是威脅他違反馬爾他刑法第 337E 條（電腦犯罪），然後要他簽 NDA 在「當天下班前」回覆，最後還甩了一句「帳號安全是使用者自己的責任」。

漏洞最後修了，但公司有沒有通知受影響用戶（GDPR 第 34 條要求的）？沒有任何確認。

**重點：**
- 經典的「射殺信使」模式：回應漏洞通報的是律師而不是工程師，就知道這家公司的安全文化了
- GDPR 明確規定資料控制者有責任實施適當的安全措施，「使用者自己要改密碼」不是合法的免責聲明
- 但是... 這種恐嚇在很多司法管轄區仍然有效，資安研究員每次通報都像在賭博

### [Keep Android Open](https://f-droid.org/2026/02/20/twif.html)

F-Droid 在 FOSDEM26 跟用戶聊天時發現一件嚇人的事：大多數人以為 Google 已經取消了鎖死 Android 的計畫。事實是，去年八月宣布的那些限制措施仍在按計畫推進。Google 說的「進階流程」？沒人看過。Android 16 QPR2？沒有。Android 17 Beta 1？也沒有。

F-Droid 現在在官網和客戶端上放了警告橫幅。這不是恐慌——是在 Google 成為所有 Android 裝置的守門人之前，最後的呼籲。如果你在乎 sideloading、在乎替代應用商店、在乎 Android 的開放性，這件事值得你關注。

**重點：**
- Google 的 PR 策略很成功，讓大多數人以為問題已經解決，但技術限制仍在推進
- F-Droid 的存亡直接取決於 Android 是否保持開放的 sideloading 能力
- 但是... 944 個 HN upvotes、367 則留言，代表社群是醒的，但光靠開發者的聲量夠不夠撼動 Google 是另一回事

---

## ⚡ 一句話帶過

- **[6 Prisma vs Drizzle Patterns That Cut Serverless Cold Starts by 700ms](https://dev.to/jsgurujobs/6-prisma-vs-drizzle-patterns-that-cut-serverless-cold-starts-by-700ms-5dl5)** — Prisma 7 砍了 Rust engine 快了不少，但 Drizzle 的 bundle 還是小得沒道理，六個具體場景的冷啟動實測數據
- **[CERN rebuilt the original browser from 1989](https://worldwideweb.cern.ch)** — CERN 把 Tim Berners-Lee 1989 年寫的第一個瀏覽器重建了，可以直接在網頁上玩，Web 考古愛好者必訪
- **[HTML 早已不是标签了，它现在是系统级接口：这 9 个 API 直接干翻常用 JS 库](https://juejin.cn/post/7608102583496720393)** — Popover API、Navigation API、View Transitions... 原生 HTML 能做的事已經超乎你想像，jQuery 表示欣慰
- **[Wikipedia deprecates Archive.today, starts removing archive links](https://arstechnica.com/tech-policy/2026/02/wikipedia-bans-archive-today-after-site-executed-ddos-and-altered-web-captures/)** — Archive.today 對部落格發動 DDoS 還竄改網頁快照，維基百科直接把它列入黑名單，網頁存檔界的信任危機
- **[I Let Users Write HTML Templates - Here Are 6 Security Holes I Had to Patch](https://dev.to/vincentventalon/i-let-users-write-html-templates-here-are-6-security-holes-i-had-to-patch-lfi)** — 讓使用者寫 HTML 範本生成 PDF，結果 XSS、SSRF、路徑遍歷全來了，踩坑日記很實在
- **[Add `go fix` to Your CI Pipeline](https://dev.to/jcorral/addgo-fix-to-your-ci-pipeline-5426)** — `go fix` 沉睡了十幾年，現在接上 CI 可以自動處理 API 遷移，Go 1.24+ 的隱藏好物
- **[Python Just Turned 35](https://dev.to/wallaceespindola/python-just-turned-35-heres-what-kept-it-alive-all-these-years-4jh9)** — 1991 年的聖誕節 side project 活到了 2026 年，比大多數「改變世界」的框架都長壽
- **[Your Logs Are a Security Risk — 6 Patterns That Leak PII](https://dev.to/suhteevah/your-logs-are-a-security-risk-6-patterns-that-leak-pii-5jd)** — 你的 log 大概是整個系統裡最大的未稽核資料流，六種常見的 PII 外洩模式值得自查
- **[How to Lint Your Cursor Rules in CI](https://dev.to/nedcodes/how-to-lint-your-cursor-rules-in-ci-so-broken-rules-dont-ship-2n7a)** — .mdc 規則壞了 Cursor 就直接無視，現在可以在 CI 裡自動檢查 frontmatter 和 glob pattern
- **[Escaping Misconfigured VSCode Extensions (2023)](https://blog.trailofbits.com/2023/02/21/vscode-extension-escape-vulnerability/)** — Trail of Bits 的經典重溫：VSCode 擴充功能的 WebView 設定錯一個 flag 就能 RCE，最近又被翻出來討論

---

## 📚 慢慢啃

- **[Lil' Fun Langs](https://taylor.town/scrapscript-000)** — Scrapscript 作者聊小型語言設計的樂趣與哲學，如果你對「為什麼要發明新語言」這個問題有好奇心，這篇會讓你讀得很開心
- **[Clean Architecture in Kotlin: No Exceptions, No Magic, No Compromise](https://dev.to/wakita181009/clean-architecture-in-kotlin-no-exceptions-no-magic-no-compromise-5ha1)** — 5 個 Gradle module、Arrow-kt Either、顯式 DI，Spring Boot 專案如何不知不覺耦合成一坨以及怎麼拆
- **[How We Built Transcript-Powered Video Editing in Go](https://dev.to/alexneamtu/how-we-built-transcript-powered-video-editing-in-go-4p58)** — 用逐字稿驅動影片剪輯，Go 的 FFmpeg binding 實戰分享，從「嗯」的偵測到自動剪接
- **[Chapter 3: Terraform + Helm — A Better Abstraction](https://dev.to/glukas/chapter-3-terraform-helm-a-better-abstraction-5dka)** — Terraform 管 K8s 能跑但不該全管，Helm 接手 lifecycle management 的分工心得，第三篇系列但獨立可讀
