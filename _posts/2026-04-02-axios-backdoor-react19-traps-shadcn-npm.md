---
title: "axios 被下毒、React 19 讓 AI 寫出 bug、shadcn/ui 該不該繼續用"
date: 2026-04-02
description: "axios 1.14.1 供應鏈攻擊透過 plain-crypto-js 後門竊取資料，React 19 四個常見陷阱讓 AI 產出有毒程式碼，shadcn/ui copy-paste 模式與 npm 套件的取捨分析"
tags: [frontend, react, nextjs, typescript, nodejs, tooling, ai]
---

今天最重要的事：如果你昨天跑過 `npm install`，現在立刻去查你的 axios 版本。不是開玩笑，1.14.1 和 0.30.4 被植入後門了。另外，你的 AI 助手還在寫 React 18 的 code，而你可能渾然不覺。

---

## 🔧 今日硬菜

### [Compromised npm Maintainer Account Publishes Malicious Axios Versions with Backdoor via `plain-crypto-js` Dependency](https://dev.to/pavkode/compromised-npm-maintainer-account-publishes-malicious-axios-versions-with-backdoor-via-2l7o)

這次不是什麼假警報。axios 的 npm maintainer 帳號被攻破了，1.14.1 和 0.30.4 兩個版本繞過 GitHub Actions 直接發佈到 npm，偷偷塞了一個叫 `plain-crypto-js` 的依賴。這個東西在 runtime 攔截 HTTP request，把你的 API key、token、header 全部偷走送到攻擊者的伺服器。最恐怖的是 transitive dependency——你的專案可能根本沒直接用 axios，但某個套件間接依賴了它，你就中招了。

**重點：**
- 立刻執行 `npm install axios@1.14.0 --save-exact` 鎖定安全版本
- 光鎖版本不夠，`plain-crypto-js` 可能透過其他套件的依賴樹混進來，需要掃描整個 dependency tree
- 根本問題是 npm maintainer 帳號沒有強制 MFA——一組密碼被偷，整個生態系跟著遭殃

### [React 19 migration traps your AI keeps falling into (and how to fix them with .mdc rules)](https://dev.to/vibestackdev/react-19-migration-traps-your-ai-keeps-falling-into-and-how-to-fix-them-with-mdc-rules-2aoe)

每個 AI coding assistant 的訓練資料都是 React 18 的天下。結果就是：它產出的 code 編譯過了、TypeScript 不報錯，但跑起來炸給你看。作者整理了四個最常踩的坑：`useFormStatus` 放錯位置永遠拿到 `pending: false`、AI 還在 import 已經被 deprecated 的 `useFormState`、多餘的 `forwardRef` 包裝、以及 Next.js 15 的 `params` 變成 Promise 導致 production crash。每個坑都附了 `.mdc` rule 檔案，讓 AI 在開檔案時自動注入正確 pattern。

**重點：**
- `useFormStatus` 必須放在 `<form>` 的子元件裡，不能跟 form 同一層——這個 AI 幾乎 100% 會寫錯
- React 19 的 `useActionState` 取代了 `useFormState`，從 `react-dom` 搬到 `react`
- Next.js 15 的 `params` 是 Promise，同步解構在 dev 可能沒事但 production 直接 crash——這種 bug 最陰險

### [Copy-Paste Components vs npm Packages: shadcn/ui vs Ninna UI in 2026](https://dev.to/chnkc41/copy-paste-components-vs-npm-packages-shadcnui-vs-ninna-ui-in-2026-1n0a)

shadcn/ui 開創了「別裝套件，把 code 複製到你的 codebase」這個流派，兩年過去，該認真檢視這個模式的代價了。文章拿 shadcn/ui 跟走傳統 npm 路線的 Ninna UI 做了七個維度的比較。結論很明確：solo developer 或小團隊想要完全掌控可以選 shadcn/ui，但多專案團隊、需要跨 repo 一致性的組織，copy-paste 模式的維護成本會隨時間指數增長。每個專案的 Button 都長不一樣、Radix 更新要手動同步、accessibility patch 吃不到——這些都是 copy-paste 的隱性帳單。

**重點：**
- shadcn/ui 典型專案會累積 10-15 個 Radix 套件需要自己管版本相容性
- npm 套件模式讓 accessibility 修復自動到位，copy-paste 得手動追上游
- 但兩者的 bundle size 其實差不多，tree-shaking 都做得不錯——這不是選擇的關鍵因素

