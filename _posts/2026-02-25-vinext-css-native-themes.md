---
title: "Cloudflare 花 $1,100 用 AI 重寫 Next.js、CSS 原生四模式主題切換"
date: 2026-02-25
description: "Cloudflare 發布 ViNext，一個工程師用 AI 花一週重寫 Next.js，建構快 4 倍、bundle 小 57%。另有原生 CSS 四模式主題切換、LLM 串流語音合成架構解析。"
tags: [frontend, react, nextjs, css, ai, tooling]
---

今天值得聊的就一件事，但它夠大：Cloudflare 一個工程師用 AI 花了一週重寫 Next.js，然後真的跑起來了。其他的——原生 CSS 四模式主題切換做得很漂亮，LLM 串流語音合成的概念有意思但實作還嫩了點。

---

## 🔧 今日硬菜

### [How we rebuilt Next.js with AI in one week](https://blog.cloudflare.com/vinext/)

這不是標題黨。Cloudflare 真的用一個工程師（技術上是工程經理）加 Claude，花了不到一週重寫了 Next.js 的 API 表面。成品叫 [ViNext](https://github.com/cloudflare/vinext)，基於 Vite 建構，不是 OpenNext 那種逆向工程 Next.js build output 的做法，而是完全重新實作 routing、SSR、React Server Components、Server Actions、Middleware、Caching。

數字很漂亮：Vite 8 + Rolldown 版本建構時間比 Next.js 16 快 4.4 倍（1.67s vs 7.38s），client bundle 小 57%（72.9KB vs 168.9KB gzipped）。94% 的 Next.js 16 API 覆蓋率，1,700+ Vitest 測試 + 380 Playwright E2E 測試。整個專案花了大約 $1,100 的 token 費用。

最有意思的是 Traffic-aware Pre-Rendering（TPR）：不像 Next.js 在 build time 預渲染所有 `generateStaticParams()` 列出的頁面，ViNext 查 Cloudflare 的流量數據，只預渲染真正有人看的頁面。10 萬個產品頁？通常只有 200 頁佔了 90% 流量，剩下的用 ISR 按需渲染。這個設計思路很聰明——你的 build time 不再隨頁面數量線性成長。

**重點：**
- 基於 Vite 完全重新實作 Next.js API，不是 adapter 或 wrapper，95% 的程式碼跟 Cloudflare 無關
- Vite 8 / Rolldown 建構快 4.4 倍，bundle 小 57%，已有美國政府網站在 production 跑
- 但是...還是 experimental，連 static pre-rendering 都還沒有，而且 Cloudflare 作為 Next.js 的競爭平台方做這件事，動機值得想想

### [Two Color Schemes, Four Modes: Native CSS Theme Switching](https://dev.to/olgaurentseva/two-color-schemes-four-modes-native-css-theme-switching-17fo)

「前端終於走向 vanillaization」——這句話我聽了要起立鼓掌。作者用純原生 CSS 實作了兩組色彩方案 × 明暗模式 = 四種主題變體，零 JavaScript 控制顏色值。

核心技巧：CSS `light-dark()` 函數搭配 `oklch()` 色彩空間，加上 `:root:root:root` 的 specificity 疊加（沒錯，就是重複三次 `:root`）來打敗 styled-components 注入的樣式。第二組主題用 `.spring:root:root:root` 覆蓋，因為多了一個 class selector 所以 specificity 更高。切換只需要 `classList.toggle()`，不用 React state、不用 context、不用 re-render。

載入時在 `main.tsx` 裡同步讀 `localStorage` 並套用 class，在 React render 之前就完成，所以沒有 FOUC。作者也試過 `@container style()` queries，Chrome 可以但 Firefox 和 Safari 不行——這個 API 值得關注但現在還不能用。

**重點：**
- `light-dark()` + `oklch()` + specificity 疊加 = 原生 CSS 四模式主題，零框架依賴
- 零 React state、零 context、零 re-render，全部交給瀏覽器處理
- 但是...`light-dark()` 瀏覽器支援度還要注意，需要 `@media (prefers-color-scheme)` fallback

### [How to Build a Real-Time Talking Assistant with Next.js, Vercel AI SDK, and Web Speech API](https://dev.to/programmingcentral/how-to-build-a-real-time-talking-assistant-with-nextjs-vercel-ai-sdk-and-web-speech-api-3hbg)

概念很好，執行差了點。文章標題提到 Vercel AI SDK 和 RSC，但實際程式碼用的是 mock 資料模擬串流——連 `useChat` 都沒用到。不過拋開實作不談，背後的架構思路值得聊。

核心問題：LLM 一個 token 一個 token 吐，但 Web Speech API 需要完整句子才不會結巴。解法是在中間加一個 buffer，偵測到標點或空格才 flush 到 `SpeechSynthesisUtterance` 佇列，加上 timeout 防止延遲累積。這個 buffering + boundary detection 策略是所有串流 TTS 場景的基本功。踩過 Web Speech API 坑的人都知道：`getVoices()` 第一次呼叫會回傳空陣列（voices 是非同步載入的），iOS Safari 不允許沒有使用者互動就播放音訊，而且不同瀏覽器的語音品質差距大到你會懷疑人生。

**重點：**
- 串流 TTS 的 buffer + boundary detection 策略是正確方向，任何前端 × LLM 語音場景都用得上
- Web Speech API 的限制：跨瀏覽器一致性差、需要使用者互動觸發、語音品質參差
- 但是...這基本上是一篇書的業配，實作用 mock 而非真實 AI SDK 串流，概念參考就好

---

## ⚡ 一句話帶過

- **[5 Things to Know About Migrating Angular Tests to Vitest](https://dev.to/cristiansifuentes/5-things-to-know-about-migrating-angular-tests-to-vitest-after-moving-40-repositories-3fd0)** — 從 20+ Angular 專案遷移到 Vitest 的實戰心得，踩坑清單值得先看再動手
- **[How We Made Our E2E Tests 12x Faster](https://dev.to/alexneamtu/how-we-made-our-e2e-tests-12x-faster-51pm)** — Playwright 套件從 90 秒壓到個位數，關鍵就是別每個測試都重新登入
- **[Generating 21 Multilingual Promo Videos from React Code with Remotion](https://dev.to/shusukedev/generating-21-multilingual-promo-videos-from-react-code-with-remotion-o26)** — 用 React 寫影片然後批次輸出 21 種語言版本，這件事本身就很 React（褒義）
- **[Ng-News 26/07: Angular's Router, Vitest, Hashbrown, History & Popularity](https://dev.to/playfulprogramming-angular/ng-news-2607-angulars-router-vitest-hashbrown-history-popularity-4phc)** — Angular 本週大事：Router 改進、Vitest 整合、State of JS 衝上 10 萬星
- **[Memory Leaks in Angular: The Silent Performance Killer](https://dev.to/cristiansifuentes/memory-leaks-in-angular-the-silent-performance-killer-3ie4)** — 你的 Angular app 跑 30 分鐘後開始卡？先檢查 subscription 有沒有 unsubscribe
- **[一文搞懂 SEO 全流程技术](https://juejin.cn/post/7609891142464159780)** — 前端工程師的 SEO 技術全流程指南，從 meta tag 到 sitemap 一次講完
- **[The serverless lie: Why I refuse to default to Next.js](https://dev.to/jeremy_mahuvava_88324105f/the-serverless-lie-why-i-refuse-to-default-to-next-js-2n74)** — 今天看完 ViNext 再看這篇特別有感，Next.js 預設 serverless 的隱性成本
- **[How to generate a PDF from HTML in Node.js (without Puppeteer)](https://dev.to/custodiaadmin/how-to-generate-a-pdf-from-html-in-nodejs-without-puppeteer-3gg8)** — 不想在 dependency 裡塞 400MB Chromium 的人有福了
- **[How We Fixed Firefox's localStorage Race in Playwright](https://dev.to/papredapp/how-we-fixed-firefoxs-localstorage-race-in-playwright-two-navigation-helpers-bbi)** — Firefox 的 `addInitScript` 和 localStorage 的 race condition，解法簡單但坑很痛
- **[Tubes Cursor (WebGL, WebGPU)](https://dev.to/saborize_prime_b41a630a97/tubes-cursor-webgl-webgpu-547p)** — ThreeJS + WebGPU 做的管狀游標特效，技術含量有但請別問怎麼說服 PM 加到專案裡

---

## 📚 慢慢啃

- **[Bun 全景指南：下一代 All-in-One 运行时详解与实战](https://juejin.cn/post/7610478822881853482)** — 從 runtime 到 bundler、package manager、test runner 的完整導覽，想搞懂 Bun 到底在幹嘛的看這篇就夠了
- **[Building a Local-First Tauri App with Drizzle ORM, Encryption, and Turso Sync](https://dev.to/huakun/building-a-local-first-tauri-app-with-drizzle-orm-encryption-and-turso-sync-31pn)** — Tauri + Drizzle ORM + libSQL + 加密 + Turso 同步的完整架構，local-first 桌面應用的實戰參考
- **[Bringing Microsoft SAM Back to Life: How SAPI4 TTS Works in the Browser](https://dev.to/kaomojiya/bringing-microsoft-sam-back-to-life-how-sapi4-tts-works-in-the-browser-3ej7)** — 把 Windows 2000 的經典 TTS 引擎搬到瀏覽器裡跑，技術考古學的浪漫（而且真的能動）
