---
title: "Next.js 登入的 timing attack 沒人講、現代 CSS 九招救你 47 行 JS、CSS Breakpoint Units 想取代 rem"
date: 2026-04-19
description: "今天值得聊的三件事：Next.js 15 登入端點背後那個沒人提的 timing attack 防禦、2026 該收編的九個現代 CSS 特性，以及一個想用像素重寫斷點系統的新套件。"
tags: [frontend, react, nextjs, css, tooling]
---

今天值得聊的就這幾樣。有人終於把 Next.js 15 的登入流程寫到 timing attack 那層，這是多數教學自動跳過的段落；然後又一篇「2026 該會的 CSS 特性」——這次真的沒灌水，九個都打中痛點；最後有個人想用 `tan(atan2())` 數學魔法把 media query 拆掉換成像素驅動的斷點系統，立意驚人、文風更驚人。

---

## 🔧 今日硬菜

### [How to build a login flow in Next.js 15 (sessions, cookies, CSRF, and the timing attack nobody talks about)](https://dev.to/thegdsks/how-to-build-a-login-flow-in-nextjs-15-sessions-cookies-csrf-and-the-timing-attack-33pm)

這篇是整個系列第四篇，講登入端點。重點不在 bcrypt 或 cookie 那幾個老生常談的 flag，而是那個幾乎所有教學都會漏掉的細節：**當 email 不存在時，你必須還是跑一次 `bcrypt.compare()`，用一個預先 hash 好的假值**。否則你的 endpoint 就變成 user enumeration 的時序神諭——curl 一下就能量到 60ms 的落差，攻擊者直接拿 wordlist 把你整張使用者表 enum 出來。作者還順便把 DB 只存 token 的 SHA-256 hash、`sameSite: lax` 剛好擋住大部分 CSRF、以及為什麼 session 不能綁 IP（電信 NAT 會換）這些細節一起交代清楚。

**重點：**
- session token 在 wire 上是明文 cookie，但 DB 裡只存 SHA-256 hash——DB 外洩也不能當 cookie replay
- 登入端點必恆時跑 bcrypt.compare，找不到 user 時 fallback 到 FAKE_HASH（實測差 3ms vs 沒做防護時差 60ms）
- 認證失敗永遠回同一句 "Email or password is incorrect"，別貼心告訴對方是哪個錯
- 但是...作者用自家 `kavachOS` 當範例容器，商業廣告味偏重，抄 pattern 可以，工具自己評估

### [10 CSS tricks that feel illegal to know in 2026](https://dev.to/mamoor123/10-css-tricks-that-feel-illegal-to-know-in-2026-96g)

標題很 click-baity 但內容實在。整理了 2026 該會的九個 CSS 特性（標題寫十但我算過是九個，無妨），每個都能幹掉一段 JS。`:has()` 終於讓 CSS 有了反向選擇器——`.card:has(img)` 一行就能判斷「有圖的卡片」，以前你要嘛 querySelectorAll 要嘛用 JS class 補丁。Container queries 讓同一個 widget 在 300px 側邊欄和 900px 主區域自動長不同樣子，不用再盯著 viewport 寫 media query。`text-wrap: balance` 一行解決標題奇行怪字；`accent-color` 讓你別再手刻一個 200 行的 checkbox 元件來換顏色；`color-mix()` 取代 Sass 變體函式；`@layer` 終於把 specificity 戰爭講清楚。

**重點：**
- `:has()` 和 container queries 是近五年最重要的兩個 CSS 特性，沒之一
- scroll-driven animations (`animation-timeline: view()/scroll()`) 能直接幹掉 GSAP ScrollTrigger 的入門用法，而且在 compositor 上跑
- `subgrid` 解決「卡片內部元素要對齊跨卡片」這個多年老問題
- 但是...別把這些當銀彈，`:has()` 寫複雜時選擇器效能還是會炸，container queries 需要設定 `container-type` 會影響佈局計算

### [CSS Breakpoint Units - design with pixels and get fluid UX for free while automatically solving two of the oldest accessibility problems](https://dev.to/janeori/css-breakpoint-units-design-with-pixels-and-get-fluid-ux-for-free-while-automatically-solving-two-4mdj)

作者 PropJockey (Jane Ori) 一向以 CSS hack 聞名，這次她用自己發明的 `tan(atan2())` 技巧把 `<length>` 強轉成 `<number>`，然後把 `100cqw` 當數字拿去跟使用者設定的 10 個像素斷點比較，產出 0/1 的位元旗標，最後用 `calc()` 開關、`@container style()` 查詢、或 `if(style())` 來做響應式佈局。聲稱可以取代 rem 在斷點上的地位，還自動處理 scrollbar 進佔斷點、以及系統字級放大時自動「降一個斷點但視覺放大」的 a11y 行為。立意真的有料，91% 瀏覽器支援度也夠用。但是⋯文章中後段整個飄起來——「更高的力量給我這個靈感」、「心是敞開的接收豐盛」、附一串加密貨幣捐款地址——風格衝擊力遠大於技術說明。想認真評估的直接看 CodePen demo 和套件 repo。

**重點：**
- 底層是 `tan(atan2())` scalar 的 CSS 黑魔法，把 length 轉 number 來跑數學比較
- 三種 API 路徑可以選：`calc()` switch、`@container style()` 查詢、`if(style())`
- 自動修正 scrollbar gutter 吃掉視窗寬度、和系統 font-size 放大的 a11y 問題
- 但是...作者個人風格非常「特別」，建議把套件跟作者人設分開看，純粹用 demo 評估值不值得引進

