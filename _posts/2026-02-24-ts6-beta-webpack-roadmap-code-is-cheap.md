---
title: "TypeScript 6.0 Beta 清場中、webpack 2026 還沒死、寫 code 變太便宜了"
date: 2026-02-24
description: "TypeScript 6.0 Beta 為 Go 驅動的 TS7 鋪路、webpack 公布 2026 路線圖宣告自己還活著、Simon Willison 說寫程式變便宜但好程式碼依然昂貴"
tags: [frontend, typescript, tooling, react, ai, nodejs]
---

今天的主題是「清理與重新定位」。TypeScript 6.0 Beta 出了，但它不是來給你新功能的——它是來幫你清 tsconfig 的，因為 Go 版 TS7 年底要來了。webpack 公布 2026 路線圖，證明自己還有心跳。然後 Simon Willison 寫了一篇很值得靜下來想的東西：當寫 code 變得幾乎免費，工程師的價值到底在哪？

---

## 🔧 今日硬菜

### [栗子前端技術周刊第 117 期 — TypeScript 6.0 Beta、webpack 2026 路線圖、React 生態調查](https://juejin.cn/post/7608750276679188516)

這期周刊資訊密度很高，直接拆重點。TypeScript 6.0 Beta 本質上是個「清場版本」：`--strict` 預設開啟、型別預設值改為 `[]`、一堆破壞性變更和棄用項。微軟擺明了在替年底的 Go 驅動原生 TypeScript 7 做準備——先把歷史包袱甩掉。如果你的 tsconfig 還在用一堆過時 flag，現在不清以後會更痛。

webpack 2026 路線圖更有意思：原生 TypeScript 支援（不需要 loader）、內建 CSS Modules（不需要 plugin）、通用編譯目標。Vite 黨先別笑——你公司那個 2019 年的 monorepo 大概還在跑 webpack，這些改進是有市場的。React 生態調查收集了近 4000 名開發者的回饋，React Native 0.84 把 Hermes v1 設為預設引擎，Bun v1.3.9 加了 `--parallel` 執行腳本。

**重點：**
- TypeScript 6.0 Beta 是「清 tsconfig」版，為 Go 驅動的 TS7 鋪路。`--strict` 預設啟用，一堆 flag 被棄用
- webpack 2026 路線圖：原生 TS 建置、內建 CSS Modules、通用編譯目標——在 Vite 當道的時代依然有人在用心維護
- React Native 0.84 預設 Hermes v1，Bun v1.3.9 支援平行/串行腳本執行
- 但 TS 6.0 的破壞性變更不少，升級前先跑一次 build 看看會爆幾個

### [Writing Code is Cheap Now — Simon Willison](https://simonwillison.net/guides/agentic-engineering-patterns/code-is-cheap/)

Simon Willison 這篇是 Agentic Engineering Patterns 系列的一章，講的道理很簡單但值得每個工程師認真想：coding agent 讓「把 code 打進電腦」變得幾乎免費了，但交付「好的 code」依然昂貴。我們整個產業的習慣——設計、估時、trade-off 決策——都建立在「寫 code 很貴」這個前提上。當這個前提被打破，從宏觀的專案規劃到微觀的「要不要多寫個 test」都需要重新校準。

他定義的「好 code」很務實：能跑、確認能跑、解決對的問題、錯誤處理到位、簡潔、有測試、有文件、設計考慮未來但不過度設計。這些東西 agent 能幫，但最終責任還是在開發者身上。他的建議是：任何「不值得花時間做」的直覺反應，都該先丟個 prompt 試試——最壞就是浪費幾分鐘 token。

**重點：**
- 「寫 code」便宜了，但「好 code」的標準沒降：能跑、確認能跑、有測試、有文件、設計合理
- 平行 agent 讓一個人可以同時在多個地方 implement、refactor、test、寫文件
- 新的習慣正在被建立中——整個產業都在摸索，沒有標準答案
- 但別被「便宜」沖昏頭：agent 降低的是打字成本，不是判斷成本

### [Next.js Codebase Analysis (1) — Render Callstacks](https://dev.to/jade_chou/nextjs-codebase-analysis-1-render-callstacks-5gbk)

有人直接去挖 Next.js 原始碼，追蹤從 HTTP request 到最終 `sendRenderResult` 的完整 render pipeline。從 `handleRequest` 開始，經過 `renderImpl` → `renderToResponse` → `renderPageComponent` → `findPageComponents`，一路追到 `.next/server/app` 底下的 SSR bundle 怎麼被載入和執行。App Router 的 `ComponentMod.handler` 怎麼接手渲染、`handleResponse` 怎麼從 `routeModule` 拿快取資料、PPR 流程在哪個節點分歧——全部有 code path。

這種文章的價值不在於你會不會去改 Next.js 原始碼，而是當你 debug production 的 SSR 問題時，知道 callstack 長什麼樣、cache 在哪一層、render result 怎麼被送出去。踩過 Next.js 快取地獄的人都知道，理解這些內部機制能省你幾個通宵。

