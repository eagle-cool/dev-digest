---
title: "Vercel 被打了、Dark Mode 六層境界、WebRTC 一次講清楚"
date: 2026-04-20
description: "Vercel 證實資安事件，ShinyHunters 名下宣稱已竊 GitHub token 與 NPM token，整條前端供應鏈抖三下；另有 CSS Dark Mode 六個實作層級完整盤點、WebRTC ICE/STUN/TURN 連線機制一次講透。"
tags: [frontend, css, nodejs, tooling, web-platform, ai]
---

今天的開發圈值得聊的就三件事。第一，Vercel 被人捅了，而且捅得剛好捅在前端供應鏈的動脈上——如果你把 secrets 交給他們管，今天就該去輪換。第二，有人把 dark mode 拆成六個實作等級，寫得比資安事件還仔細，正好提醒你 2026 年了 `color-scheme` 那行 meta 還是不能省。第三，有位工程師把 WebRTC 從 NAT 到 ICE candidate 到 TURN relay 一次講清楚，寫得比三年前任何一篇官方文件都好懂。剩下的大部分是 DEV Community 的 SEO 垃圾，我幫你跳過了。

---

## 🔧 今日硬菜

### [The Vercel Breach: What Actually Happened, Why It Matters, and What Every Developer Should Do Right Now](https://dev.to/freerave/the-vercel-breach-what-actually-happened-why-it-matters-and-what-every-developer-should-do-right-4mjn)

Vercel 今天確認內部系統遭未授權存取。自稱 ShinyHunters 的攻擊者在 BreachForums 開價 200 萬美金兜售，宣稱拿到了 employee 帳號、API keys、NPM tokens、GitHub tokens，還有內部 Linear 的全量資料。Vercel 官方承認入侵屬實但只說「影響有限」——這是資安事件公告的標準翻譯，意思是「我們還在算」。重點不是誰被駭，而是 Vercel 存的東西：你的環境變數、你串 GitHub 的 OAuth token、你發 npm 套件的發行 token。一個平台被打穿，下游幾千個應用全部連帶暴露，這就是 supply chain compromise 的 ROI 為什麼這麼高。

**重點：**
- GitHub OAuth token 就算是 read-only 也能把你所有 private repo clone 走，包含從沒部署過的；write 權限的 token 更可怕，可以直接 push 後門進你自己的 npm 套件
- NPM token 被拿走等於攻擊者能對任何你有發行權限的套件推送惡意版本，下游所有 CI 都中招。`npm audit` 看的是「現在的程式碼」，防不了「明天才被偷偷改」
- 「我們有 encryption at rest」不是防線。攻擊者拿到員工帳號後是用「合法」路徑請求解密，加密層根本不會擋——這就是為什麼 access control 比加密重要十倍
- 但是：攻擊者宣稱的東西 Vercel 沒全部證實，ShinyHunters 本尊還跳出來否認這是他們幹的。可以等官方完整報告，但該輪換的憑證今天就要輪換

### [Six Levels of Dark Mode](https://cssence.com/2024/six-levels-of-dark-mode/)

CSSence 把 dark mode 的實作方式拆成六個等級，從「只加一行 meta tag」（Level 1: Barebone）到用 `prefers-color-scheme` 搭 CSS variables 做完整切換（Level 5: Bisectional），再到用新的 `light-dark()` 函式把整份 stylesheet 寫一遍（Level 6: Ballistic）。最有啟發的是 Level 1——很多網站今年還是沒加 `<meta name="color-scheme" content="light dark">` 這行，結果 browser 根本不知道你支援深色模式，連捲軸和預設 form 元件都給你用錯色。這篇的寫法很工程師：每一層都有可執行的 code、有 trade-off、也承認 Safari 舊版的坑。

**重點：**
- 沒有 CSS 也能做 dark mode——加一行 `color-scheme` meta 就能讓瀏覽器處理預設元素的配色，成本幾乎為零
- `light-dark()` 函式現在瀏覽器支援度夠好了，能讓你在一個 CSS property 裡同時寫兩種模式的值，不用每個 token 都開兩份
- 但是：Level 6 的 `light-dark()` 方案 Safari 要到相對近期的版本才完整支援，老用戶比例高的網站還是得留 fallback

### [How WebRTC Actually Works, All in one post](https://dev.to/mohamedamine_benhima/how-webrtc-actually-works-all-in-one-post-3g7o)

這是少見的把 WebRTC 從頭講到底又不會讓人睡著的文章。核心只有一件事：WebRTC 不會直接丟資料，它會先在兩個 peer 之間找出「最佳 candidate pair」——也就是兩端能用來通訊的最佳 IP+port 組合——然後鎖定。找的過程就是 ICE，而 ICE 會試 host（同網段）、STUN（公網 IP）、TURN（中繼），最後還有 peer-reflexive 這種 runtime 才發現的。以前寫 Google Meet 整合或 AI voice agent 被 NAT 穿透搞過的都知道這個順序有多重要——知道 TURN 是最後手段、不是第一選擇，能讓你的延遲降一半。