---

## ⚡ 一句話帶過

- **[How to Use Shaders in React (2026 WebGPU / WebGL Tutorial)](https://dev.to/shaders/how-to-use-shaders-in-react-2026-webgpu-webgl-tutorial-3h25)** — 當每個網站都長一樣的時候，會寫 shader 的前端就是稀缺資源
- **[AI Writes Better UI Without React Than With It](https://dev.to/endenwer/ai-writes-better-ui-without-react-than-with-it-26fl)** — 用 Web Components 讓 AI 生 UI，不用 React 反而更順，這觀點值得想想
- **[I Spent $11,922 on Cursor in Under 4 Weeks](https://dev.to/bytedesk/i-spent-11922-on-cursor-in-under-4-weeks-heres-how-i-fixed-it-3gh2)** — 一個人、四週、六個專案平行跑 AI agent，帳單直接噴到三十幾萬台幣
- **[GLM 5V Turbo on AI Gateway](https://vercel.com/changelog/glm-5v-turbo-on-ai-gateway)** — Vercel AI Gateway 上了 Z.ai 的多模態 coding model，截圖直接轉 code
- **[Building a Data-Driven Chart UI with Zero JavaScript (Pure CSS + FSCSS)](https://dev.to/fscss/building-a-data-driven-chart-ui-with-zero-javascript-pure-css-fscss-gb9)** — 純 CSS 畫圖表，不載 Chart.js 的浪漫主義
- **[Arbitrary JavaScript Execution via eval() in chrome-local-mcp](https://dev.to/crafted_cyber_solutions/arbitrary-javascript-execution-via-eval-in-chrome-local-mcp-1nbl)** — MCP server 用 eval() 跑任意 JS，CVSS 9.1，給 AI 瀏覽器控制權之前先想清楚
- **[Stop Writing Zod Schemas by Hand: What I Learned After 40 API Endpoints](https://dev.to/dileep1415/stop-writing-zod-schemas-by-hand-what-i-learned-after-40-api-endpoints-5ape)** — 手寫 Zod schema 寫到懷疑人生，後端改個欄位名 debug 三小時的那種痛
- **[Vue vs React in 2026: What AI-First Development Teams Actually Choose](https://dev.to/krunal_groovy/vue-vs-react-in-2026-what-ai-first-development-teams-actually-choose-419b)** — 2026 年了還在戰框架，不過加了 AI 輔助開發的維度倒是有點新意
- **[The 22,000 Token Tax: Why I Killed My MCP Server](https://dev.to/codewithagents_de/the-22000-token-tax-why-i-killed-my-mcp-server-2c12)** — MCP 工具描述吃掉 22K token，省錢不如先砍 context 的肥肉
- **[Apple's April SDK Deadline Is Here. Your App Might Get Rejected.](https://dev.to/alanwest/apples-april-sdk-deadline-is-here-your-app-might-get-rejected-5di7)** — 每年都有人被 Apple 最低 SDK 版本要求 reject，每年都有人驚訝

---

## 📚 慢慢啃

- **[I studied Claude Code's leaked source and built a terminal UI toolkit from it](https://dev.to/minnzen/i-studied-claude-codes-leaked-source-and-built-a-terminal-ui-toolkit-from-it-4poh)** — Claude Code 的 terminal UI 層其實是一個自製 React reconciler，這篇拆解了它的渲染引擎架構，想搞 terminal app 的值得細讀
- **[The axios Attack Was a Wake-Up Call. Your AI Agent Just Ran npm install Without Asking You.](https://dev.to/mkdelta221/the-axios-attack-was-a-wake-up-call-your-ai-agent-just-ran-npm-install-without-asking-you-5k2)** — 從 axios 事件延伸討論 AI agent 自動跑 npm install 的安全隱患，沒人 review lockfile 的世界有多危險
- **[Claude Code's Source Didn't Leak. It Was Already Public for Years.](https://dev.to/nikitaeverywhere/claude-codes-source-didnt-leak-it-was-already-public-for-years-34le)** — JS 混淆工具開發者的視角：minified code 本來就不算保護，AI 時代更是如此
- **[Building Confidence with Testcontainers](https://dev.to/jolodev/building-confidence-with-testcontainers-3dfe)** — Node.js 整合測試用 Testcontainers 跑真實資料庫，不再被 mock 騙過關
