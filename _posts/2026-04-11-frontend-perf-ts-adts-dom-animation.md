---
title: "前端效能從 5.2 秒砍到 1.9 秒、TypeScript ADT 防呆支付流程、20KB 動畫引擎挑戰 Lottie"
date: 2026-04-11
description: "Vue 3 電商頁面效能優化實戰全紀錄、用 TypeScript 判別聯合型別讓無效狀態編譯不過、一個 20KB 的 DOM 原生動畫引擎想取代 280KB 的 Rive"
tags: [frontend, typescript, css, web-platform, tooling]
---

今天三道硬菜都跟「少即是多」有關。一篇把電商首屏從 5.2 秒壓到 1.9 秒，靠的不是什麼黑科技而是把「網路→資源→渲染→計算」四層搞清楚；一篇用 TypeScript 的判別聯合讓支付流程裡的非法狀態直接編譯不過——比 runtime 炸掉優雅多了；還有人嫌 Rive 的 280KB runtime 太肥，自己寫了個 20KB 的 DOM 原生動畫引擎。膽子不小。

---

## 🔧 今日硬菜

### [前端性能优化：从"术"到"道"的完整修炼指南](https://juejin.cn/post/7626960753633902638)

這篇來自掘金的長文切中了一個痛點：面試時「懶加載、CDN、壓縮合併」背得滾瓜爛熟，真碰到首屏白屏 5 秒的頁面卻無從下手。作者用一個電商詳情頁從 5.2 秒優化到 1.9 秒的完整案例，把「術」和「道」拆成兩層——術篇給你可以直接抄的 Vue 3/Vite 配置、Web Worker 元件和快取策略；道篇建立「網路→資源→渲染→計算」四層優化模型，從測量、驗證到監控形成閉環。踩過效能優化坑的人都知道，最怕的不是不會優化，是優化完三個月又退化回原點。

**重點：**
- 四層優化模型把問題分類搞清楚——是網路慢、資源大、渲染卡還是計算重，打不到七寸就是瞎忙
- Vue 3 + Vite 的實戰配置可以直接搬，包含 Web Worker 拆分計算密集任務的具體寫法
- 但是⋯⋯這篇是簡體中文、以面試導向為主軸，部分建議偏教科書；真正的效能瓶頸往往藏在你的業務邏輯裡，不是套公式能解的

### [I built a 20KB Motion Engine because Svgator, Rive and Lottie were too heavy for the DOM](https://dev.to/habibeba/i-built-a-20kb-motion-engine-because-svgatorrive-and-lottie-were-too-heavy-for-the-dom-e6n)

Rive 的 runtime 起跳 280KB+，Lottie 也不輕，而且它們都是 Canvas-based 的黑盒——搜尋引擎看不到裡面的東西，螢幕閱讀器也抓瞎。這位開發者做了一個叫 Fluv 的「語意化動畫引擎」，直接在 DOM 裡操作 SVG path，runtime 壓到 20KB。它用模組化架構做到 selective loading——你的動畫只用到 easing 就不會載入 staggering 的程式碼。用數學曲線取代原始 frame data，讓壓縮邏輯取代巨大的 JSON 檔案。

**重點：**
- 20KB vs 280KB，DOM 原生意味著 SEO 可爬、CSS/JS 可互動、瀏覽器原生渲染
- 支援 path morphing、transform、複雜 staggering 和自訂 easing function
- 但是⋯⋯這還是個早期專案，生態系跟 Lottie/Rive 的設計師工具鏈比差太遠。Landing page 用用可以，production app 裡大規模動畫還是得考慮設計師的工作流程

### [Algebraic Data Types in TS: Indestructible Payment Flows](https://dev.to/campadev/algebraic-data-types-in-ts-indestructible-payment-flows-jak)

`status: string` 配一堆 optional field——這種 interface 在支付流程裡就是定時炸彈。一個手誤把 `"pending"` 打成 `"pednign"`，runtime 不會報錯，但你的錢會不見。這篇用 TypeScript 的 discriminated union 把每個狀態綁定對應的資料：`pending` 只有 `intentId`、`success` 才有 `transactionId` 和 `receiptUrl`、`failed` 才有 `errorCode`。配合 exhaustive switch 裡的 `never` type，新增狀態忘記處理？編譯直接擋下來。作者說得好：「A string is a promise. A union is a contract.」

**重點：**
- Discriminated union 讓非法狀態在型別層面就不可能存在，零 runtime overhead
- `never` type 做 exhaustive check 是 TypeScript 裡最被低估的技巧之一
- 但是⋯⋯這個 pattern 在狀態數量爆炸時（比如 20+ 種狀態的訂單流程）會讓 union type 變得很長，需要搭配 type alias 和 helper function 管理複雜度

---

