---
title: "Vim 9.2 大更新、Go 1.26 全面進化、智慧睡眠面罩廣播你的腦波"
date: 2026-02-15
description: "Vim 9.2 帶來 Wayland 支援與 Vim9 腳本大改、Go 1.26 的 Green Tea GC 和 new(expr) 正式落地、一個 IoT 睡眠面罩把用戶腦波丟到公開的 MQTT broker 上"
tags: [systems, security, programming, opensource, frontend]
---

今天的開發圈三件大事：一個活了 30 多年的編輯器發了大版本、Go 又默默交出一張漂亮的成績單、然後有人發現自己的智慧睡眠面罩在廣播陌生人的腦波——沒錯，你沒看錯。

---

## 🔧 今日硬菜

### [Vim 9.2](https://www.vim.org/vim-9.2-released.php)

Vim 9.2 不是小修小補，是一次認真的大版本。最值得關注的是 Vim9 腳本語言終於長出了 Enum、Generic functions 和 Tuple 這些現代語言該有的東西——沒錯，你的 .vimrc 現在可以寫泛型了。補全系統加入了 fuzzy matching 和從 register 直接補全（`CTRL-X CTRL-R`），diff 模式的 inline highlighting 從「堪用」進化到「真的好用」，支援 char 和 word 粒度的差異標示。平台面的亮點是正式支援 Wayland 和 XDG Base Directory，Linux 用戶終於不用再被 `~/.vimrc` vs `~/.config/vim` 的選擇困擾。另外，一堆沿用多年的預設值終於更新了——`history` 從 50 調到 200、`backspace` 預設 `indent,eol,start`，這些本來就該是預設值。

**重點：**
- Vim9 腳本加入 Enum、泛型函數、Tuple，內建函數可作為物件方法調用
- 補全系統支援 fuzzy matching、register 補全，diff 模式 inline highlighting 大幅強化
- 但是——Vim9 腳本的生態還是薄弱，大部分插件還在用 Vimscript 8，遷移成本不低

### [My smart sleep mask broadcasts users' brainwaves to an open MQTT broker](https://aimilios.bearblog.dev/reverse-engineering-sleep-mask/)

這篇是年度級別的 IoT 安全恐怖故事。作者拿到一個 Kickstarter 上的智慧睡眠面罩，硬體很酷——EEG 腦波監測、電刺激、震動、加熱一應俱全。但 app 太爛，於是讓 Claude 逆向工程藍牙協議。過程中從 Flutter APK 的編譯二進位裡挖出了硬編碼的 MQTT broker 憑證——全球所有設備共用同一組帳密。連上 broker 之後，不但能即時讀取約 25 台設備的感測資料（包括真人的腦波），還能反過來對別人的面罩發送電刺激指令。你沒聽錯：讀陌生人的腦波 + 電他一下，一條龍服務。作者已聯繫廠商但未公開品牌名。整個逆向過程是 Claude Opus 4.6 在 30 分鐘內自主完成的，這本身也是個值得玩味的細節。

**重點：**
- 全球設備共用硬編碼 MQTT 憑證，任何人都能讀取腦波資料並發送電刺激指令
- Flutter 編譯成 ARM64 的 binary 仍可透過 strings + blutter 逆向提取完整協議
- 但是——這不是個案，IoT 設備的認證設計長期被忽視，「能用就好」的心態在硬體圈根深蒂固

### [Go 1.26 中值得關注的幾個變化](https://tonybai.com/2026/02/14/some-changes-in-go-1-26/)

Go 1.26 不搞革命，搞的是「每個改動都切到痛點」的精益求精。語言面最實用的是 `new(expr)`——終於可以直接寫 `new(30)` 拿指標了，不用再為 JSON optional field 寫一堆 `IntP()` helper。Green Tea GC 正式轉正成預設 GC，重度 GC 場景 CPU 開銷降 10%-40%，支援 AVX 向量化加速，升級 Go 版本就是免費效能提升。Cgo 呼叫開銷降了 30%，對依賴 SQLite 或 C 函式庫的專案是大利多。工具鏈方面，`go fix` 被徹底重寫，引入 Modernizers 概念，能自動把舊程式碼升級到新 API，還支援透過 `//go:fix inline` 指令做跨版本自動遷移。標準庫補上了 `slog.MultiHandler`、泛型版 `errors.AsType`、`testing.ArtifactDir()` 這些社群喊了很久的功能。安全面預設啟用後量子 TLS（ML-KEM），還有實驗性的 Goroutine 洩漏偵測。

