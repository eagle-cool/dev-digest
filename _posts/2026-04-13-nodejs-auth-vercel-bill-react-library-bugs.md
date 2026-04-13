---
title: "Node.js 認證大洗牌、Vercel 帳單的殘酷真相、React 元件庫踩坑實錄"
date: 2026-04-13
description: "Lucia 棄用、Auth.js 併入 Better Auth、Keycloak 官方 adapter 也棄用——Node.js 認證生態正在重新洗牌。另外有人被 Vercel 寄了一張 $4,700 的帳單，還有 React 安全元件庫上線後才冒出的六個 bug。"
tags: [frontend, react, nextjs, nodejs, typescript]
---

今天值得聊的就三件事。Node.js 的認證生態正式進入戰國時代——Lucia 棄用了、Auth.js 被 Better Auth 吸收了、連 Keycloak 的官方 Node adapter 都要被移除。然後有個團隊被 Vercel 寄了一張 $4,700 的帳單，才驚覺 ISR 加 Edge Function 的組合拳有多燒錢。最後是一個 React 安全元件庫的踩坑實錄，六個 bug 全部是上線後才被真實使用者發現的——測試矩陣再完美也擋不住真實世界的多樣性。

---

## 🔧 今日硬菜

### [Self-hosted NodeJS Authentication in 2026](https://dev.to/noorix1/self-hosted-nodejs-authentication-in-2026-9ao)

這篇是目前看過最完整的 Node.js 自架認證方案橫評。比較了五個選項：nauth-toolkit、Better Auth、SuperTokens、Logto、Keycloak，而且不是那種比功能打勾勾的表面文章，是從架構層面切入——你的 auth 到底要嵌進 app process、還是拆成 sidecar、還是跑獨立的 IdP？這個選擇決定了後續所有的 trade-off。

Better Auth 吃掉 Auth.js 之後成了 embedded auth 的預設選擇，40+ social providers、plugin 生態最大，但 NestJS 和 Angular 的支援偏弱。SuperTokens 的 session 機制（JWT + refresh token rotation + theft detection）是這堆裡面最嚴謹的，可惜 MFA 要付費（$0.01/MAU，最低 $100/月）。Logto 走 standalone identity server 路線，最大賣點是零功能鎖——SSO、RBAC、MFA 全部免費。Keycloak 還是功能最完整的，但 `keycloak-connect` 被棄用，Node.js 的開發者體驗每況愈下。

**重點：**
- Better Auth 成為 JS 生態 embedded auth 的新王者，Lucia 使用者該遷移了
- SuperTokens 的 session 設計最成熟，但付費牆會讓成本快速攀升
- 但是... nauth-toolkit 雖然架構設計最激進（hybrid token、adaptive MFA、Argon2id 預設），目前還在 v0.2.4 Early Access，social provider 只有三個，拿來做 side project 可以，production 再觀望

### [The Vercel Bill Conversation Every Startup Avoids (Until It's Too Late)](https://dev.to/adioof/the-vercel-bill-conversation-every-startup-avoids-until-its-too-late-5bj6)

又一個被 Vercel 帳單嚇到的團隊。Next.js monorepo、ISR、Edge Function、Image Optimization 全部用好用滿，Lighthouse 分數 98+，然後月底收到 $4,700 的帳單。踩過這個坑的都知道，Vercel 的計費模型獎勵的是簡單架構，你搞越複雜就越貴。

三個主要燒錢點：50,000 個產品頁的 ISR revalidation 風暴（每次產品更新觸發連鎖 function invocation，預估 $200 變成 $2,800）、個人化 Edge Function 對 8 個微服務做 fan-out（請求量指數成長）、Image Optimization 按轉換次數收費（$20/1000 次，一萬張圖多尺寸 = 爆炸）。

最後解法是把 ISR 搬到 Cloudflare Pages + KV、Edge Function 搬到 Cloudflare Workers、Image Optimization 搬到 Cloudinary。同樣的效能，帳單從 $4,700 降到 $287。遷移花了三週。

**重點：**
- Vercel 的 pricing 模型在複雜架構下會指數放大成本
- ISR revalidation storm 是最大的隱形殺手——50k 頁面 × 每次 3 calls
- 但是... 這不代表 Vercel 不好，而是你得在架構設計階段就把計費模型算進去。免費的 DX 最貴

### [Six bugs that only appeared after real users installed my React security library](https://dev.to/nedunuri_anurag/six-bugs-that-only-appeared-after-real-users-installed-my-react-security-library-29mk)

FieldShield 是一個 React 安全輸入元件庫，用 Web Worker 隔離真實輸入值，DOM 永遠只存 `xxxxx`。概念很漂亮，架構也沒問題——在開發者自己的機器上。一上 npm 之後，六個 bug 接力登場。

