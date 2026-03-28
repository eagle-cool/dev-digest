---
title: "GitHub 偷練私有 Repo、TS Branded Types、Async 防爆六式"
date: 2026-03-28
description: "GitHub 默默把私有 Repo 納入 AI 訓練、TypeScript 十二行程式碼消滅整類金額 Bug、六個 Async 模式讓你的 Production 不再半夜炸裂"
tags: [frontend, typescript, web-platform, tooling]
---

今天值得聊的就三件事。GitHub 又搞了一波靜悄悄的 opt-in（對，又是你沒同意就被同意了），有人用十二行 TypeScript 在 97 個檔案裡消滅了一整類金額 Bug，然後有篇 Async 模式整理讓我想起當年被 race condition 搞到凌晨三點的往事。

---

## 🔧 今日硬菜

### [If you don't opt out by Apr 24 GitHub will train on your private repos](https://news.ycombinator.com/item?id=47548243)

GitHub 又來了。這次他們默默把所有用戶的私有 Repo 納入 Copilot AI 模型訓練的 opt-in 名單——注意，是「預設開啟」，不是「詢問你要不要」。你需要在 4 月 24 日之前手動到 `github.com/settings/copilot/features` 關掉，否則你的私有程式碼就會被拿去訓練。HN 上 343 個 upvote、161 則留言，大家的反應基本上就是「又來？」

踩過這個坑的人都知道，大公司的「默認同意」策略從來不是意外。上次是 Copilot 的程式碼建議會吐出跟你 private repo 驚人相似的 snippet，這次直接把整個 repo 端走。如果你的 repo 裡有商業邏輯、API key pattern、或任何不想被模型學走的東西，現在就去關。

**重點：**
- 4 月 24 日前到 GitHub Settings → Copilot → Features 手動 opt-out
- 預設開啟，不關就等於同意你的私有 Repo 被拿去訓練
- 但是... GitHub 的說法是「只用於改善 Copilot」，信不信由你

### [How a Branded Cents Type Eliminated an Entire Class of Bugs Across 97 Files](https://dev.to/emmanueln07/how-a-branded-cents-type-eliminated-an-entire-class-of-bugs-across-97-files-2o6o)

這篇是真正的硬菜。作者在做一個 React 19 + TypeScript 的個人財務 PWA（Talliofi），碰到經典問題：`0.1 + 0.2 !== 0.3`。解法不是裝 library，是十二行 TypeScript branded type：

```typescript
export type Cents = number & { readonly __brand: 'Cents' };
export function cents(value: number): Cents {
  if (!Number.isSafeInteger(value)) {
    throw new Error(`Invalid cents value: ${value}`);
  }
  return value as Cents;
}
```

這個 `Cents` 型別在 runtime 完全沒有 overhead——JavaScript 根本不知道它的存在。但 TypeScript compiler 會把它當成跟 `number` 不同的型別，所以你不可能「不小心」把美元金額傳進期望 cents 的函式。所有算術運算都透過 `addMoney()`、`subtractMoney()` 等 wrapper，確保結果永遠經過驗證。928 個金額相關的值、97 個檔案，全部被這個型別守住了。

最痛的部分是遷移——每個 `income - expenses` 都要改成 `subtractMoney(income, expenses)`，compiler 會告訴你哪裡要改，但不會幫你改。作者說得好：**introduce branded types early，遷移成本跟 codebase 大小成正比。**

**重點：**
- Branded type 是零 runtime 成本的型別安全——intersection type `& { readonly __brand: 'Cents' }` 在 JS 層面不存在
- 所有金額都必須通過 `cents()` 驗證閘門，overflow、NaN、Infinity 全部在此攔截
- 但是... 這個 pattern 的代價是每個算術運算都需要 wrapper function，寫起來比較囉嗦，而且遷移老專案是個苦差事

### [6 Async JavaScript Patterns That Prevent Partial Failures in Production](https://dev.to/jsgurujobs/6-async-javascript-patterns-that-prevent-partial-failures-in-production-449d)

大部分 async 程式碼在 happy path 都沒問題，炸的都是 workflow 做到一半掛掉的時候——付款成功但出貨失敗、舊的 fetch 結果覆蓋新的、一次噴 100 個 request 被 rate limit 擋住。

這篇整理了六個實戰模式：compensated steps（失敗就 rollback，像付款成功但出貨失敗時自動退款）、提早啟動獨立 Promise（不要無腦 sequential await）、用 `Promise.allSettled` 取代 `Promise.all`（dashboard 不該因為一個 API 掛掉就整個白屏）、只 retry transient error 加 exponential backoff、用 `AbortController` 取消過期的 request（React `useEffect` 裡的經典 race condition）、還有 concurrency limiter 控制同時發出的請求數量。

沒有哪個是新概念，但作者把 before/after 對照得很清楚。如果你的 async code 只考慮 happy path，那它已經壞了，只是你還不知道。

