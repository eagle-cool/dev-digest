---
title: "React Compiler 終於上 Vite 8、AVIF 完勝 JPEG、瀏覽器跑 LLM 不是夢"
date: 2026-04-15
description: "React Compiler 1.0 搭 Vite 8 的正確安裝姿勢、AVIF 圖片格式在 2026 年的全面攻略、用 WebLLM 在瀏覽器端跑完整 RAG pipeline——今天三道硬菜都跟前端效能有關"
tags: [frontend, react, tooling, web-platform, ai]
---

今天三件事都跟「效能」這兩個字脫不了關係。React Compiler 1.0 終於讓你不用手動 `useMemo` 了但 Vite 8 把 Babel 踢掉搞得一堆教學失效，AVIF 在 2026 年已經沒有理由不用了，然後有人真的把 LLM 搬進瀏覽器跑 RAG——不是 demo，是真的能用的那種。

---

## 🔧 今日硬菜

### [React Compiler 1.0 + Vite 8: The Right Way to Install After @vitejs/plugin-react v6 Drops Babel](https://dev.to/recca0120/react-compiler-10-vite-8-the-right-way-to-install-after-vitejsplugin-react-v6-drops-babel-p0i)

React Compiler 在 2025 年 10 月正式 GA，它做的事情很單純：在 build time 自動幫你加上 `useMemo` / `useCallback` / `React.memo`，粒度比你手寫的還細（per-expression 而不是 per-component）。Meta 在 Instagram 和 Quest Store 跑了一年多，render time 降了約 12%，re-render 次數少了 2.5 倍。

問題是 Vite 8 + `@vitejs/plugin-react` v6 把內建 Babel 換成了 oxc（Rust 寫的，快很多），所以網路上一堆教你用 `react({ babel: { plugins: [...] } })` 的教學全部失效。現在正確做法是裝 `@rolldown/plugin-babel`，然後用 `reactCompilerPreset()` 這個從 plugin-react 匯出的 preset。順序很重要：`babel()` 要放在 `react()` 前面，不然 compiler 根本不會跑。

**重點：**
- Vite 8 用 `@rolldown/plugin-babel` + `reactCompilerPreset()`，Vite 7 以下用舊的 `react({ babel: {...} })`
- 先裝 ESLint plugin 跑 CI，再開 compiler——compiler 遇到違反 Rules of React 的 component 會靜默跳過，你根本不知道
- 但是... 別急著大量刪除手寫的 `useMemo` / `useCallback`，compiler 是 additive 的，先 profile 再動手

### [AVIF in 2026: The Complete Guide to the Image Format That Beat JPEG, PNG, and WebP](https://dev.to/aralroca/avif-in-2026-the-complete-guide-to-the-image-format-that-beat-jpeg-png-and-webp-34n2)

2026 年了，如果你的網站還在直接 serve JPEG/PNG，那你就是在用 2G 手機的速度折磨你的用戶。AVIF 現在瀏覽器覆蓋率 93%，比同品質 JPEG 小 30-50%，比 WebP 再小 15-25%。HTTP Archive 的數據說圖片佔網頁總重量約 50%，換算下來光是切格式就能讓 LCP 明顯下降。

技術上 AVIF 用的是 AV1 video codec 的 intra-frame coding，支援 10/12-bit 色深、HDR、alpha 透明度、film grain synthesis。Alliance for Open Media（Google、Apple、Microsoft、Mozilla、Netflix 都在裡面）開發的，royalty-free。實作面 Next.js 的 `next/image` 直接支援 `formats: ['image/avif', 'image/webp']`，Cloudflare 和 Netlify 都有自動轉換，Node.js 用 Sharp 就搞定。

**重點：**
- 用 `<picture>` + `<source>` 做 progressive format delivery，AVIF → WebP → JPEG 三層 fallback
- `next/image` 在 `next.config.js` 加一行就支援，Cloudflare Polish 自動轉換
- 但是... 編碼速度比 JPEG 慢 5-10 倍，適合 build-time 批次處理而不是 real-time；最大解析度限制 8192x4320，超過要 tiling

### [I Built a Browser-Local AI Assistant in Next.js with WebLLM, WASM, ONNX Runtime, Web Workers, and RAG](https://dev.to/databro/i-built-a-browser-local-ai-assistant-in-nextjs-with-webllm-wasm-onnx-runtime-web-workers-and-58b5)

大部分所謂的「AI 聊天 widget」就是包了一層皮的 API call。這位老兄不一樣——他把完整的 RAG pipeline 搬進瀏覽器跑：WebLLM 負責 generation、ONNX Runtime Web 負責 embedding 跟 reranking、Web Worker 當 orchestration layer、WASM 提供接近 native 的運算效能。整個推理不出瀏覽器。

