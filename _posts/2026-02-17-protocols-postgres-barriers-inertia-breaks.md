---
title: "協定打敗服務、Postgres 競態測試有解、Inertia.js 的生產地雷"
date: 2026-02-17
description: "用協定取代中心化服務的論述再起、同步屏障讓 Postgres race condition 測試不再靠運氣、Inertia.js 在生產環境踩出六個痛點。今日前端與系統工程的硬菜。"
tags: [systems, frontend, programming, opensource]
---

今天三篇硬菜各自精彩。有人重新喊出「用協定不用服務」這句老話（但時機剛好）、有人終於把 Postgres race condition 測試從「跑一千次祈禱」變成確定性的事、還有人在 Laravel + React + Inertia.js 的生產環境裡被 router 的語義陷阱搞到懷疑人生。

---

## 🔧 今日硬菜

### [Use protocols, not services](https://notnotp.com/notes/use-protocols-not-services/)

這篇講的道理其實不新——用開放協定（SMTP、XMPP、Matrix、ActivityPub）取代封閉平台服務——但時間點抓得很準。Discord 開始強制年齡驗證要刷臉或交身分證，各國政府對平台的監管力道只會越來越大。作者的核心論點很簡單：政府要管一個服務，只需要一封律師信；要管一個協定，得逐一施壓成千上萬個獨立節點，這在執法上幾乎不可能。

文章用 email 當正面例子——Google 封你帳號，你換個 provider 照樣寄信給所有 Gmail 用戶。SMTP 的實作還在、協定還活著，就算 Google 和 Microsoft 同時退出某個地區也一樣。這跟 Discord 封你就是永久消失完全不同。

**重點：**
- 中心化服務是政府監管的最佳施力點，一封傳票就能讓平台交出所有使用者資料
- 協定的去中心化特性讓合規要求在執行面幾乎不可能——每個節點各自決策
- 但是... 協定生態容易走向寡頭（看看 email 被 Google/Microsoft 壟斷的現況），去中心化不是萬靈丹

### [Testing Postgres race conditions with synchronization barriers](https://www.lirbank.com/harnessing-postgres-race-conditions)

踩過並行 bug 的人都知道，race condition 最討厭的地方不是修——是重現。你的測試跑一次一次過，但生產環境偶爾就是會少一筆 $50。這篇提出用 synchronization barrier 來「製造」確定性的 race condition，讓測試不再靠機率。

核心手法很優雅：在兩個並行操作的 read 和 write 之間放一個 barrier，強制兩邊都讀完舊值之後才放行寫入。作者從最裸的 SELECT + UPDATE（失敗）、包 transaction（還是失敗，因為 READ COMMITTED 不是寫鎖）、到 `SELECT ... FOR UPDATE`（barrier 造成 deadlock——但這個 deadlock 恰好「證明」了鎖是有效的），一路展示不同隔離等級下的行為差異。最後把 barrier 往前移到 BEGIN 之後、SELECT 之前，完美通過。

最狠的是文末那句：每次改 barrier 或業務邏輯，都要拿掉鎖跑一次確認測試會失敗。「如果加鎖和不加鎖都通過，那你寫的是虛榮測試。」

**重點：**
- Synchronization barrier 讓 race condition 測試從機率變成確定性——barrier 釋放前所有 task 都在等
- 用 hook 注入 barrier，生產環境零開銷，測試時才啟用
- 但是... 需要真實 Postgres 實例，mock 沒有鎖、沒有 contention、測不出東西

### [Inertia.js Silently Breaks Your App](https://dev.to/danieltofan/inertiajs-silently-breaks-your-app-oi8)

Inertia.js 的賣點是「不用維護獨立 API 就能做 SPA 體驗」，聽起來很美。但這位老兄在 Laravel 12 + React 19 + Inertia v2 的生產環境踩了六個坑，每一個都不是 happy path demo 能發現的。

