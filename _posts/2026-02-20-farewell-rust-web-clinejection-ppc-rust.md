---
title: "Rust 揮別 Web、Cline 供應鏈炸裂 CVSS 9.9、老 Mac 上的 Rust 編譯器"
date: 2026-02-20
description: "一位獨立開發者花了兩年用 Rust 寫 Web 最後投降回 Node.js，Cline VS Code 套件被 AI prompt injection 搞出 CVSS 9.9 的供應鏈攻擊，還有人在 20 年前的 PowerPC Mac 上從零寫了一個帶 borrow checker 的 Rust 編譯器。"
tags: [systems, security, frontend, programming, retro, devops]
---

今天三件事值得認真聊。一個 Rust 信徒花了兩年在 Web 上硬撐，最後坦然說「Node.js 真的夠用了」——這種話從踩過坑的人嘴裡說出來，份量不一樣。然後 Cline 那個 VS Code AI 套件被自己的 AI agent 反咬一口，CVSS 9.9，供應鏈直接被打穿。最後有個瘋子在 2005 年的 PowerPC Mac 上用 C 寫了一個 Rust 編譯器，borrow checker 都有。

---

## 🔧 今日硬菜

### [Farewell, Rust for web](https://yieldcode.blog/post/farewell-rust/)

Dmitry Kudryavtsev 是個有 15 年經驗的全端工程師，從 Pascal 寫到 C 再到 PHP，最後因為懷念 C 的記憶體控制而一頭栽進 Rust。他用 Rust 從零寫了一個完整的 Web 應用，上線賺錢，還在兩個 conference 上講過。然後他放棄了，全部遷回 Node.js。

問題出在哪？不是 borrow checker——他早過了那關。是 Web 開發的本質跟 Rust 的設計哲學根本性衝突。`tera`、`handlebars` 這些模板引擎沒有型別安全，改個變數名就可能在 production 上看到 `{{dearCustomer}}`。i18n 支援貧弱到他得自己實作 API。`sqlx` 的 compile-time SQL 檢查很讚，但動態查詢寫起來像在受刑。更要命的是 CI/CD：Rust 的 Docker build 要 14 分鐘（還不含測試），Node.js 只要 5 分鐘還跑完了 lint 和測試。

最讓我共鳴的一句話：「I found myself ignoring bugs in Sentry because it meant going back to long compile times。」踩過這個坑的獨立開發者都懂——當修 bug 的心理成本太高，你就開始假裝沒看到。

**重點：**
- 模板引擎缺乏型別安全、i18n 生態不成熟、動態 SQL 寫起來痛苦——Web 的動態本質跟 Rust 的靜態哲學不合
- CI/CD 時間差 3 倍（14 分鐘 vs 5 分鐘），獨立開發者的時間成本被嚴重放大
- 但是⋯⋯ 他說得很清楚：如果是純 API service 或 CPU-intensive 的工作，他還是會選 Rust。工具沒有對錯，只有適不適合

### [Clinejection: When AI Agents Go Rogue and Poison Your Supply Chain](https://dev.to/cverports/ghsa-9ppg-jx86-fqw7-clinejection-when-ai-agents-go-rogue-and-poison-your-supply-chain-39hm)

CVSS 9.9。沒打錯，九點九。Cline 是 VS Code 上熱門的 AI coding 套件，它的 GitHub repo 裡有一個用 Claude 做 issue triage 的 GitHub Actions workflow。攻擊者在 issue title 裡塞了 prompt injection，讓 AI agent 乖乖執行任意 Bash 指令。拿到的初始權限不高？沒關係——他汙染了 GitHub Actions cache，等下一次有 release workflow 被觸發時，惡意程式碼就搭了順風車，偷走了 NPM publish token、VSCE PAT、OVSX PAT，然後推了一個被汙染的 `cline@2.3.0` 到 npm registry。

這整個攻擊鏈是教科書等級的 CI/CD exploitation：prompt injection → cache poisoning → credential theft → supply chain compromise。最諷刺的是，攻擊者不需要找任何傳統漏洞，AI agent 本身就是那個「naive, over-privileged accomplice」。

**重點：**
- AI agent 在 CI/CD 裡處理不受信任的輸入（issue title）並擁有 shell access，這是明確的 attack surface
- GitHub Actions cache 的 scope 設計讓低權限 workflow 能汙染高權限 workflow 的快取——架構性問題
- 但是⋯⋯ 修復方法其實很直白：刪掉那個 workflow、撤銷所有 token、unpublish 惡意版本。真正該學的教訓是：別讓 AI agent 在有 secrets 的環境裡處理使用者輸入

### [I Built a Rust Compiler for a 20-Year-Old Mac (Borrow Checker and All)](https://dev.to/scottcjn/i-built-a-rust-compiler-for-a-20-year-old-mac-borrow-checker-and-all-37n7)

有人用 C 寫了一個 Rust-to-PowerPC 編譯器，跑在 Mac OS X Tiger 10.4 上，用 2005 年的 GCC 4.0.1 編譯。聽起來像在開玩笑，但他是認真的——1,205 行 C 搞定 parsing、型別檢查、code generation，另外 500 行實作 borrow checker（包含 NLL），還有 900 行的 async/await runtime。

Borrow checker 不是做做樣子。他實作了完整的 ownership tracking、mutable/immutable borrow 衝突偵測，連 Non-Lexical Lifetimes 都有。async runtime 用的是 `select()` ——1983 年的 4.2BSD 那個 `select()`，因為 Tiger 沒有 epoll 也沒有 io_uring。Code generation 會輸出 AltiVec SIMD 指令，讓 G4 的 128-bit 向量單元能用上。`Arc` 的 atomic reference counting 用 PowerPC 的 `lwarx`/`stwcx.` 實作——跟現代 ARM 的 `ldxr`/`stxr` 是同一個設計思路。