最致命的是 Worker 檔案用了絕對路徑，在自己的 Vite 專案裡跑得好好的，消費者從 npm 安裝後直接白屏——Worker 路徑解析失敗。修法是 build time 把 Worker 內嵌成 blob URL。第二個坑是字型：開發用等寬字體 IBM Plex Mono，使用者用 Inter（proportional），30 個字元後游標跟遮罩層對不齊。第三個是 CSS 繼承地獄——消費者的 `#root` 上有 `text-align: center`（Vite 預設），一路繼承到元件內部。

**重點：**
- npm 元件庫的打包路徑問題永遠是第一殺手——`npm pack --dry-run` 是你的好朋友
- CSS 繼承屬性在 overlay 元件中會製造意想不到的 bug，至少要 reset 六個屬性
- 但是... 這些都不是測試能抓到的。最強的測試矩陣也比不上三個不同 bundler 的消費者專案

---

## ⚡ 一句話帶過

- **[栗子前端技术周刊第 124 期](https://juejin.cn/post/7627396290868396095)** — ESLint v10.2.0 支援語言感知規則、React Native 0.85 帶來新動畫能力、Node.js 25.9.0 加了 `--max-heap-size`，三個值得留意的版本更新
- **[TypeScript Utility Types That Actually Save Time in Production SaaS Code](https://dev.to/whoffagents/typescript-utility-types-that-actually-save-time-in-production-saas-code-5afd)** — 不是又一篇 cheatsheet，而是在 Next.js API、Stripe 整合、DB layer 裡實際怎麼用 `Partial<T>` 和朋友們
- **[How I Built a Zero-Buffering Video Player in React](https://dev.to/michael_dl/how-i-built-a-zero-buffering-video-player-in-react-hls-adaptive-bitrate-nn8)** — HLS + adaptive bitrate 在 React 裡做到零緩衝，15+ 裝置類型的實戰經驗
- **[How to add AI to your React Native app in 5 minutes](https://dev.to/nikapkh/how-to-add-ai-to-your-react-native-app-in-5-minutes-d14)** — `react-native-ai-hooks` 這個 npm 套件把 Claude/OpenAI 包成 React hooks，API 設計蠻乾淨的
- **[使用 VueUse 构建支持暂停/重置的 CountUp 组件](https://juejin.cn/post/7627535770001866779)** — 嫌 vue-countup-v3 只能自動播放？用 VueUse 的 composable API 自己做一個完全可控的版本
- **[I replaced 6 @Injectable() gRPC interceptors with composable factories](https://dev.to/harsh_m04/i-replaced-6-injectable-grpc-interceptors-with-composable-factories-5ggi)** — Angular monorepo 裡六個 gRPC interceptor 的 boilerplate 地獄，用 factory pattern 乾淨解決
- **[I cut 40% of my NgRx Signal Store boilerplate](https://dev.to/harsh_m04/i-cut-40-of-my-ngrx-signal-store-boilerplate-heres-the-5-utilities-i-extracted-52mn)** — 20 個 Signal Store 裡重複的 loading flag、pagination、search pipe，提取成五個 utility
- **[We Ran Four Security Tools Against Express.js](https://dev.to/copyleftdev/we-ran-four-security-tools-against-expressjs-they-found-each-others-proof-34ah)** — 單一掃描工具給你清單，四個工具交叉驗證給你 corroboration——Express 安全掃描的正確姿勢
- **[I built a free self-hosted alternative to Pusher](https://dev.to/darknautica/i-built-a-free-self-hosted-alternative-to-pusher-1n92)** — Pusher 太貴、自架 WebSocket 方案，適合需要即時功能但預算有限的團隊
- **[Angular 21 + Spring Boot 3.4 in Docker](https://dev.to/zukovlabs/angular-21-spring-boot-34-in-docker-the-plumbing-nobody-shows-you-5a7l)** — 每次都在前十分鐘踩 DB 連線順序和 proxy 設定的坑，這篇把那些沒人寫的 plumbing 講清楚了

---

## 📚 慢慢啃

- **[Real-Time Breath Detection in the Browser](https://dev.to/felix_zeller_6f3c43a7513f/real-time-breath-detection-in-the-browser-spectral-centroid-dual-path-state-machines-and-a-nasty-56bb)** — 用 Web Audio API 做即時呼吸偵測，涉及 spectral centroid 分析、雙路徑狀態機，還有一個 iOS Safari 靜默回傳空 buffer 的坑。難得一見的瀏覽器音訊深潛
- **[The Art of API Design: Principles from 50,000+ Daily Requests](https://dev.to/oluwatosinolamilekan/the-art-of-api-design-principles-i-learned-building-apis-that-handle-50000-daily-requests-157h)** — 不是那種列十條規則的水文，而是從高流量 API 的實戰經驗倒推設計原則——schema 不要洩漏、錯誤格式要統一、分頁不要用 offset
- **[The agent over-applies everything: why "don't" is my most-used word](https://dev.to/euzharkov/the-agent-over-applies-everything-why-dont-is-my-most-used-word-2omc)** — 在 1,703 次 agent 修正中，出現最多的詞是「DON'T」。如果你在用 AI coding assistant 寫 code，這篇會讓你很有共鳴
