---
title: "Windows Start 沒用 React、Next.js 查五次 DB、Vue 3 效能陷阱"
date: 2026-03-24
description: "Windows Start Menu 沒用 React 只是用了 React Native for Windows、Next.js 元件架構導致重複查詢、Vue 3 Proxy reactivity 在大數據集直接跪了，加上 Vercel 併購與 npm 生態荒謬日常"
tags: [frontend, react, nextjs, web-platform]
---

今天值得聊的就三件事。有人（又）以為 Windows 開始選單用了 React 然後崩潰了——拜託先搞清楚 React 和 React Native 的差別好嗎。Next.js 的元件架構讓你的 DB 被同一個 query 打了五次而你渾然不知，然後 Vue 3 的 Proxy reactivity 在大數據集面前直接跪了。另外 Vercel 又買東西了，npm 生態系繼續荒謬，Claude Code 的兩篇文章在 HN 上都破百讚。

---

## 🔧 今日硬菜

### [No, Windows Start does not use React](https://pathar.tl/blog/no-windows-start-does-not-use-react/)

Windows Central 報導微軟要把 Start Menu 從 React 換成 WinUI，然後整個網路炸鍋了。問題是——Start Menu 根本沒有用 React。它用的是 React Native for Windows，而且只有底部那個「Recommended」區塊。React Native for Windows 是直接呼叫 Windows API 編譯成原生程式碼的東西，跟你想像中那個跑 JavaScript 的 React 完全是兩回事。

這篇文章的作者 Pat Hartl 直接打臉了這個流言。重點不是 React 好不好，而是媒體和社群有多容易把相似名字的技術搞混。React、React Native、React Native for Windows——三個完全不同的東西，但在新聞標題裡全都變成了「React」。踩過命名陷阱的都知道，naming is hard，但至少寫新聞的人能 Google 一下嗎？

**重點：**
- Start Menu 用的是 React Native for Windows，不是 React，那個區塊還可以關掉
- React Native for Windows 編譯成原生程式碼，直接呼叫 WinUI 3 API，沒有 WebView
- 但是...Windows 的 UI 框架數量確實太多了（WinUI、WPF、UWP、RNW...），這本身才是真正的問題

### [Your Next.js App Makes the Same Database Query 5 Times Per Page Load](https://dev.to/aditya_kushwah/your-nextjs-app-makes-the-same-database-query-5-times-per-page-load-3j24)

這是一個幾乎所有 Next.js 開發者都踩過但不知道自己踩過的坑。五個元件各自獨立 fetch `/api/user`，每個 fetch 都打一次 Prisma query，結果同一筆 user 資料被查了五次。然後 React Strict Mode 在 dev 環境還會把它變成十次，讓你以為是 bug 但其實只有一半是 bug。

元件化架構的精神就是每個元件自己管自己的 data——這本身是好設計。但當五個元件都需要同一筆資料時，「自給自足」就變成了「重複浪費」。更要命的是 code review 根本看不到這個問題，因為每個元件的程式碼單獨看都是對的，重複只發生在 runtime 當所有元件同時渲染的時候。

**重點：**
- 用 React Query / SWR 的 query key 做 client-side 去重，或用 React 的 `cache()` 在 server render 層去重
- 把共用資料提升到 `layout.tsx`，透過 Context 分享給子元件
- 但是...文章最後在推銷自家產品 Brakit，技術分析到一半突然變成廣告，這個套路大家都熟

### [The Vue 3 Reactivity Trap: Why Large Datasets Crash Your Browser](https://dev.to/ameer-pk/the-vue-3-reactivity-trap-why-large-datasets-crash-your-browser-1ikb)

Vue 3 的 Proxy-based reactivity 在 95% 的場景下優雅得不像話，但一旦你塞進去五萬筆資料，它就開始遞迴遍歷每個物件的每個屬性，幫你建了五十萬個 Proxy。5MB 的 JSON 瞬間膨脹成 50MB 的記憶體佔用，主執行緒直接卡死。

解法其實很直覺：`shallowRef` 只追蹤 `.value` 的賦值，不做 deep proxy；`markRaw` 讓 Vue 完全忽略某個物件。再搭配 `useVirtualList` 做 DOM 虛擬化，五萬筆資料渲染起來跟五十筆一樣順。從 800ms 降到 5ms，這不是微優化，這是救命。