目標？編譯 Firefox。TenFourFox（最後一個 PowerPC Firefox fork）2021 年就停了，新版 Firefox 有大量 Rust 程式碼。他的計劃是用這個編譯器把 Rust 元件編譯成 PowerPC native code，搭配自己建的 mbedTLS 網路棧，在 20 年前的硬體上跑現代瀏覽器。

**重點：**
- 1,205 行 C 實作 Rust 型別系統 + 500 行 borrow checker（含 NLL）+ 900 行 async runtime，不是玩具
- 用 AltiVec SIMD 和 PowerPC 原子指令做硬體加速，充分利用老硬體的設計——這些 RISC 概念被現代 ARM 直接繼承
- 但是⋯⋯ 即便技術上驚人，在 dual G4 上 build Firefox 要 8-12 小時，這更像是一個「能不能做到」的工程挑戰而非實用方案

---

## ⚡ 一句話帶過

- **[GHSA-VRHM-GVG7-FPCF: SvelteKit Remote Functions DoS](https://dev.to/cverports/ghsa-vrhm-gvg7-fpcf-sveltekit-remote-functions-death-by-type-coercion-2h45)** — SvelteKit 實驗性 remote functions 有 CVSS 7.5 的 DoS 漏洞，type coercion 導致記憶體耗盡，用了這功能的趕緊更新
- **[告别视口依赖：Container Queries](https://juejin.cn/post/7607598321469341742)** — CSS Container Queries 終於讓元件根據「自己的容器」而非「整個視窗」來決定樣式，微前端和元件庫開發者等這個等了十年
- **[GitLab CI: Achieving 3-Second Jobs on Million-Line Codebases](https://dev.to/zenika/gitlab-ci-achieving-3-second-jobs-on-million-line-codebases-3nlm)** — 百萬行程式碼的 GitLab CI job 跑 3 秒，關鍵是 shell runner + shallow clone + 把快取玩到極致
- **[Writing a BPF packet filter on macOS in Go](https://dev.to/krjakbrjak/writing-a-bpf-packet-filter-on-macos-in-go-45al)** — 在 macOS 上用 Go 寫 BPF packet filter，附帶清楚的有/無 filter 對比圖，系統程式設計入門好題材
- **[Show HN: cmux — Ghostty-based terminal with vertical tabs](https://github.com/manaflow-ai/cmux)** — 多個 Claude Code / Codex session 同時跑的人終於有救了，vertical tabs + 智慧通知讓你知道哪個 agent 在等你
- **[Building a JIT Compiler from Scratch](https://dev.to/darmie/building-a-jit-compiler-from-scratch-part-1-why-build-a-jit-compiler-590o)** — 寫完 bytecode VM 覺得不夠快？這位仁兄決定自己寫 JIT 而不是接 LLVM，用 Rust 寫 Haxe 編譯器的過程記錄
- **[File Conversion Fully in the Browser (WASM, LibreOffice, FFmpeg)](https://dev.to/digitalofen/i-tried-running-file-conversion-fully-in-the-browser-wasm-libreoffice-ffmpeg-57mh)** — 用 WASM 把 FFmpeg 和 LibreOffice 搬進瀏覽器做檔案轉換，不傳伺服器、零隱私問題，但記憶體吃法會讓你嚇一跳
- **[Symfony + FrankenPHP: A Modern Stack for Developer Tools](https://dev.to/santacruz/symfony-frankenphp-a-modern-stack-for-developer-tools-1d03)** — 「PHP is boring. That's exactly why we chose it.」用 Symfony + FrankenPHP 三週做了 19 個開發者工具，穩定就是最大的特色
- **[weathr: Terminal weather app with ASCII animations](https://github.com/Veirt/weathr)** — 即時天氣資料驅動的 ASCII 動畫，HN 上 140 分，程式碼不多但 terminal 美學拉滿
- **[Choosing a Language Based on Its Syntax?](https://www.gingerbill.org/article/2026/02/19/choosing-a-language-based-on-syntax/)** — Odin 語言作者談「語法」到底重不重要，結論是：比你想的重要，但原因不是你想的那個

---

## 📚 慢慢啃

- **[Rarely Unique Shared Mutexes](https://dev.to/walt_karas_llsw/rarely-unique-shared-mutexes-54m6)** — 當 shared lock 遠多於 unique lock 時，`std::shared_mutex` 的效能可以被大幅超越。需要理解 CPU cache line 和 atomic 操作的底層機制，適合週末配咖啡細看
- **[从样式表到渲染引擎：2026 年前端必须掌握的 CSS 架构新特性](https://juejin.cn/post/7607636614357598234)** — CSS Houdini 的 Paint API 實戰、Layout Worklet、Typed OM，從「寫樣式」升級到「寫渲染邏輯」，前端工程師進階必讀
- **[Beyond the Dockerfile: A 7-Layer Blueprint for Production-Grade Container Hardening](https://dev.to/shireen/beyond-the-dockerfile-a-7-layer-blueprint-for-production-grade-container-hardening-24hk)** — 不要再用 root 跑 container 了。七層加固策略從 base image 選擇到 runtime 安全政策，DevOps 工程師收藏用
- **[Searchable JSON Compression: Page-Level Random Access + ms Lookups](https://dev.to/kodomonocch1/searchable-json-compression-page-level-random-access-ms-lookups-and-smaller-than-zstd-on-our-3k1h)** — 不只壓得比 Zstd 小，還能在壓縮狀態下直接搜尋，用 Semantic Entropy Encoding 做到毫秒級隨機存取。資料密集型應用的人值得花時間理解原理