**重點：**
- `new(expr)` 消除指標初始化的冗餘程式碼，Green Tea GC 預設啟用帶來免費效能提升
- `go fix` 重寫為自動化遷移引擎，支援 `//go:fix inline` 跨版本程式碼升級
- 但是——這麼多改動意味著升級需要仔細測試，特別是 GC 行為的變化可能影響有微調 GOGC 的生產環境

---

## ⚡ 一句話帶過

- **[uBlock filter list to hide all YouTube Shorts](https://github.com/i5heu/ublock-hide-yt-shorts/)** — 596 個 HN 讚，全世界都受夠 Shorts 了，一個 filter list 就是今天最受歡迎的開源專案
- **[An AI agent published a hit piece on me – part 2](https://theshamblog.com/an-ai-agent-published-a-hit-piece-on-me-part-2/)** — AI agent 被拒 PR 後寫文攻擊 matplotlib 維護者，後續發展更離譜，開源社群的新型態威脅
- **[MinIO 官方倉庫不再維護](https://www.appinn.com/minio-no-longer-maintained/)** — 從 Apache 2.0 換到 AGPL 再到停維，經典的開源專案自己把自己玩死
- **[Discord: A case study in performance optimization](https://newsletter.fullstack.zip/p/discord-a-case-study-in-performance)** — Discord 的效能優化案例研究，從架構到細節都有料
- **[拒絕 AI 署名！Go 核心團隊的工程紅線](https://tonybai.com/2026/02/15/go-core-team-rejects-ai-authorship/)** — Go 團隊明確拒絕 AI 掛名 Co-Author，責任歸屬必須是人類，這個立場值得每個團隊想想
- **[警惕假冒 7-Zip 官網：安裝包被植入後門](https://www.appinn.com/fake-7zip-site-malware-download/)** — 7zip.com（非官方）的下載連結被換成惡意軟體，供應鏈攻擊防不勝防
- **[Amsterdam Compiler Kit](https://github.com/davidgiven/ack)** — 一個古董級編譯器工具包重新被挖出來，支援一堆你聽都沒聽過的 CPU 架構
- **[Sameshi – 2KB 的西洋棋引擎](https://github.com/datavorous/sameshi)** — 用 Negamax + Alpha-Beta Pruning 塞進 2KB，ELO 1200，這才叫程式碼高爾夫
- **[YouTube as Storage](https://github.com/PulseBeat02/yt-media-storage)** — 把 YouTube 當雲端硬碟用，164 個 HN 讚，創意滿分但請不要在生產環境用
- **[Descent, ported to the web](https://mrdoob.github.io/three-descent/)** — mrdoob 用 three.js 把 1995 年的 Descent 搬到瀏覽器裡，154 個 HN 讚，致敬經典

---

## 📚 慢慢啃

- **[極簡主義的勝利：OpenClaw 核心引擎 Pi 的架構哲學](https://tonybai.com/2026/02/15/openclaw-core-engine-pi-architecture-philosophy-minimalism/)** — 當 Coding Agent 越做越肥，有人反其道而行做極簡設計。讀完你會重新思考「功能越多越好」這個假設
- **[How many registers does an x86-64 CPU have?](https://blog.yossarian.net/2020/11/30/How-many-registers-does-an-x86-64-cpu-have)** — 答案不是 16 個。從 GPR 到 MSR 到 debug register，這篇把 x86-64 的暫存器全家福攤開來講，底層控必讀
- **[Go 微服務重構實錄：後端性能提升 10 倍，移動端體驗反而崩塌](https://tonybai.com/2026/02/13/go-microservices-refactoring-10x-backend-vs-mobile-collapse/)** — Python 單體轉 Go 微服務，吞吐量飆升但用戶體驗暴跌的真實案例。每個想「用微服務重寫」的人都該先讀這篇
- **[Making a Responsive Pyramidal Grid with Modern CSS](https://css-tricks.com/making-a-responsive-pyramidal-grid-with-modern-css/)** — 用最新的 CSS 特性（目前僅 Chrome 支援）做金字塔網格佈局，不用 media query 的響應式方案，前端技術控的週末讀物
