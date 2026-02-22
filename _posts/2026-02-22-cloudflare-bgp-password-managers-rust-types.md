---
title: "Cloudflare BGP 踩雷、密碼管理器被打臉、Rust 型別驅動設計"
date: 2026-02-22
description: "Cloudflare 一個 query parameter 沒給值就把客戶的 BGP prefix 全刪了，ETH Zurich 打臉三大密碼管理器的零知識加密承諾，還有一篇用 Rust 講 Parse Don't Validate 的好文。"
tags: [systems, security, frontend, devops, opensource]
---

今天的重頭戲是 Cloudflare 又搞出一次讓人冒冷汗的事故——一個 query string 的值沒傳好，就把客戶的 BGP route 全撤了。另外 ETH Zurich 的研究團隊直接打臉三大密碼管理器的「零知識加密」承諾，還有一篇把經典的 Parse Don't Validate 用 Rust 重新講一遍的好文，值得細讀。

---

## 🔧 今日硬菜

### [Cloudflare outage on February 20, 2026](https://blog.cloudflare.com/cloudflare-outage-february-20-2026/)

這次事故的根因簡單到會讓你懷疑人生。Cloudflare 的 BYOIP（Bring Your Own IP）清理任務在呼叫內部 API 時，把 `pending_delete` 這個 query parameter 傳了但沒給值。API 端用 `Query().Get("pending_delete")` 拿到空字串，直接跳過了「只回傳待刪除 prefix」的邏輯，改為回傳所有 BYOIP prefix。然後清理任務就很盡責地開始把它們全部刪掉。

1,100 個 BGP prefix 被撤回，25% 的 BYOIP 客戶受影響，整起事故持續 6 小時 7 分鐘。連 1.1.1.1 的網站都噴 403。

**重點：**
- 根因是 API 的 query parameter 處理邏輯：空字串 vs 未傳的語義混淆，這種 bug 在任何語言的 web framework 都可能踩到
- 恢復為什麼慢：受影響的 prefix 狀態不一致，有的只是被撤回廣播、有的連 service binding 都被刪了，需要逐批手動修復
- 但是... 這是 Code Orange: Fail Small 計畫的一部分——諷刺的是，為了讓部署更安全而做的改動，本身就造成了大規模事故。staged rollout 機制還沒上線就先炸了

### [Parse, Don't Validate and Type-Driven Design in Rust](https://www.harudagondi.space/blog/parse-dont-validate-and-type-driven-design-in-rust/)

經典的「Parse, Don't Validate」概念被用 Rust 重新詮釋了一遍，而且這次不用 Haskell（終於）。核心觀點很直接：與其在函式裡不斷做 validation 然後回傳 `Option`，不如用 newtype 把不變量（invariant）編碼進型別系統裡。`NonZeroF32`、`NonEmptyVec` 這些例子讓你只需要在邊界驗證一次，之後整條呼叫鏈都不用再擔心。

文章從除以零的簡單例子一路講到 `serde_json` 的反序列化、shotgun parsing 的安全隱患，層次清楚。如果你寫 Rust 還在到處撒 `unwrap()` 和 `if x.is_empty()`，這篇該看。

**重點：**
- 「弱化回傳型別」vs「強化參數型別」——兩種處理錯誤的哲學，後者讓驗證只做一次
- `String` 本身就是 `Vec<u8>` 的 newtype，`from_utf8` 就是 parse 函式——你早就在用了
- 但是... Rust 的 newtype 缺乏 delegation 語法糖，包一層就要手動實作一堆 trait，這是實務上的最大阻力

### [Password managers less secure than promised](https://ethz.ch/en/news-and-events/eth-news/news/2026/02/password-managers-less-secure-than-promised.html)

ETH Zurich 的 Applied Cryptography Group 對 Bitwarden、LastPass、Dashlane 做了系統性安全分析，結果不太好看。他們在三個產品上總共展示了 25 個攻擊（Bitwarden 12 個、LastPass 7 個、Dashlane 6 個），範圍從針對單一 vault 的完整性破壞到整個組織所有 vault 的全面淪陷。

研究團隊的威脅模型是「惡意伺服器」——假設攻擊者已控制伺服器端。在這個前提下，他們不需要什麼高級手段，只要使用者做日常操作（登入、同步、查看密碼），就能讀取甚至竄改儲存的密碼。所謂的「零知識加密」承諾，基本上撐不住這個威脅模型。