最致命的是第一個：`router.put()` 長得像 Promise，但它不是。你寫 `await router.put(A); await router.put(B)`，以為 A 完成才執行 B——結果 Inertia 把 A 取消了，只跑 B。這是 by design。社群要求加回 Promise 支援多年未果。第二個是部署時 stale chunk 問題——server 已經指向新的 component manifest，但客戶端還拿著舊的 runtime，chunk import 失敗，使用者看到「死掉的」導航。第三個是錯誤處理預設靜音：JS 錯誤導致導航失敗時，頁面什麼都不變，沒有任何提示。

**重點：**
- Inertia router 的 `await` 是假的——overlapping visit 會取消前一個，這是設計決策不是 bug
- 部署期間 server/client manifest 不同步會導致 chunk 載入失敗，使用者只能硬刷新
- 但是... 作者也承認 Inertia 在 CRUD-heavy 的後台管理介面運作良好，問題出在複雜工作流程

---

## ⚡ 一句話帶過

- **[How We Replaced MinIO with Garage for Self-Hosted S3 Storage](https://dev.to/alexneamtu/how-we-replaced-minio-with-garage-for-self-hosted-s3-storage-23f7)** — MinIO 改授權條款，Rust 寫的 Garage 一個下午就搬完，self-hoster 又多一個選擇
- **[Fff.nvim – Typo-resistant code search](https://github.com/dmtrKovalenko/fff.nvim)** — 打錯字也能搜到 schema 的 Neovim 插件，對 agent 和手殘人類都友善
- **[Rise of the Triforce](https://dolphin-emu.org/blog/2026/02/16/rise-of-the-triforce/)** — Dolphin 模擬器成功跑起 Triforce 街機板，逆向工程的浪漫
- **[Go Heap Fragmentation Deep Dive](https://dev.to/kanywst/go-heap-fragmentation-deep-dive-the-battle-against-invisible-memory-continues-4o7h)** — GOMEMLIMIT 救不了的「隱形記憶體」，Go runtime 記憶體碎片化的最後一哩路
- **[WebMCP Proposal](https://webmachinelearning.github.io/webmcp/)** — W3C 提案要把 MCP 搬進瀏覽器，126 個 HN upvotes 說明大家有多想要這個
- **[Running NanoClaw in a Docker Shell Sandbox](https://www.docker.com/blog/run-nanoclaw-in-docker-shell-sandboxes/)** — Docker 官方示範怎麼用 shell sandbox 跑 AI agent，安全感 +1
- **[STM32G431 Analogue TV Transmitter](https://slyka.net/blog/2026/tinyvision/)** — 用一顆 STM32 當類比電視發射器，硬體 hacker 的快樂就這麼簡單
- **[Show HN: FreeFlow – Free Alternative to Wispr Flow](https://github.com/zachlatta/freeflow)** — 開源的語音轉文字工具，Superwhisper 的免費替代方案
- **[Suicide Linux (2009)](https://qntm.org/suicide)** — 打錯指令就 `rm -rf /` 的 Linux distro，2009 年的經典惡搞今天又上 HN
- **[State of Show HN: 2025](https://blog.sturdystatistics.com/posts/show_hn/)** — 有人統計了去年所有 Show HN 的趨勢數據，meta 到不行但真的有趣

---

## 📚 慢慢啃

- **[Idempotency: The Concept Everyone Mentions but Few Implement Correctly](https://dev.to/speaklouder/idempotency-the-concept-everyone-mentions-but-few-implement-correctly-2pbc)** — 別再用 flag 和 retry 假裝冪等了，這篇從實作層面拆解怎麼做才對
- **[Python Internals: Generators & Coroutines](https://dev.to/aykhlf_yassir/python-internals-generators-coroutines-11j2)** — 從 `yield` 到 coroutine 的底層機制，讀完你會重新看待每一個 Python generator
- **[Observability Is Authored, Not Installed](https://dev.to/stevenstuartm/observability-is-authored-not-installed-4lcc)** — 裝了 Datadog 不等於有 observability，有意義的 context 要寫進程式碼裡
- **[The long tail of LLM-assisted decompilation](https://blog.chrislewis.au/the-long-tail-of-llm-assisted-decompilation/)** — LLM 幫你反編譯的極限在哪裡？答案是長尾問題比你想的嚴重