**重點：**
- 大數據集改用 `shallowRef`，不需要 deep mutation tracking 時別讓 Vue 幫你做
- 用 `markRaw` 在 `reactive` 物件中標記不需要響應式的屬性
- 但是...你需要清楚知道哪些資料「不需要」深度響應式，判斷錯了就是靜默的 bug，不會報錯

---

## ⚡ 一句話帶過

- **[Vercel acquires new.website](https://vercel.com/blog/vercel-acquires-new-website)** — Vercel 又買了一家 AI 網站建構工具，併入 v0 團隊，版圖越來越大，錢燒得也越來越快
- **[oxlint-tailwindcss: the linting plugin Tailwind v4 needed](https://dev.to/sergioazoc/oxlint-tailwindcss-the-linting-plugin-tailwind-v4-needed-fm2)** — oxlint 開放 native plugin alpha，馬上有人做了 Tailwind v4 的 linting 插件，又多一個理由考慮換掉 ESLint
- **[A Single Regex Got Its Own npm Package. It Gets 70 Million Downloads a Week.](https://dev.to/adioof/a-single-regex-got-its-own-npm-package-it-gets-70-million-downloads-a-week-4h59)** — 一行 `^#!(.*)` 正則表達式，獨立成一個 npm 套件，週下載七千萬次，這就是 JavaScript 生態系
- **[Git hooks are your best defense against AI-generated mess](https://dev.to/jonesrussell/git-hooks-are-your-best-defense-against-ai-generated-mess-5h1a)** — AI 寫的 code 越來越多，pre-commit hook 從「可有可無」變成「非裝不可」的守門員
- **[How to Add Screenshot Tests to Your GitHub Actions CI Pipeline](https://dev.to/custodiaadmin/how-to-add-screenshot-tests-to-your-github-actions-ci-pipeline-3a6f)** — 視覺回歸測試不用自己搞 headless browser，直接在 CI 截圖比對抓 UI 壞掉
- **[Claude Code Cheat Sheet](https://cc.storyfox.cz)** — HN 106 讚，Claude Code 的快捷鍵和 workflow 整理，寫得清楚又實用
- **[How I'm Productive with Claude Code](https://neilkakkar.com/productive-with-claude-code.html)** — HN 105 讚，一個開發者分享怎麼把 Claude Code 融入日常工作流，不是教你 prompt 而是教你思維模式
- **[Node.js Deployment in 2026: Railway vs DigitalOcean](https://dev.to/axiom_agent_1dc642fa83651/nodejs-deployment-in-2026-railway-vs-digitalocean-1m2j)** — Railway 一鍵部署很爽但貴，DigitalOcean 便宜但要自己搞 Nginx，各取所需
- **[How to Generate Open Graph Images Automatically](https://dev.to/custodiaadmin/how-to-generate-open-graph-images-automatically-no-design-tools-required-6pk)** — 動態產生 OG image 的方案整理，別再手動做圖了
- **[Stop Wrestling with D3.js: 8 Free Tools That Do It Better](https://dev.to/elysiatools/stop-wrestling-with-d3js-8-free-tools-that-do-it-better-2ff1)** — 別再跟 D3.js 搏鬥了，八個替代工具讓你幾分鐘出圖，D3 留給需要完全客製化的場景

---

## 📚 慢慢啃

- **[Debugging Memory Leaks in Node.js: A Complete Guide](https://dev.to/benriemer/debugging-memory-leaks-in-nodejs-a-complete-guide-p1g)** — 從 heap snapshot 到 event listener 清理的完整排查流程，下次 Node.js 吃掉 4GB 記憶體時你會感謝自己讀過這篇
- **[HTML Parsing Algorithm and Memory Structure](https://dev.to/jocerfranquiz/html-parsing-algorithm-and-memory-structure-3e3j)** — 瀏覽器怎麼把 HTML bytes 變成 DOM tree？這個系列從底層講起，適合想搞懂渲染管線第一步的人
- **[I Built a Webhook Relay on Cloudflare Workers — Here's Every Bug That Almost Killed It](https://dev.to/eventdock/i-built-a-webhook-relay-on-cloudflare-workers-here-is-every-bug-that-almost-killed-it-22dc)** — 在 Cloudflare Workers 上做 webhook relay 的完整踩坑記，KV 限制、cold start、併發處理一個都沒少
- **[JavaScript Gems: 7 Useful Functions You Should Try To Use](https://dev.to/mari_kitsay/javascript-gems-useful-functions-you-probably-arent-using-3jl3)** — `structuredClone`、`Array.at()`、`Object.groupBy()` 這些你可能還沒習慣用的 JS 內建方法，值得花十分鐘看完