**重點：**
- Host candidate 最快（同網段直連），STUN 其次（知道自己公網 IP 後直連），TURN 最慢（走中繼，但穩定）。順序就是 ICE 的優先順序
- STUN 和 TURN 解的是不同問題：STUN 只告訴你「你在外面看起來是誰」，TURN 是「直連不通時幫你轉發」。很多人把兩者混為一談
- 但是：Symmetric NAT（企業網路、部分行動網路）會讓 STUN 探到的 address 失效，只能退回 TURN。AI voice agent 這類 browser-to-server 的場景其實比 p2p 單純，不用 STUN/TURN 也能跑得很順

---

## ⚡ 一句話帶過

- **[SvelteKit 2: Code-based router instead of file-based router now is just a Gist!](https://dev.to/maxcore/sveltekit-2-code-based-router-instead-of-file-based-router-now-is-just-a-gist-28cm)** — 把 SvelteKit 的檔案路由換成程式碼路由，看起來聰明但你真的想手動維護那張 pattern map 嗎？
- **[Symbiote.js: superpowers for Web Components](https://dev.to/foxeyes/symbiotejs-superpowers-for-web-components-1gid)** — 又一個「讓 Web Components 變好用」的輕量函式庫，這種東西每兩年就出一個，等社群投票吧
- **[Your Cypress Tests Are Slower Than You Think](https://dev.to/cydavid/your-cypress-tests-are-slower-than-you-think-8ba)** — 正確。但 Playwright 早就把這問題解決得七七八八了，還沒遷的可以開始考慮
- **[I Analysed 200 PRs in Shadcn-UI To Find Duplicates](https://dev.to/chinmaymhatre/i-analysed-200-prs-in-shadcn-uiui-to-find-duplicates-it-went-surprisingly-well-3jh7)** — 開源 maintainer 現在的地獄：AI 批量生成 PR，內容 80% 重複，人工 review 變不可能
- **[htop for Your Git History](https://dev.to/ticktockbent/htop-for-your-git-history-53oj)** — 看 repo 形狀不看程式碼，先看 commit 頻率與 contributor 分布，適合接手陌生專案的第一天
- **[When Your Mocks Lie: Contract Testing with TWD](https://dev.to/kevinccbsg/when-your-mocks-lie-contract-testing-with-twd-2e58)** — Mock drift 是前端測試的隱形殺手，contract testing 才是解法，不是手動維護 fixture
- **[Testing Payment Flows Without the Payment SDK](https://dev.to/kevinccbsg/testing-payment-flows-without-the-payment-sdk-3obm)** — 金流 SDK 不讓你端到端測試？自己造 fake server 繞過，Stripe 整合一定踩過
- **[Build a Logger and Validator with TypeScript Decorators (Like NestJS)](https://dev.to/malloc72p/build-a-logger-and-validator-with-typescript-decorators-like-nestjs-376n)** — 想知道 NestJS decorator 怎麼運作的、而不是背 API 的人，自己刻一個最快
- **[I Replaced Chrome with Safari for AI Browser Automation](https://dev.to/achiya-automation/i-replaced-chrome-with-safari-for-ai-browser-automation-heres-what-broke-and-what-finally-worked-15ep)** — Playwright MCP 幾乎都綁 Chromium，Safari 的坑文件都沒寫，作者踩完給你看
- **[Why We Switched Back from Claude Opus 4.7 to 4.6](https://dev.to/vibeagentmaking/why-we-switched-back-from-claude-opus-47-to-46-47f9)** — 不是 4.7 變弱，是太聰明停不下來，長任務 agent 被它「完美主義」燒掉預算。這種回饋很值得看
- **[The 12 Security Issues I Keep Finding in Vibe-Coded Apps (Lovable, Bolt, v0)](https://dev.to/systagproject/the-12-security-issues-i-keep-finding-in-vibe-coded-apps-lovable-bolt-v0-786)** — AI 生成的 SaaS 出貨快，但 API key 寫前端、沒 auth、CORS 全開這幾個經典都還在
- **[Your structured outputs are probably less portable than you think](https://dev.to/sravan27/your-structured-outputs-are-probably-less-portable-than-you-think-2p1l)** — 同一份 JSON schema 換個 LLM provider 就炸，這不是抽象洩漏是抽象根本沒建立

---

## 📚 慢慢啃

- **[Why closures finally clicked for me after 2 years](https://dev.to/samareshdas/why-closures-finally-clicked-for-me-after-2-years-3i2g)** — 如果你還在用「closure 就是函式記住變數」這種含糊理解，這篇會讓你從 call stack 和 lexical scope 角度把它重新建立一次
- **[Boring code is an organizational tell](https://dev.to/simme/boring-code-is-an-organizational-tell-4gca)** — 作者的觀察：無聊的程式碼少見不是因為工程師愛炫技，是因為組織在獎勵炫技。讀完你會重新思考 code review 標準
- **[Stop Overengineering: How to Build Scalable Apps Without Killing Your Momentum](https://dev.to/georgegoodluck/stop-overengineering-how-to-build-scalable-apps-without-killing-your-momentum-l15)** — 給入行一兩年開始「架構病」的——可擴展不等於先上微服務，先做對再做大
- **[A Truth Filter for AI Output: An Experiment with Property-Based Testing](https://dev.to/copyleftdev/a-truth-filter-for-ai-output-an-experiment-with-property-based-testing-1j9c)** — 用 property-based testing 來過濾 AI 產生的內容哪些是真的可驗證，哪些是幻覺，實驗性但思路值得借鑒
