---
title: "Chrome CSS 零日漏洞被野外利用、Let's Encrypt 搞了新驗證方式、Ladybird 放棄 Swift"
date: 2026-02-19
description: "Chrome CSS 引擎的 use-after-free 零日漏洞正被積極利用，Let's Encrypt 推出 DNS-PERSIST-01 讓你不用每次換 DNS 記錄，Ladybird 瀏覽器花了兩年終於放棄 Swift。"
tags: [security, frontend, systems, opensource]
---

今天最該注意的是 Chrome 又爆了個零日漏洞，而且是在 CSS 引擎裡——對，就是你每天寫的那個 CSS。另外 Let's Encrypt 終於解決了 DNS-01 那個「每次續憑證都要改 DNS」的煩人問題，然後 Ladybird 瀏覽器花了兩年試著用 Swift 重寫，最後承認走不通。

---

## 🔧 今日硬菜

### [Chrome Stable Channel Update — CVE-2026-2441](https://chromereleases.googleblog.com/2026/02/stable-channel-update-for-desktop_13.html)

Chrome 145.0.7632.75 修了一個 CSS rendering engine 的 use-after-free 漏洞，嚴重等級 High，而且 Google 直接說了：「an exploit for CVE-2026-2441 exists in the wild」。翻成白話就是——已經有人在用了，你還不更新嗎？

Use-after-free 在瀏覽器裡一直是最危險的那類漏洞。記憶體被釋放後還被存取，攻擊者可以控制那塊記憶體的內容，最終達成任意程式碼執行。而這次漏洞在 CSS engine，意味著單純瀏覽一個有惡意 CSS 的網頁就可能中招——不需要你點任何東西。由 Shaheen Fazim 在 2 月 11 日回報，一週內就出了修補。

**重點：**
- CSS rendering engine 的 use-after-free，嚴重等級 High，已有野外攻擊
- 影響所有 Chromium 系瀏覽器（Chrome、Edge、Brave、Arc⋯⋯你用什麼都一樣）
- 但是⋯⋯ Google 沒公佈漏洞細節（慣例），所以我們只知道它很嚴重，不知道它怎麼觸發——趕快更新就對了

### [DNS-PERSIST-01: A New Model for DNS-Based Challenge Validation](https://letsencrypt.org/2026/02/18/dns-persist-01.html)

踩過 Let's Encrypt DNS-01 驗證的人都知道那個痛：每次簽發憑證都要改一次 DNS TXT 記錄、等 propagation、祈禱 TTL 不要太長。大型部署環境裡，DNS API 的 credentials 散落在整個 CI/CD pipeline 各處，attack surface 大得嚇人。

DNS-PERSIST-01 的概念很簡單但很聰明：你設一筆 `_validation-persist.example.com` 的 TXT 記錄，裡面放 CA 的 domain name 和你的 ACME account URI，設好之後就不用再動了。所有後續的簽發和續期，Let's Encrypt 看到這筆記錄就知道你授權了。不用每次改 DNS，DNS write credentials 可以收回保管箱，renewal pipeline 瞬間簡化。

還支援 `policy=wildcard` 給 wildcard 憑證用，也可以加 `persistUntil` 設過期時間。CA/Browser Forum 已經在 2025 年 10 月全票通過了，Staging 預計 Q1 2026 上線，Production 在 Q2。

**重點：**
- 一次性 DNS 記錄取代反覆改 DNS，把 DNS write credentials 移出 issuance pipeline
- 支援 wildcard、多 CA 同時授權、可選過期時間，設計上相當完整
- 但是⋯⋯ 永久授權記錄意味著 ACME account key 變成最關鍵的安全資產——掉了就等於把簽發權限送出去

### [Ladybird: Closing this as we are no longer pursuing Swift adoption](https://github.com/LadybirdBrowser/ladybird/issues/933)

Ladybird 瀏覽器在大約兩年前開始評估用 Swift 取代部分 C++ 程式碼，理由是 Swift 有 C++ interop、記憶體安全、而且語法比 Rust 對 C++ 工程師更友善。結果呢？兩年過去，Swift 6.0 Blockers 這個 issue 被關掉了，理由是「After making no progress on this for a very long time, let's acknowledge it's not going anywhere」。

問題出在 Swift 的 C++ interop 理論上有、實務上不行。遇到 conflicting C++ versioned libraries 就跪了，想用 C++ interop 呼叫特定 library 結果不支援，最後只能寫 C wrapper 繞過去——那還不如直接寫 C++。加上 Swift 仍然是 Apple 主導的語言，獨立於 Apple 生態的 expertise 少之又少，團隊內沒人真的擅長。