**重點：**
- 完整追蹤 Next.js render pipeline：`handleRequest` → `renderImpl` → `renderPageComponent` → `sendRenderResult`
- `findPageComponents` 透過路徑比對在 `.next/server/app` 底下找到對應的 SSR bundle
- App Router 的 `ComponentMod.handler` 負責實際渲染，PPR 和非 PPR 路徑在 `handleResponse` 分歧
- 系列文第一篇，後續還會繼續挖——如果你用 Next.js 寫生產環境，值得追

---

## ⚡ 一句話帶過

- **[How I Found a CSS Bug on Etsy's Engineering Blog](https://dev.to/kevinlu-swe/how-i-found-a-css-bug-on-etsys-engineering-blog-k0f)** — 打開 DevTools 發現 `overflow: hidden` 吃掉最後一個選項，小 bug 但除錯過程很教科書
- **[Why We Chose Astro over SvelteKit](https://dev.to/hostingsift/why-we-chose-astro-over-sveltekit-for-our-hosting-comparison-platform-3cc6)** — 內容站選 Astro 不選 SvelteKit，結論不意外但選型思路值得參考
- **[Building a React Native App for 20+ Languages: Lessons in i18n](https://dev.to/pocket_linguist/building-a-react-native-app-for-20-languages-lessons-in-i18n-378d)** — RN 多語系不只是丟 JSON，RTL 排版、字體 fallback、plural rule 才是地獄
- **[Using a Headless CMS with Angular and Analog Content Loaders](https://dev.to/brandontroberts/using-a-headless-cms-with-angular-and-analog-content-loaders-21e7)** — Angular 生態的 Analog framework 整合 Headless CMS，SSG 的另一種選擇
- **[Porting Sileo's Toast System to Angular — ngx-dynamic-toast](https://dev.to/eder_avendao_fd25195a5a2/porting-sileo-to-angular-building-a-dynamic-toast-system-from-scratch-2699)** — 從 iOS 的 toast 設計移植到 Angular，過程比結果有趣
- **[AI Agent That Makes Any Next.js App Multilingual in 3 Minutes](https://dev.to/kashifrezwi/i-built-an-ai-agent-that-makes-any-nextjs-app-multilingual-in-3-minutes-4bdm)** — 三分鐘自動多語系化聽起來太美好，但 demo 確實能動
- **[AI Is Destroying Open Source, and It's Not Even Good Yet](https://www.youtube.com/watch?v=bZJ7A1QoUEI)** — HN 熱門影片，討論 AI 對開源生態的衝擊——標題聳動但論點有料
- **[Introducing EnvGuard: Catch .env Mistakes Before They Break Your App](https://dev.to/deyemie/introducing-envguard-catch-env-mistakes-before-they-break-your-app-32mg)** — 上線前驗證 .env 是否完整，簡單但該有的工具
- **[How to Test LLM Integrations in CI Without Burning Tokens](https://dev.to/akarshc/how-to-test-llm-integrations-in-ci-without-burning-tokens-1ibh)** — 用 mock 和 snapshot 在 CI 裡測 LLM 整合，不燒錢的務實做法
- **[Why Kiro Looks Unassuming: Design Philosophy vs Claude Code and Cursor](https://dev.to/aws-builders/why-kiro-looks-unassuming-organizing-design-philosophy-differences-in-the-age-of-claude-code-and-1dp9)** — AWS 的 Kiro 走 spec-driven 路線，跟 Cursor 和 Claude Code 的 agent-first 哲學不同

---

## 📚 慢慢啃

- **[Everybody Knows That Drizzle Is the Word!](https://dev.to/rubenoalvarado/everybody-knows-that-drizzle-is-the-word-5f75)** — 從 Prisma 跳到 Drizzle ORM 的深度比較，如果你在糾結 ORM 選型，這篇能幫你省幾天評估時間
- **[Building a Real-Time Video Conferencing App with WebRTC, Node.js, and Socket.IO](https://dev.to/snehaa1989/building-a-real-time-video-conferencing-web-app-with-webrtc-nodejs-and-socketio-2neg)** — WebRTC 從零搭視訊通話的完整教學，signaling server 到 ICE candidate 都有走過一遍
- **[The Terminal Renaissance: Why CLI Tools Are Eating Dev Workflows in 2026](https://dev.to/hassanjan/the-terminal-renaissance-why-cli-tools-are-eating-dev-workflows-in-2026-5a7)** — 2026 年 CLI 工具為什麼又紅了？從 AI agent 到 TUI 框架的生態盤點
- **[Voice AI Integration: From Silent Pixels to Conversational UI with Whisper](https://dev.to/programmingcentral/voice-ai-integration-from-silent-pixels-to-conversational-ui-with-whisper-3ii8)** — 用 Vercel AI SDK + Whisper 在前端做語音對話 UI，語音介面入門的好起點
