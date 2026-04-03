---
title: "Cursor 3 全面改版、Zig 重寫 Git 讓 Bun 快 100 倍、瀏覽器原生影片渲染"
date: 2026-04-03
description: "Cursor 3 推出 Agent-first 全新介面，Zig 重寫 Git 讓 Bun 加速 100 倍，WebCodecs + OffscreenCanvas 實現純前端影片渲染管線，另有 10 則快訊和 4 篇值得週末啃的長文"
tags: [frontend, tooling, web-platform, ai]
---

今天三件大事。Cursor 直接把 IDE 砍掉重練，做了一個「Agent 調度台」出來；有人用 AI agent 群蜂把 Git 用 Zig 重寫了一遍，Bun 跑起來快 100 倍；然後瀏覽器的 WebCodecs 已經強到可以在前端做完整的影片渲染管線了——不花一分錢伺服器。

---

## 🔧 今日硬菜

### [Cursor 3](https://cursor.com/blog/cursor-3)

Cursor 做了一件很大膽的事：他們沒有繼續在 VS Code fork 上面疊功能，而是從零打造了一個全新介面。這不是 UI 微調，是整個產品定位的轉向——從「有 AI 的 IDE」變成「管理 AI agent 的工作台」。新介面支援多 repo 同時操作、本地和雲端 agent 無縫切換、parallel agent 在側邊欄一目了然。雲端 agent 跑完還會自己截圖給你 review，像是交差報告一樣。最狠的是他們出了 Plugin Marketplace，MCP、skill、sub-agent 都可以一鍵裝。

**重點：**
- 多 repo 工作區讓你同時操作前後端專案，不用再開一堆 VS Code 視窗
- 本地跑到一半可以丟上雲端繼續——蓋上筆電回家路上 agent 還在幫你寫 code
- 但是原本的 IDE 功能還在，`Cmd+Shift+P → Agents Window` 就能切換——這算是給保守派留的後路

### [We sped up bun by 100x](https://vers.sh/blog/git-zig-bun-100x)

標題聽起來像 clickbait，但數字是真的。Vers 團隊用他們自己的 AI agent 群蜂架構（他們叫「code cannon」），花了一週、燒了 130 億 token，把 Git 用 Zig 整個重寫了一遍。然後拿這個 Zig 版 Git 直接跟 Bun 的 codebase 做 native 整合，取代原本呼叫 git CLI 的方式。結果？`findCommit` 在 M4 Mac 上快了 85 倍，`cloneBare` 在 Linux 上快了 34 倍。整個 clone + checkout 流程大概是 10-30 倍的提升。他們還順手編譯成了 WASM，比原版小 5 倍。

**重點：**
- 關鍵不只是「重寫」，而是 Zig 可以跟 Bun 做 native 整合，省掉 CLI subprocess 的開銷——這才是 100x 的真正來源
- AI agent 群蜂的工作模式值得研究：每個 agent 有獨立 VM、獨立 prompt、共享一個 Git repo，像微服務一樣分工
- 但這整套的成本是 130 億 token（大約幾千到上萬美金），不是每個團隊玩得起的遊戲

### [How to Render and Export Video in the Browser with WebCodecs, OffscreenCanvas, and a Web Worker](https://dev.to/nareshipme/how-to-render-and-export-video-in-the-browser-with-webcodecs-offscreencanvas-and-a-web-worker-mm3)

做過影片處理的都知道 server-side rendering 有多貴——FFmpeg 吃記憶體、容器超時、compute 帳單噴到不敢看。這篇文章展示了一個完整的純前端影片渲染管線：用 WebCodecs 解碼影片、OffscreenCanvas 做裁切和字幕疊加、Web Worker 確保主執行緒不卡、最後重新編碼成 H.264 MP4 直傳 R2。架構乾淨得像教科書，但每一步都有 production 等級的細節——GPU memory leak 怎麼防、進度條怎麼拆分、bitrate 怎麼依解析度調整。

**重點：**
- `VideoSample` 持有 GPU 記憶體，忘記呼叫 `sample.close()` 幾秒內就會把 Worker 搞掛——這種坑文件不會告訴你
- 目前 Safari 完全不支援 WebCodecs，所以 production 還是需要 server fallback——這是最大的但書
- 在 Next.js 裡用 `new Worker(new URL('./worker.ts', import.meta.url))` 可以直接打包，零設定——這點倒是很順

---

## ⚡ 一句話帶過