**重點：**
- 問題核心：為了使用者體驗（密碼恢復、家庭共享等功能），加密架構變得複雜，攻擊面隨之擴大
- 很多密碼管理器還在用 90 年代的密碼學技術，因為怕升級會讓使用者鎖死在外面
- 但是... 論文將在 USENIX Security 2026 發表，已給廠商 90 天修復期。不是所有廠商都積極回應——畢竟要動的是核心架構，不是補個 patch 就好

---

## ⚡ 一句話帶過

- **[Why is Claude an Electron app?](https://www.dbreunig.com/2026/02/21/why-is-claude-an-electron-app.html)** — HN 289 分的月經題：為什麼又是 Electron？答案永遠是 time-to-market 贏了一切
- **[Are compilers deterministic?](https://blog.onepatchdown.net/2026/02/22/are-compilers-deterministic-nerd-version/)** — 簡短回答：理論上是，實務上各種隨機因素讓你的 reproducible build 哭出來
- **[I Scanned Every Server in the Official MCP Registry](https://dev.to/kai_security_ai/i-scanned-every-server-in-the-official-mcp-registry-heres-what-i-found-4p4m)** — 518 個 MCP server 全掃了一遍，30.7% 沒有認證。AI agent 的新攻擊面，認真的
- **[OpenClaw Is Unsafe By Design](https://dev.to/dendrite_soup/openclaw-is-unsafe-by-design-58gb)** — Cline 被 prompt injection 打穿的完整攻擊鏈分析，AI agent 的供應鏈安全問題正在成形
- **[Agentic AI is reintroducing ClickOps](https://dev.to/dortort/agentic-ai-is-reintroducing-clickops-53d4)** — 花了十年消滅 ClickOps，AI agent 又把它帶回來了。IaC 要哭了
- **[Terminal UI: BubbleTea (Go) vs Ratatui (Rust)](https://dev.to/rosgluk/terminal-ui-bubbletea-go-vs-ratatui-rust-2plj)** — Elm 架構 vs immediate mode，選你的風格。兩個都很香
- **[Canvas_ity: A tiny, single-header canvas-like 2D rasterizer for C++](https://github.com/a-e-k/canvas_ity)** — 一個 header file 搞定 HTML Canvas 風格的 2D 渲染，C++ 極簡主義的浪漫
- **[EDuke32 – Duke Nukem 3D (Open-Source)](https://www.eduke32.com/)** — Duke Nukem 3D 的開源引擎還在活躍維護，134 個 HN 讚表示懷舊永不退流行
- **[The $100k AWS Routing Trap: S3 + NAT Gateways](https://dev.to/ntctech/the-100k-aws-routing-trap-s3-nat-gateways-and-how-to-fix-it-with-terraform-41fo)** — S3 流量走 NAT Gateway 燒錢的經典坑，用 VPC endpoint 就能省一大筆
- **[Kubernetes ImagePullBackOff: It's Not the Registry (It's IAM)](https://dev.to/ntctech/kubernetes-imagepullbackoff-its-not-the-registry-its-iam-2fek)** — 2026 年了，ImagePullBackOff 十次有九次是 IAM 權限問題，不是 registry 掛了

---

## 📚 慢慢啃

- **[浏览器时间管理大师：深度拆解 5 大核心调度 API](https://juejin.cn/post/7608118243883614217)** — 從 rAF 到 Scheduler API，把瀏覽器的任務調度機制講透了。寫過高頻渲染或大量資料處理的前端工程師會很有收穫
- **[你不知道的 JS——现代系统级 API 篇](https://juejin.cn/post/7608760879781724169)** — AsyncContext（Stage 3）、WebTransport、WebCodecs...JS 正在接管原本屬於 native app 的領地，看看你錯過了哪些
- **[Read-your-writes on replicas: PostgreSQL WAIT FOR LSN and MongoDB Causal Consistency](https://dev.to/franckpachot/read-your-writes-on-replicas-postgresql-wait-for-lsn-and-mongodb-causal-consistency-4he2)** — 讀寫分離架構下的一致性保證，PostgreSQL 17 的 WAIT FOR LSN 和 MongoDB 的 causal consistency 怎麼解決 read-after-write 問題
- **[The Software Development Lifecycle Is Dead](https://boristane.com/blog/the-software-development-lifecycle-is-dead/)** — 傳統 SDLC 在 AI 時代還適用嗎？有點標題黨但論點值得想想
