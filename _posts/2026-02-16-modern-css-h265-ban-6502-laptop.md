---
title: "現代 CSS 不用再寫 2015、H.265 專利禁售令炸德國、6502 自製筆電"
date: 2026-02-16
description: "64 組新舊 CSS 對照讓你戒掉過時寫法、Nokia H.265 專利戰讓 Acer 和 ASUS 在德國禁售 PC、有人用 6502 CPU 手搓了一台筆記型電腦。"
tags: [frontend, systems, security, opensource]
---

今天三件事值得停下來看。一個網站把 64 組「你還在這樣寫 CSS？」的對照整理出來，看了會想回去把自己的 codebase 翻一遍。Nokia 拿 H.265 專利在德國把 Acer 和 ASUS 的 PC 給禁售了——對，你沒看錯，是「禁售」。然後有人用 1975 年的 6502 CPU 做了一台筆電，還跑 BASIC，我只能說致敬。

---

## 🔧 今日硬菜

### [Modern CSS Code Snippets: Stop writing CSS like it's 2015](https://modern-css.com)

64 組新舊 CSS 寫法對照，涵蓋 Layout、Animation、Color、Selector、Typography、Workflow 六大類。不是那種「看看這個新屬性好酷喔」的展示頁，而是直接告訴你：你那個 `@media (max-width: 768px)` 可以換成 `@container` 了、你的 JavaScript modal 可以用 `<dialog>` 取代了、你的顏色系統該從 HSL 換到 `oklch()` 了。

每組對照都標了瀏覽器支援度（35% 到 96%），所以不是在跟你講理論——是告訴你今天就能用的東西。`@starting-style` 讓 CSS transition 終於能處理進場動畫、`text-wrap: balance` 解決了標題斷行的千年問題、`@scope` 和 `@layer` 讓你不用再跟 specificity 搏鬥。踩過 CSS 坑的都知道，這些東西等了多久。

**重點：**
- `@container` queries 讓元件自己決定 layout，不再依賴 viewport——真正的元件化 CSS
- `appearance: base-select` 和 Popover API 大幅減少為了 UI 控制項寫的 JavaScript
- 但是... 瀏覽器支援度參差不齊，`@scope` 只有 66%，production 上之前先查 caniuse

### [Court orders Acer and Asus to stop selling PCs in Germany over H.265 patents](https://videocardz.com/newz/acer-and-asus-are-now-banned-from-selling-pcs-and-laptops-in-germany-following-nokia-hevc-patent-ruling)

慕尼黑法院 1 月 22 日裁定 Nokia 的 H.265（HEVC）標準必要專利侵權成立，直接對 Acer 和 ASUS 下了禁售令。不是罰款、不是警告，是「你的電腦不准在德國賣了」。兩家的德國線上商店已經受影響，直接銷售通路暫停。

法院認定 Acer 和 ASUS 不符合 FRAND（公平、合理、非歧視性授權）框架下的「願意被授權者」條件，所以 Nokia 拿到了禁制令。對比之下，Hisense 今年 1 月就乖乖簽了授權。這件事的意義不只是兩家 OEM 的銷售問題——它再次證明影片編碼的專利地雷有多恐怖，也讓開放標準（AV1、VVC 的開放實作）的重要性更加凸顯。

**重點：**
- Nokia 手上的影片專利不只 H.265，還包括 H.264、H.266、串流最佳化等一整個組合拳
- 零售商目前還能賣庫存，但補貨會受影響——不是全面消失，但供應鏈會受阻
- 但是... 這是德國法院的判決，Acer 和 ASUS 都表示會上訴，最終可能以授權協議收場

### [LT6502: A 6502-based homebrew laptop](https://github.com/TechPaula/LT6502)

有人用 65C02 CPU（對，就是 Apple II 那顆的後繼者）做了一台真正能用的筆記型電腦。8MHz 時脈、46K RAM、ROM 裡跑 EhBASIC、9 吋螢幕、Compact Flash 儲存、10000mAh 電池、USB-C 充電。從 PCB 設計、CPLD 邏輯到 3D 列印外殼全部自己來，還加了繪圖指令——`CIRCLE`、`LINE`、`PLOT`，用 BASIC 就能畫圖。

看那個 memory map 就知道作者是認真的：0x0000-0xBEAF 是 RAM、0xBF00 區段精心分配給各個周邊（VIA、CF 卡、螢幕、鍵盤、FTDI 序列埠）。從去年 11 月的初始 commit 到今年 2 月組裝完成，整個開發時程清清楚楚記在 README 裡。這不是概念驗證，是一台有鍵盤、有螢幕、能存檔讀檔的完整電腦。

