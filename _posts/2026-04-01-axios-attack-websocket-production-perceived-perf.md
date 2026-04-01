---
title: "Axios 被植入木馬、WebSocket 生產環境實戰、你的 App 為什麼感覺很慢"
date: 2026-04-01
description: "Axios 遭供應鏈攻擊植入 RAT 木馬，影響所有 npm install 用戶；Node.js WebSocket 生產環境 Socket.IO vs ws 完整比較；前端感知效能的八個常見錯誤與修正方式。"
tags: [frontend, nodejs, tooling, web-platform]
---

今天的頭條是個讓人脊背發涼的消息——axios 被植入了遠端存取木馬。對，就是那個你每個專案都在用的 axios。另外兩件值得聊的：WebSocket 上生產環境到底該選 Socket.IO 還是 ws，以及為什麼你的 App 明明很快但用戶就是覺得慢。

---

## 🔧 今日硬菜

### [The Axios Attack Proved Vibe Coding's Biggest Blind Spot](https://dev.to/agentkit/the-axios-attack-proved-vibe-codings-biggest-blind-spot-1mmh)

3 月 30 日，axios 的維護者帳號 `jasonsaayman` 被劫持，攻擊者發布了 `axios@1.14.1` 和 `axios@0.30.4` 兩個惡意版本。這不是什麼理論上的風險——是真真切切的遠端存取木馬（RAT），跨平台通殺 Windows、macOS、Linux。`npm install` 跑下去兩秒鐘，木馬就已經在你機器上了。更可怕的是，如果你讓 AI 助手幫你自動跑 `npm install`，你連惡意安裝的畫面都沒看到。

這已經是三月份第二次重大 npm 供應鏈攻擊了（第一次是 3/19 的 Trivy/CanisterWorm）。踩過坑的都知道，`^` 和 `~` 在 package.json 裡就是定時炸彈。這次攻擊完美展示了 vibe coding 的盲區：AI 幫你選依賴、跑安裝、測試通過就交差，但它完全不會檢查套件發布時間、維護者帳號異動、postinstall script 內容。

**重點：**
- 攻擊透過隱藏依賴 `plain-crypto-js@4.2.1` 的 postinstall script 執行，影響窗口約 2-3 小時
- 立即行動：`npm config set save-exact true` 鎖定版本、CI 用 `npm ci` 取代 `npm install`、`.npmrc` 加上 `ignore-scripts=true`
- 但是... 72 小時版本冷卻期和禁用 lifecycle scripts 會影響某些合法套件（如 sharp），需要額外維護 allowlist

### [Node.js WebSockets in Production: Socket.IO vs ws, Scaling, and Reconnection Strategies](https://dev.to/axiom_agent/nodejs-websockets-in-production-socketio-vs-ws-scaling-and-reconnection-strategies-5b68)

WebSocket 在開發環境跑得好好的，上了生產環境就是另一回事。這篇把 `ws` 和 Socket.IO 的選型、水平擴展、reconnection、backpressure 一次講清楚，而且附了可以直接抄的 production-ready code。

核心結論：`ws` 原始吞吐量比 Socket.IO 快 3-5 倍，但 Socket.IO 內建 rooms、自動 reconnection、Redis adapter 水平擴展。如果你在做聊天、協作編輯這類功能，Socket.IO 的運維優勢遠大於那點效能差距。如果你在做高頻交易或 IoT sensor stream，才需要 `ws` + 自定義 protocol。

**重點：**
- 單一 Node.js process 大約能撐 10,000-50,000 個 WebSocket 連線，之後需要 Redis pub/sub backplane 做水平擴展
- Reconnection 一定要加 exponential backoff + jitter，不然 server 重啟時會被 thundering herd 打死
- 但是... 文章沒提到 WebSocket 與 HTTP/2 Server-Sent Events 的取捨——很多「即時」需求其實 SSE 就夠了，不需要全雙工

### [Things That Instantly Make a Web App Feel Slow (Even If It's Not)](https://dev.to/rohith_kn/things-that-instantly-make-a-web-app-feel-slow-even-if-its-not-kic)

Lighthouse 分數全綠，用戶還是嫌慢？因為感知效能和實際效能是兩回事。這篇列了八個最常見的「感覺慢」原因：沒有即時 UI feedback、layout shift、缺少 skeleton loader、過長的動畫 duration、全頁刷新。每一條都是踩過才知道痛的。

說實話，這些都不是什麼新概念，但就是有無數專案在犯同樣的錯。最關鍵的一條：`transition: all 1s ease` 改成 `0.2s`，用戶感知就差了十萬八千里。還有，debounce API call 但讓 input state 立即更新——這個 pattern 很多人搞反了，先擋住了用戶輸入再去 debounce，完全本末倒置。