這是一個很經典的技術選型教訓：語言「官方支援 X」跟「X 在生產環境真的能用」之間的距離，有時候比你想的遠。

**重點：**
- 兩年實驗後放棄，Swift 的 C++ interop 在複雜 C++ codebase 中無法實際運作
- Build system 整合也是大問題，CMake + Swift 的搭配充滿地雷
- 但是⋯⋯ 這不代表 Swift 不好，而是它離開 Apple 生態後的成熟度還不夠——瀏覽器這種規模的 C++ 專案需要的不是「能用」而是「穩定」

---

## ⚡ 一句話帶過

- **[Tailscale Peer Relays is now generally available](https://tailscale.com/blog/peer-relays-ga)** — NAT 穿透失敗時讓附近的 peer 幫忙 relay，不用走 Tailscale 的 DERP server，延遲直接砍一刀
- **[CVE-2026-1669: Keras Model Poisoning — Arbitrary File Read](https://dev.to/cverports/cve-2026-1669-model-poisoning-turning-keras-weights-into-weaponized-file-readers-14kn)** — Keras 的模型載入有 CVSS 7.1 的任意檔案讀取漏洞，載入不信任的 `.h5` 就會被偷 `/etc/passwd`
- **[Cosmologically Unique IDs](https://jasonfantl.com/posts/Universal-Unique-IDs/)** — UUID 不夠用？來看看怎麼設計一個在可觀測宇宙裡都不會碰撞的 ID，HN 上 253 分不是沒道理的
- **[Echo: iOS SSH + mosh client built on Ghostty](https://replay.software/updates/introducing-echo)** — 用 Ghostty 的 terminal 引擎做了個 iOS SSH client，支援 mosh，終於有個像樣的手機 terminal 了
- **[Pocketbase lost its funding from FLOSS fund](https://github.com/pocketbase/pocketbase/discussions/7287)** — 開源界又一個「用愛發電到彈盡援絕」的案例，FLOSS fund 停了 Pocketbase 的贊助
- **[Why JavaScript String Length Lies to You](https://dev.to/vftiago/why-javascript-string-length-lies-to-you-2a9a)** — `'🇹🇼'.length === 4` 這種事到現在還有人不知道，UTF-16 的歷史債永遠在
- **[What developers don't get about Idempotency](https://dev.to/manuelarte/what-developers-dont-get-about-idempotency-1hgm)** — DELETE 回 404 算不算冪等？如果你猶豫了，這篇值得看
- **[Signals Made Angular Faster — But Also Easier to Misuse](https://dev.to/mridudixit15/signals-made-angular-faster-but-also-easier-to-misuse-2dii)** — Angular Signals 讓效能變好了，但也讓開發者更容易寫出反模式——每個框架的宿命
- **[GHSA-RWJ8-P9VQ-25GV: BlueBubbles Path Traversal](https://dev.to/cverports/ghsa-rwj8-p9vq-25gv-openclaw-bluebubbles-when-your-imessage-bridge-becomes-a-spy-1m72)** — iMessage bridge 的 Path Traversal，CVSS 8.6，傳個訊息就能讀你的本機檔案
- **[How I Built a Type-Safe Excel Library with Zod](https://dev.to/tyson_cung/how-i-built-a-type-safe-excel-library-with-zod-1hac)** — 用 Zod schema 驗證 Excel 匯入，終於不用再手動 parse 每一欄然後祈禱

---

## 📚 慢慢啃

- **[R3forth: A concatenative language derived from ColorForth](https://github.com/phreda4/r3/blob/main/doc/r3forth_tutorial.md)** — 從 ColorForth 衍生出來的 concatenative language，極簡主義到極致，適合週末研究語言設計的底層美學
- **[27-year-old Apple iBooks can connect to Wi-Fi and download official updates](https://old.reddit.com/r/MacOS/comments/1r8900z/macos_which_officially_supports_27_year_old/)** — 1999 年的 iBook 還能連 Wi-Fi 下載官方更新，Apple 的硬體支援週期長到不可思議（或者說當年的軟體寫得真好）
- **[Virtually bootstrapping a virtual OS](https://dev.to/treytomes/virtually-bootstrapping-a-virtual-os-4158)** — 從零開始寫 bootloader，一步步從 BIOS 到 kernel handoff，如果你好奇開機時到底發生了什麼事
- **[TypeScript 类型体操练习笔记（二）](https://juejin.cn/post/7606973101919879195)** — 188 題 TypeScript 型別體操刷到第 90 題的心得，ReplaceKeys、Merge 這些 Medium 難度的型別操作值得跟著做一遍