**重點：**
- 完整的硬體設計開源，包含電路圖、PCB、韌體和外殼 CAD 檔案
- EhBASIC 被擴充了自訂指令：`CIRCLE`、`ELIPSE`、`SAVE`、`LOAD`、`DIR`——復古但實用
- 但是... 這是個人 passion project，不要指望量產或社群支援，純粹是致敬和學習用

---

## ⚡ 一句話帶過

- **[Gwtar: A static efficient single-file HTML format](https://gwern.net/gwtar)** — gwern 把圖片 CSS JS 全塞進一個 HTML 檔，離線閱讀的終極型態，就很 gwern
- **[GNU Pies – Program Invocation and Execution Supervisor](https://www.gnu.org.ua/software/pies/)** — 不想用 systemd 但需要 process supervisor？GNU 自家的輕量方案，支援 inetd 模式
- **[Error payloads in Zig](https://srcreigh.ca/posts/error-payloads-in-zig/)** — Zig 的 error payload 設計跟 Rust 的 Result 走了不同路線，值得比較
- **[Show HN: Lightwave – 3.5 years of hand-rolled JavaScript](https://news.ycombinator.com/item?id=47027463)** — 一個人花三年半不用框架手刻即時筆記 app，勇氣可嘉，成品也真的能用
- **[Show HN: Knock-Knock.net – Visualizing bots knocking on my server](https://knock-knock.net)** — 即時視覺化你的 server 被多少 bot 在掃，看了之後會想去檢查自己的 firewall
- **[Pocketblue – Fedora Atomic for mobile devices](https://github.com/pocketblue/pocketblue)** — Fedora 的不可變桌面架構搬到手機上，Linux phone 生態又多一個選項
- **[AWS App Mesh Deprecated: Migration Guide](https://dev.to/kseniyaseliverstava/aws-app-mesh-deprecated-escaping-app-mesh-before-september-2026-1l7m)** — App Mesh 九月收攤，還在用的趕快遷到 ECS Service Connect 或 VPC Lattice
- **[Partial Indexes in PostgreSQL](https://dev.to/mrpercival/partial-indexes-in-postgresql-24pb)** — 只索引你真正查詢的資料子集，index 瘦身的好招，PostgreSQL 用戶必學
- **[Show HN: VOOG – Moog-style synthesizer in Python](https://github.com/gpasquero/voog)** — 純 Python + tkinter 實作 Moog 複音合成器，三個 oscillator 加 24dB 梯形濾波器，音訊程式設計範例
- **[Show HN: DSCI – Dead Simple CI](https://github.com/melezhik/DSCI)** — 極簡 CI 框架，透過 webhook 接 Gitea/Forgejo/GitLab，適合不想碰 Jenkins 的小團隊

---

## 📚 慢慢啃

- **[Circumventing Internet Censorship: Protocols, Techniques, and the Arms Race](https://dev.to/shinomontaz/circumventing-internet-censorship-protocols-techniques-and-the-arms-race-20ff)** — 從 DPI 到 SNI 封鎖到混淆協定，完整解析俄羅斯網路審查的技術攻防史，讀完你會對「牆」有更深的理解
- **[Hideki Sato, designer of all Sega's consoles, has died](https://www.videogameschronicle.com/news/hideki-sato-designer-of-segas-consoles-dies-age-75/)** — 從 Mega Drive 到 Dreamcast，所有 Sega 主機都出自他手。一位硬體工程傳奇的一生，值得花十分鐘讀完
- **[The silver bullet – why building software is still hard](https://dev.to/nuri/the-silver-bullet-why-building-software-is-still-hard-4o6p)** — 重讀 Fred Brooks 的經典，對照 2026 年的 AI coding 工具，問一個尖銳的問題：本質複雜度真的被解決了嗎？
- **[I built a GPU-accelerated PDF renderer from scratch in C++](https://dev.to/informal061/i-built-a-gpu-accelerated-pdf-renderer-from-scratch-in-c-9mn)** — 用 C++ 和 Direct2D 從零打造 PDF 渲染引擎，如果你想知道 PDF 規格到底有多複雜，這篇會讓你大開眼界
- **[Stop Installing NPM Packages for Everything](https://dev.to/maxxmini/stop-installing-npm-packages-for-everything-use-these-browser-native-apis-instead-2pbn)** — 盤點 Web Crypto、Clipboard API、Intl 等原生 API 能取代的 npm 套件，每刪一個依賴就少一個 supply chain 攻擊面