## ⚡ 一句話帶過

- **[The JavaScript Runtime: Fixing the Mental Model](https://dev.to/marshateo/the-javascript-runtime-fixing-the-mental-model-5f5b)** — 「JavaScript 是單執行緒」這句話技術上沒錯，但它解釋不了你觀察到的一半行為。這篇從實驗出發重建心智模型。
- **[Notifee is Archived. Here's a Maintained, New-Architecture Drop-in Replacement](https://dev.to/marco_crupi/notifee-is-archived-heres-a-maintained-new-architecture-drop-in-replacement-3ib5)** — Invertase 正式封存了 Notifee，React Native 的通知方案又要搬家了。好消息是有人接手了，壞消息是你又要改 native config。
- **[JSON formatter Chrome plugin now closed and injecting adware](https://github.com/callumlocke/json-formatter)** — 又一個 Chrome extension 被賣掉然後注入廣告軟體的故事。133 個 HN points，71 則留言，大家都很火大。定期檢查你裝的 extension 吧。
- **[How to Build an Accessible Data Table in 2026](https://dev.to/elonr/how-to-build-an-accessible-data-table-in-2026-j94)** — 拜託不要再用 div 模擬表格了。`<table>` 活得好好的，而且螢幕閱讀器真的需要它。
- **[Anomaly alert configuration now available](https://vercel.com/changelog/anomaly-alert-configuration-now-available)** — Vercel 終於讓你細粒度設定異常警報了，可以指定專案、HTTP 狀態碼和路由。Next.js 部署在 Vercel 上的可以少被半夜吵醒幾次。
- **[Stop Writing Old JavaScript — Start Using Modern Built-in APIs (Part 2)](https://dev.to/nagendrababun/stop-writing-old-javascript-start-using-modern-built-in-apis-part-2-2729)** — `Object.hasOwn()` 取代 `hasOwnProperty`、`structuredClone` 取代深拷貝⋯⋯2026 年了，該更新肌肉記憶了。
- **[Why Math.random() Will Fail Your Next Security Audit](https://dev.to/tim_o_5617baa5171354e/why-mathrandom-will-fail-your-next-security-audit-4h2c)** — 如果你在 production 用 `Math.random()` 產生任何跟安全沾邊的東西，不是「可能」過不了 audit，是「一定」過不了。用 `crypto.getRandomValues()`。
- **[WordPress 7.0: The Good, the AI, and the Still Missing](https://dev.to/adamgreenough/wordpress-70-the-good-the-ai-and-the-still-missing-4oa8)** — WordPress 7.0 本來要在 WordCamp Asia 發布，結果因為即時協作的資料儲存問題延期了。新排程四月底出來。AI 功能加了，但核心問題還是那些。
- **[We Scanned 6 SaaS Starter Templates — Here's What We Found](https://dev.to/sstart/we-scanned-6-saas-starter-templates-heres-what-we-found-116d)** — 掃了 Next.js、Remix、SvelteKit 的 6 個 starter template，用 9 個 production readiness 指標評分。結論：大部分 starter 離 production-ready 還有段距離。
- **[E2E Email Testing with Playwright and Cypress — No Gmail Credentials Required](https://dev.to/qasim157/e2e-email-testing-with-playwright-and-cypress-no-gmail-credentials-required-33ig)** — 用真實的測試信箱取代 mock SMTP，本地測試過了但 staging 炸了的日子可以結束了。

---

## 📚 慢慢啃

- **[JavaScript Event Loop Series: Building the Event Loop Mental Model from Experiments](https://dev.to/marshateo/javascript-event-loop-series-building-the-event-loop-mental-model-from-experiments-4d8i)** — 一整個系列，用實驗法拆解 `setTimeout`、`Promise`、`requestAnimationFrame` 的實際行為差異。不是又一篇「event loop 懶人包」，是讓你自己跑實驗建立直覺。
- **[How I automated 62% of Europe's RGAA accessibility criteria](https://dev.to/chabane_lemared_4cf92157/how-i-automated-62-of-europes-rgaa-accessibility-criteria-3pc3)** — 歐洲無障礙法 2025 年生效，法國標準 RGAA 有 106 條檢查標準。這位開發者自動化了其中 62%。如果你的產品有歐洲使用者，這篇值得花時間讀。
- **[I Built a Real-Time Multiplayer Bingo Engine with Next.js, Supabase, and Ably](https://dev.to/forrestmiller/i-built-a-real-time-multiplayer-bingo-engine-winextjs-webdev-javascript-reactth-nextjs-2og1)** — Next.js + Supabase + Ably 做即時多人遊戲，最多 20 人同時玩。架構決策的過程比成品本身更值得看——什麼時候用 Supabase realtime、什麼時候需要 Ably 這種專門的 pub/sub。