**重點：**
- 每個互動必須在 100ms 內有視覺回饋，否則用戶會覺得 app 壞了
- Skeleton loader 不只是好看，它穩定了 layout 防止 CLS（Cumulative Layout Shift）
- 但是... 文章全用 React 範例，沒有提到 CSS `content-visibility` 和 `contain` 這類瀏覽器原生的效能優化手段

---

## ⚡ 一句話帶過

- **[Anthropic accidentally leaked Claude Code's source code](https://dev.to/aws-builders/anthropic-accidentally-leaked-claude-codes-source-code-heres-what-that-means-2f89)** — npm 包體積從 17MB 暴漲到 60MB，因為忘了剝離 source map。51 萬行 TypeScript 就這樣公開了，build pipeline 的 .gitignore 不能省
- **[20 Niche CSS Libraries for 2026](https://dev.to/butterflycss/20-niche-css-libraries-for-2026-f5)** — 厭倦了每個網站長得一樣？這裡有 20 個小眾 CSS 庫，從 classless minimalist 到動畫特效，適合週末拿來玩
- **[Next.js + Nest.js 構建全栈自動化數據分析 AI Agent](https://juejin.cn/post/7623338525065216036)** — 從資料上傳到自動清洗、智能分析、視覺化圖表的完整 pipeline，附保姆級教學。架構規劃比大部分「AI 玩具」專案認真得多
- **[Blind npm install Execution Risks Security Vulnerabilities](https://dev.to/denlava/blind-npm-install-execution-risks-security-vulnerabilities-review-lockfiles-to-mitigate-threats-2nid)** — 配合今天的 axios 事件服用更佳：你上次認真看 lockfile diff 是什麼時候？
- **[Building a Chrome Extension with Zero Unnecessary Permissions](https://dev.to/pmestreforge/building-a-chrome-extension-with-zero-unnecessary-permissions-3cj5)** — 一個 tab manager 要「讀取所有網站資料」的權限？這位開發者證明了做 extension 不需要過度索權
- **[Building Secure Session Management in NestJS](https://dev.to/bismark66/building-secure-session-management-in-nestjs-refresh-tokens-device-tracking-session-1h32)** — Refresh token + device tracking + session revocation，NestJS 認證系統的正經做法，不是只驗 JWT signature 就完事
- **[JSSE: A JavaScript Engine Built by an Agent](https://p.ocmatos.com/blog/jsse-a-javascript-engine-built-by-an-agent.html)** — AI agent 從零寫了一個 JavaScript 引擎。能跑多少 test262 case 不好說，但這個實驗本身就很有趣
- **[How I Built a Privacy-First Offline PWA Expense Tracker](https://dev.to/pda-dp-shop/how-i-built-a-privacy-first-offline-pwa-expense-tracker-3kob)** — 沒有 server、沒有 cloud、所有資料留在裝置上。PWA + IndexedDB 的正確打開方式
- **[My AI Coding Assistant Misapplied the Design Principle I Gave It](https://dev.to/odakin/my-ai-coding-assistant-misapplied-the-design-principle-i-gave-it-olh)** — 你告訴 AI「用這個設計模式」，它用了，但用錯了。Claude Code 遇上 6,933 種時間格式的故事
- **[OpenAI closes funding round at an $852B valuation](https://www.cnbc.com/2026/03/31/openai-funding-round-ipo.html)** — 8,520 億美元估值，122B 融資。數字大到已經不像科技公司，像是在看國家 GDP 排名

---

## 📚 慢慢啃

- **[The Event Loop You're Already Using](https://dev.to/nazq/the-event-loop-youre-already-using-5305)** — 從 select、poll 到 epoll，把 async 框架背後的系統呼叫講透。讀完你會真正理解 `await fetch()` 底下到底發生了什麼，不再只是背面試答案
- **[Claude Code's Compaction Engine: What the Source Code Actually Reveals](https://dev.to/johnib/claude-codes-compaction-engine-what-the-source-code-actually-reveals-2o4)** — 趁著 source map 洩漏，有人把 Claude Code 的 context 壓縮機制拆開來看。對你理解 AI agent 的 context management 很有幫助
- **[REST vs GraphQL vs WebSockets vs Webhooks: A Real-World Decision Guide](https://dev.to/rosewabere/rest-vs-graphql-vs-websockets-vs-webhooks-a-real-world-decision-guide-with-code-2bem)** — 面試和 system design 都會問的經典題，但這篇用實際場景講「什麼時候選什麼」，不是照搬定義
- **[性能优化之项目实战：从构建到部署的完整优化方案](https://juejin.cn/post/7623322130491457582)** — 九大前端效能優化策略的系統性實戰，從 Webpack Bundle Analyzer 到運行時優化，適合拿來對照自己的專案查漏補缺
