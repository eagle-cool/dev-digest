---
title: "BarraCUDA 硬幹 CUDA 編譯器、Rust async 上 GPU、Go 1.26 go fix 大翻新"
date: 2026-02-18
description: "有人用 15,000 行 C99 寫了個 CUDA 編譯器跑 AMD GPU，VectorWare 把 Rust async/await 搬上 GPU，Go 1.26 的 go fix 從古董變現代化武器。"
tags: [systems, programming, opensource]
---

今天三道硬菜都是底層技術。有人不爽 NVIDIA 的圍牆花園，用一萬五千行 C 硬寫了個 CUDA 編譯器讓 AMD GPU 也能跑；Rust 的 async/await 被搬上 GPU 了（對，就是那個 Future trait）；然後 Go 官方把 `go fix` 從十年前的遺跡重寫成現代化的程式碼自動升級工具。今天適合泡杯咖啡慢慢看。

---

## 🔧 今日硬菜

### [BarraCUDA: Open-source CUDA compiler targeting AMD GPUs](https://github.com/Zaneham/BarraCUDA)

有人看著 NVIDIA 的 CUDA 生態圍牆心想「能有多難？」然後真的用 15,000 行 C99 寫了一個完整的 CUDA 編譯器，直接把 `.cu` 檔編譯成 AMD RDNA 3 (GFX11) 的機器碼。不靠 LLVM，不走 HIP 轉譯，從 lexer、parser、SSA IR 到 1,700 行手寫的 instruction selection，一路硬幹到底。

這東西的完成度比你想的高：`__global__`、`__device__`、`__shared__` memory、`__syncthreads()`、atomic operations、warp intrinsics、cooperative groups 全都支援了。Build 流程也很猛——`make` 一行搞定，沒有 cmake，沒有 47 步建置流程。每一條指令編碼都跟 `llvm-objdump` 驗證過，零解碼錯誤。

**重點：**
- 零 LLVM 依賴，純 C99 實現完整的 CUDA→AMD GPU 編譯管線
- 已支援 templates、operator overloading、cooperative groups 等進階 CUDA 特性
- 但是... 還缺 compound assignment（`+=`）、`const`、多翻譯單元支援，而且目前沒有任何最佳化——能跑但別指望效能

### [Async/Await on the GPU](https://www.vectorware.com/blog/async-await-on-gpu/)

VectorWare 宣布他們成功在 GPU 上跑了 Rust 的 `Future` trait 和 `async`/`await`。這不是概念驗證而已——他們用了 Embassy（嵌入式系統的 async executor）在 GPU 上跑多個 concurrent task，而且幾乎不需要修改 Embassy 的程式碼。

核心論點很有說服力：GPU 程式設計正從單純的 data parallelism 往 warp specialization 的 task-based parallelism 演進，但目前的做法（手動管理 concurrency）跟當年 CPU 上寫 raw threads 一樣痛苦。Rust 的 async 模型剛好把 structured concurrency 編進語言本身——Future 是 compiler-generated state machine，不在乎底層是 thread、core 還是 warp。拿來跟 JAX 的 computation graph、Triton 的 block model、NVIDIA 的 CUDA Tile 比較，async/await 的優勢在於它是現有語言的一部分，不需要學新 DSL。

**重點：**
- Rust 的 Future trait 在 GPU 上成功運行，包括 chaining、conditionals、async blocks 和第三方 combinator
- 用 Embassy executor 在 GPU 上跑多任務排程，證明嵌入式生態可以直接複用
- 但是... GPU 沒有 interrupt，executor 只能用 spin loop polling，register pressure 也會影響 occupancy

### [Using go fix to modernize Go code](https://go.dev/blog/gofix)

Go 1.26 把 `go fix` 徹底重寫了。這個從 Go 1.0 前就存在的老工具，現在接上了 Go analysis framework，變成一個可以自動幫你把程式碼升級到最新 Go 慣用語法的現代化武器。

一個指令 `go fix ./...` 就能把 `interface{}` 換成 `any`、三段式 `for` 迴圈換成 `range n`、手動 `if/else` clamp 換成 `min(max(...))`。Go 1.26 還加了 `new(expr)` 語法（終於可以 `new("hello")` 了），配套的 fixer 會自動找到你程式碼裡的 `newInt()` 之類 helper function 然後全部替換掉。

有趣的是開發動機之一：LLM coding assistant 因為訓練資料的關係，傾向產出舊式 Go 程式碼，甚至被告知要用新特性時還會否認新特性的存在。Go 團隊的策略是先用 `go fix` 把全球開源 Go 程式碼都升級，讓未來的模型訓練資料跟上時代。