---

## ⚡ 一句話帶過

- **[How to build a register user flow in Next.js 15 (frontend, backend, database, email)](https://dev.to/thegdsks/how-to-build-a-register-user-flow-in-nextjs-15-frontend-backend-database-email-50b5)** — 同一作者的註冊篇，跟登入篇配一起看才完整，從驗證信到 magic link 都有
- **[How to build a secure password reset flow in Next.js (the short version)](https://dev.to/thegdsks/how-to-build-a-secure-password-reset-flow-in-nextjs-the-short-version-ja6)** — 密碼重設 token 要單次使用、要 hash 存、要過期，這三點做不對就等於幫攻擊者留後門
- **[Headless CMS for TanStack Start: Build a Blog with Cosmic](https://dev.to/tonyspiro/headless-cms-for-tanstack-start-build-a-blog-with-cosmic-k6f)** — TanStack Start 這個 SSR + type-safe router 框架慢慢有生態了，Cosmic 整合算是一個案例
- **[Introducing @accesslint/jest: progressive accessibility testing for Jest](https://dev.to/cameron-accesslint/introducing-accesslintjest-progressive-accessibility-testing-for-jest-3i7j)** — 漸進式 a11y 測試，讓你不用一次補完所有舊元件也能開啟 a11y gate，這個策略選得對
- **[Privacy-First: Building a 100% Local AI Medication Assistant with WebLLM and WebGPU](https://dev.to/beck_moulton/privacy-first-building-a-100-local-ai-medication-assistant-with-webllm-and-webgpu-2jk5)** — WebLLM + WebGPU 在瀏覽器跑 LLM 的教學示範，模型下載量不小但確實是隱私敏感場景的解法
- **[How Chrome Extensions Inject Dual Subtitles into Netflix (And Why It's Harder Than It Looks)](https://dev.to/_funlingo_/how-chrome-extensions-inject-dual-subtitles-into-netflix-and-why-its-harder-than-it-looks-2216)** — 聽起來無聊，實際是一份 SPA 路由監聽 + WebVTT + overlay 同步的實戰筆記
- **[I replaced Auth0 with an open source library in 30 minutes. Here is what broke.](https://dev.to/thegdsks/i-replaced-auth0-with-an-open-source-library-in-30-minutes-here-is-what-broke-3l2c)** — 省 $427/月爽歸爽，broke 的那幾條才是重點，別只看省錢標題
- **[How to Call Google Gemini API from Next.js (Free Tier, No Backend Needed)](https://dev.to/tahosin/how-to-call-google-gemini-api-from-nextjs-free-tier-no-backend-needed-3dng)** — 從前端直呼 Gemini 避開 serverless 10 秒逾時，能用但你的 API key 也就躺在 client bundle 裡，demo 可以，正式勸退
- **[I got tired of wiring the same caching stack every project, so I built LayerCache](https://dev.to/flyingsquirrel0419/i-got-tired-of-wiring-the-same-caching-stack-every-project-so-i-built-layercache-52e2)** — Node.js 多層快取整合包（memory + Redis），又一個 NIH 症候群產物，但 API 設計比某些成熟方案清爽
- **[VS Code Tips and Tricks That Will Seriously Boost Your Developer Experience](https://dev.to/hamidrazadev/vs-code-tips-and-tricks-that-will-seriously-boost-your-developer-experience-em5)** — VS Code 技巧合集，老手可能都會了，新進同事可以丟一份
- **[EU Age Verification App "Hacked in 2 Minutes" — What Actually Happened](https://dev.to/fldsakos/eu-age-verification-app-hacked-in-2-minutes-what-actually-happened-2d3p)** — 新聞標題跟實際技術事件之間的落差，看看 PR 是怎麼把一個 RCE 包裝成「兩分鐘被駭」的
- **[I built a local Contentful model simulator to stop testing content models blindly](https://dev.to/joshuapozos/i-built-a-local-contentful-model-simulator-to-stop-testing-content-models-blindly-n4k)** — 用 Contentful 的團隊看過來，模型改動不用再直接打 staging 試

---

## 📚 慢慢啃

- **[I've built auth six times. Here's the system I would build today](https://dev.to/thegdsks/ive-built-auth-six-times-heres-the-system-i-would-build-today-5cpf)** — 上面硬菜同一作者的 meta 文，彙整他六次從零寫 auth 累積下來的架構選型原則，值得一次讀完當 checklist
- **[How I built an open source visual QA tool after every AI agent I tried failed](https://dev.to/alexmchughdev/how-i-built-an-open-source-visual-qa-tool-after-every-ai-agent-i-tried-failed-3nf7)** — 作者把市面上 AI E2E agent 都跑過一輪發現全不堪用，最後自己開源一個——看這種文的價值在那份「為什麼現有方案都不行」的評估清單
- **[My AI Sends 30k Tokens Per Message. 80% of Them Were Wasted.](https://dev.to/juandastic/my-ai-sends-30k-tokens-per-message-80-of-them-were-wasted-1lmp)** — context engineering 實戰，從 30k token 砍到 6k 的那段 prompt 壓縮技巧，帳單要自付的人讀了會有感
- **[I Measured How Much Each Agent Design Decision Costs in Tokens (The Numbers Make Me Uncomfortable)](https://dev.to/jtorchia/i-measured-how-much-each-agent-design-decision-costs-in-tokens-the-numbers-make-me-uncomfortable-3hk7)** — 把 agent 架構每個設計選擇（工具描述長度、memory 策略、retry 邏輯）都換算成 token 成本，數字確實讓人不舒服