**重點：**
- Compensated steps 是窮人版的 Saga pattern——每個 side effect 都要有對應的 rollback
- `AbortController` 在 React useEffect 裡是必需品，不是選配
- 但是... 這些 pattern 全部手寫會讓 code 變很醜，考慮用 library（如 p-limit、p-retry）或框架層級的解決方案

---

## ⚡ 一句話帶過

- **[RSAC 2026: Every AI IDE Is Vulnerable](https://dev.to/toniantunovic/rsac-2026-every-ai-ide-is-vulnerable-heres-what-that-actually-means-for-your-workflow-69l)** — RSA 大會實測結果：100% 的 AI coding IDE（包括 Claude Code、Cursor）都擋不住 prompt injection，用之前先想清楚你的 codebase 裡有沒有不該被讀的東西
- **[8 Agents Wrote Perfect Components - And Nothing Worked](https://dev.to/aws/8-agents-wrote-perfect-components-and-nothing-worked-2176)** — AWS 這篇講到痛點了：平行跑的 AI agent 各自寫出完美 component，但 column name、URL path、參數格式全對不上，結論是先抽 shared contract 再讓 agent 動手
- **[终局之战：全链路性能体检与监控](https://juejin.cn/post/7621805099107434536)** — 掘金這篇把前端效能監控從 LCP 到告警到回滾講了一遍，凌晨三點被 pager 叫醒的人看了會有共鳴
- **[How to Fix CORS Errors in Your Frontend App Using a Proxy Server](https://dev.to/0xdezman/how-to-fix-cors-errors-in-your-frontend-app-using-a-proxy-server-5h65)** — 又一篇 CORS 教學，但如果你到現在還在被 CORS 搞，這篇確實寫得比大部分 Stack Overflow 答案清楚
- **[How npm, pnpm, and yarn Ate 40GB of My 256GB SSD](https://dev.to/bradley_nash/how-npm-pnpm-and-yarn-ate-40gb-of-my-256gb-ssd-7lb)** — 256GB SSD 被 node_modules 吃掉 40GB，這不是 bug 這是 feature（不是）
- **[Stop Rewriting Components — Convert React to Vue Automatically](https://dev.to/parsajiravand/stop-rewriting-components-convert-react-to-vue-automatically-pfc)** — React 轉 Vue 的自動轉換工具，適合那些被客戶要求「順便也出個 Vue 版」的苦命人
- **[CSS Specificity Visualizer + Tailwind Class Sorter](https://dev.to/tatelyman/css-specificity-visualizer-tailwind-class-sorter-two-new-tools-4fpl)** — CSS 權重視覺化 + Tailwind class 排序工具，debug specificity 地獄的時候會謝天謝地
- **[I built a React Native OTT video player with debugging tools](https://dev.to/zulkuadsiz/i-built-a-react-native-ott-video-player-with-debugging-tools-open-core-pro-5092)** — React Native 的串流影片播放器，內建字幕、畫質切換和 debug 工具，做過 RN 影片的都知道這有多痛
- **[Design Tokens](https://dev.to/girlwhocode/design-tokens-23cl)** — Design tokens 概念整理，如果你的 design system 還在用寫死的 hex code，是時候看看了
- **[Monitoring Past Performance vs. Alerting Real-Time Issues: What React Teams Are Missing](https://dev.to/nosyos/monitoring-past-performance-vs-alerting-real-time-issues-what-react-teams-are-missing-hdc)** — 你的 React app 有 monitoring 不代表有 alerting，dashboard 看得到歷史但救不了當下

---

## 📚 慢慢啃

- **[案例分析：从"慢"到"快"，一个后台管理页面的优化全记录](https://juejin.cn/post/7621792008844361768)** — 一個電商後台管理頁面的完整優化過程：2000 筆資料的表格、3 個圖表、搜尋卡頓、導出假死，從問題定位到解法一步步拆解，讀完你會想回去重審自己的 admin panel
- **[React vs Vue vs Svelte: Component Architecture Strategies for Visual Page Builders](https://dev.to/jasonbiondo/react-vs-vue-vs-svelte-component-architecture-strategies-for-visual-page-builders-and-marketing-369g)** — 不是又一篇「三大框架比較」，是從 visual page builder 的角度看 component 架構怎麼設計，讓非工程師也能拖拉組裝頁面，值得做 low-code 平台的人細讀
- **[Source code is now a common good, and SaaS is mostly dead](https://dev.to/remojansen/source-code-is-now-a-common-good-and-saas-is-mostly-dead-gke)** — 作者 2023 年就預言 AI 會讓 SaaS 泡沫破裂，現在他說自己錯了——不是時間表錯，是低估了速度，AI 讓個人開發者能用極低成本複製 SaaS 產品，原始碼正在變成公共財
- **[The Webhook Failure Modes Nobody Warns You About](https://dev.to/jamesbrown/the-webhook-failure-modes-nobody-warns-you-about-346m)** — Stripe 的 78 小時 retry 排程、handler 沒收到但 200 OK 已經回了、webhook 看起來簡單到你忘記它會壞的所有方式，做過金流串接的必讀