**重點：**
- 完全重寫的 `go fix` 接入 analysis framework，跟 `go vet` 共用基礎設施
- Go 1.26 新增 `new(expr)` 語法 + 配套自動升級工具，解決了十年老問題
- 但是... 「self-service」模式（讓第三方自定義 fixer）還在預覽階段，semantic conflict 需要手動解

---

## ⚡ 一句話帶過

- **[Your MCP Tools Are a Backdoor](https://dev.to/behrensd/your-mcp-tools-are-a-backdoor-5fbh)** — Claude Code 裝了個 MCP server，三秒後它讀了你的 SSH private key。沒警告、沒提示、沒 log。讀完再想想你裝了幾個 MCP tool。
- **[Gentoo on Codeberg](https://www.gentoo.org/news/2026/02/16/codeberg.html)** — Gentoo 搬家到 Codeberg 了。又一個大型開源專案離開 GitHub，Codeberg 的 infra 撐得住嗎是個好問題。
- **[AsteroidOS 2.0 – Nobody asked, we shipped anyway](https://asteroidos.org/news/2-0-release/index.html)** — 手腕大小的 Linux 八年後終於出 2.0。沒人問過他們要不要出，他們還是出了。這就是開源精神。
- **[Is it True That Go Maps Don't Shrink?](https://dev.to/kanywst/is-it-true-that-go-maps-dont-shrink-3m3)** — Go map 刪了元素記憶體不還的都市傳說，Issue #20135 開了八年了。寫 Go 的遲早踩到。
- **[pg_background: Make Postgres do the long work](https://vibhorkumar.wordpress.com/2026/02/16/pg_background-make-postgres-do-the-long-work-while-your-session-stays-light/)** — Postgres 背景執行長任務的擴充，session 不用傻等。DBA 看到會流淚。
- **[How We Reduced INP by 100ms+: GTM Isolation, React Compiler, and Better Telemetry](https://dev.to/subito/how-we-reduced-inp-by-100ms-gtm-isolation-react-compiler-and-better-telemetry-315g)** — 義大利最大分類廣告平台的 INP 優化實戰，React Compiler 上場了。前端效能調校的真實案例。
- **[Show HN: Pg-typesafe – Strongly typed queries for PostgreSQL and TypeScript](https://github.com/n-e/pg-typesafe)** — 寫原生 SQL 但要 TypeScript type safety？不靠 ORM 不靠 code gen，直接從 schema 推型別。
- **[Dolphin Emulator – Rise of the Triforce](https://br.dolphin-emu.org/blog/2026/02/16/rise-of-the-triforce/?cr=br)** — Dolphin 模擬器搞定了 Triforce arcade 板，GameCube 和 Wii 之外的新領域。逆向工程的勝利。
- **[Meta to retire Messenger desktop app and messenger.com](https://dzrh.com.ph/post/meta-to-retire-messenger-desktop-app-and-messengercom-in-april-2026-users-shift-to-web-and-mobile-platforms)** — Meta 四月關掉 Messenger 桌面版和網頁版。又砍一個 Electron app，但這次連網頁版一起砍。
- **[Surprise: You Can "Intercept" the C# lock Statement](https://dev.to/dimonsmart/surprise-you-can-intercept-the-c-lock-statement-14n)** — C# 的 `lock` 不是魔法，是語法糖，而且可以被「劫持」。知道就好，千萬別真的這樣做。
- **[无感监控：深度拆解监控 SDK 的性能平衡术与调度策略](https://juejin.cn/post/7606702049910849582)** — 如果你的監控 SDK 本身就是最大的效能瓶頸，那它監控了個寂寞。requestIdleCallback 和 Web Worker 的實戰調度。

---

## 📚 慢慢啃

- **[4 Months of Developing a Memory Allocator: Updating "Hakozuna" to v3.0](https://dev.to/charmpic/4-months-of-developing-a-memory-allocator-updating-hakozuna-to-v30-hz3hz4-9bb)** — 四個月開發一個 memory allocator 的完整紀錄。如果你想理解 malloc 背後在幹嘛，這篇從設計決策到效能測試全都有。
- **[Chess engines do weird stuff](https://girl.surgery/chess)** — 西洋棋引擎做出人類棋手看不懂的操作背後的原理。演算法和搜尋策略的有趣探討。
- **[Show HN: I wrote a technical history book on Lisp](https://berksoft.ca/gol/)** — 五年寫成的 Lisp 技術史，不是那種輕描淡寫的回顧，而是塞滿技術細節的歷史書。週末配咖啡剛好。
- **[NoamVC v0.3 — We deleted 3,500 lines and the app got better](https://dev.to/steven_hans_b26a962c69563/noamvc-v03-we-deleted-3500-lines-and-the-app-got-better-4nlb)** — P2P 加密語音聊天 app，Tauri 2 + React 19 + WebRTC。刪了 3,500 行程式碼反而更好——每個工程師都該學會的減法哲學。