架構設計最漂亮的地方是職責分離：UI thread 只管渲染跟互動，Web Worker 是真正的大腦，負責 model loading、cache 管理、KB 向量載入、hybrid retrieval、reranking、confidence gating、prompt assembly、最後才丟給 WebLLM 生成。第一次要下載 model artifacts，但之後 browser cache 會讓啟動快很多。

**重點：**
- WebLLM 是 runtime 不是 model——它載入 Llama/Phi/Gemma 等模型執行推理
- Generation 跟 retrieval-side inference 用不同引擎（WebLLM vs ONNX Runtime Web），這是正確的架構決策
- 但是... 第一次冷啟動要下載幾百 MB 的 model artifacts，UX 上需要好好處理 loading state；而且瀏覽器端的推理速度還是跟 server-side 差一大截

---

## ⚡ 一句話帶過

- **[Elastic Build Machines is now GA](https://vercel.com/changelog/elastic-build-machines-is-now-ga)** — Vercel 現在會自動幫每個 project 配對的 build machine，Pro 跟 Enterprise 預設開啟，不用再手動挑規格了
- **[What's Actually Wrong with Web Components?](https://dev.to/foxeyes/whats-actually-wrong-with-web-components-3pjk)** — 答案是：基本上沒什麼問題。瀏覽器原生 API，該用就用，別被 Twitter 上的口水戰嚇到
- **[Contributing to React Router: Implementing Callsite Revalidation Opt-out](https://dev.to/pavkode/contributing-to-react-router-implementing-callsite-revalidation-opt-out-without-prior-experience-1k8f)** — 沒有大型開源經驗也能貢獻 React Router，重點是讀懂現有程式碼再動手
- **[How to Build a Lightweight Embeddable Widget in Vanilla JS (Under 30KB)](https://dev.to/alex_boykov/how-to-build-a-lightweight-embeddable-widget-in-vanilla-js-under-30kb-5f4b)** — 拜託不要為了一個 chat bubble 就塞 150KB 的 React 進別人的網站
- **[I got tired of class-heavy UI code, so I started building Juice](https://dev.to/stinklewinks/i-got-tired-of-class-heavy-ui-code-so-i-started-building-juice-4ocg)** — 用 attribute-based 取代 class-heavy 的 UI 系統，還在 alpha，概念有趣但先觀望
- **[Why I Ditched GA4 for a Custom Next.js + Supabase Analytics Stack](https://dev.to/zenovay/why-i-ditched-ga4-for-a-custom-nextjs-supabase-analytics-stack-5foc)** — GA4 對開發者來說就像開 747 去買菜，自己用 Next.js + Supabase 搭一個輕量版有時候更香
- **[Why AI React Component Generators Break in Real Projects](https://dev.to/frontuna_system_0e112840e/why-ai-react-component-generators-break-in-real-projects-and-how-to-fix-it-4062)** — AI 生的 component 在 demo 裡很美，放進真實 codebase 就炸——問題不在 AI 品質，在 context
- **[Building a Resilience Layer for Mobile AI in React Native](https://dev.to/nikapkh/building-a-resilience-layer-for-mobile-ai-how-i-handle-429s-provider-fragmentation-and-streaming-4e94)** — react-native-ai-hooks v0.6.0，處理 rate limit 跟 provider fallback 的 hooks library
- **[The cost of building a workflow editor on React Flow](https://www.workflowbuilder.io/blog/build-vs-buy-workflow-editor-hidden-cost-react-flow)** — React Flow 拿來做 workflow editor 看似簡單，但隱藏成本比你想的多
- **[Anomaly alerts are now generally available](https://vercel.com/changelog/anomaly-alerts-ga)** — Vercel Observability Plus 用戶現在能收到異常流量告警，終於不用自己盯 dashboard 了

---

## 📚 慢慢啃

- **[I vibe-coded the same app on Supabase, Convex, Vennbase, and InstantDB](https://dev.to/alexdavies74/i-vibe-coded-the-same-app-on-supabase-convex-vennbase-and-instantdb-the-results-look-the-same-1nhg)** — 用 AI coding agent 在四個不同 backend 上 vibe code 同一個 app，結果看起來一樣但底層差很多。想知道哪個 backend 最適合 AI 輔助開發的人值得花時間讀
- **[git worktree: Multiple Working Directories Per Repo, and the Key to Parallel AI Agents](https://dev.to/recca0120/git-worktree-multiple-working-directories-per-repo-and-the-key-to-parallel-ai-agents-40)** — `git worktree` 讓你不用 stash 就能同時在多個 branch 工作，配合 AI agent 平行開發更是如虎添翼。比 `git stash` 優雅一百倍
- **[Cloudflare Tunnel in 2026: Expose localhost Without Opening Ports](https://dev.to/recca0120/cloudflare-tunnel-in-2026-expose-localhost-without-opening-ports-or-buying-an-ip-32l5)** — 2026 年了還在開 port forwarding？Cloudflare Tunnel 免費、免靜態 IP、自帶 HTTPS，讓你的 side project 五分鐘上線