- **[Migrating a Webpack-Era Federated Module to Vite Without Breaking the Host Contract](https://dev.to/mdolive/migrating-a-webpack-era-federated-module-to-vite-without-breaking-the-host-contract-31nb)** — 從 Webpack Module Federation 遷到 Vite 的實戰紀錄，最痛的不是工具切換而是 host 端 contract 不能動
- **[building an atomic bomberman clone, part 4: react vs. the game loop](https://dev.to/tomerl1/building-an-atomic-bomberman-clone-part-4-react-vs-the-game-loop-2fg8)** — React 的「state 變 → re-render」跟 game loop 的「每幀都要跑」天生衝突，這篇處理得很漂亮
- **[How I Achieved 100/100 Lighthouse Score with React & TypeScript](https://dev.to/ljresetl/how-i-achieved-100100-lighthouse-score-with-react-typescript-a-performance-deep-dive-19m3)** — React 拿滿分不是不可能，但你需要的不是「clean code」而是一整套效能策略
- **[Zero-configuration Go backend support on Vercel](https://vercel.com/changelog/zero-configuration-go-backend-support)** — Vercel 現在原生支援 Go backend 部署，不用 `/api` 目錄也不用 `vercel.json`
- **[Optimizing Vercel Sandbox snapshots](https://vercel.com/blog/optimizing-vercel-sandbox-snapshots)** — Vercel Sandbox 的 filesystem snapshot 優化實錄，先求穩再求快的工程思維
- **[Building a sidebar with React dnd-kit](https://dev.to/math-krish/building-a-sidebar-with-react-dnd-kit-5hi8)** — dnd-kit 做巢狀拖拉 sidebar 的實作筆記，pin 區域和拖放互不干擾的設計值得參考
- **[How to Finally Kill Every Last 'npm audit'](https://dev.to/tonymet/how-to-finally-and-iteratively-kill-every-last-npm-audit-51ep)** — 跑 `npm audit` 噴 400 個漏洞然後 `npm audit fix` 反而把東西搞壞——這篇教你有系統地一個一個幹掉
- **[I built a JSON viewer because the most popular one betrayed its users](https://dev.to/valentinconan/i-built-a-json-viewer-because-the-most-popular-one-betrayed-its-users-5e6e)** — JSON Formatter 在結帳頁偷塞 popup 被抓包，200 萬用戶的 Chrome 套件信任就這麼碎了
- **[DOM XSS via Unsafe Deserialization in TeleJSON](https://dev.to/cverports/ghsa-ccgf-5rwj-j3hv-ghsa-ccgf-5rwj-j3hv-dom-xss-via-unsafe-deserialization-in-telejson-4f16)** — telejson 6.0 以下的反序列化用了 `new Function()`，如果你的 Storybook 版本比較舊建議查一下
- **[Unit Testing in React Using Vitest — A Practical Guide](https://dev.to/thecoollearner/unit-testing-in-react-using-vitest-a-practical-guide-pbd)** — 從 Jest 過來的人看這篇可以快速上手 Vitest 在 React 元件測試上的差異

---

## 📚 慢慢啃

- **[Stop Chasing Scroll with JavaScript: A Deep Dive into CSS Scroll-Driven Animations](https://dev.to/tu_codigocotidiano_f173d/stop-chasing-scroll-with-javascript-a-deep-dive-into-css-scroll-driven-animations-4alg)** — 用 `scroll-timeline`、`scroll()`、`view()` 做 parallax，不再需要 `IntersectionObserver` 加一堆 JS，瀏覽器原生跑動畫永遠比你快
- **[Functions, Generics, and the Stuff That Looks Familiar But Isn't](https://dev.to/gabrielanhaia/functions-generics-and-the-stuff-that-looks-familiar-but-isnt-2phf)** — 從 Java/PHP 轉 TypeScript 的人一定會被 generics 的「看起來一樣但完全不是那回事」搞到懷疑人生，這篇講得很到位
- **[The Type System: What You Know, What's New, and What's Weird](https://dev.to/gabrielanhaia/the-type-system-what-you-know-whats-new-and-whats-weird-3ojd)** — 上一篇的續集，專攻 structural typing 和 type erasure，Java 腦的人讀完會有一種「原來如此」的解脫感
- **[JavaScript 设计模式（八）：命令模式实现与应用](https://juejin.cn/post/7624043736572395520)** — 撤銷重做、快捷鍵、批次操作——命令模式在前端的實用場景比你想像的多，這篇用前端真實案例講得很清楚
