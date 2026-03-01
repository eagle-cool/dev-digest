---
title: "流式 Markdown 防閃爍、型別安全 EventTarget、JS 碎片化危機"
date: 2026-03-01
description: "AI 串流 Markdown 渲染怎麼不閃？原生 EventTarget 加 TypeScript 泛型有多香？JS 工具鏈碎片化到底多嚴重？三道硬菜加十則快訊帶你看完。"
tags: [frontend, typescript, web-platform, tooling]
---

今天三篇硬菜剛好串成一條線：先是有人認真解決了 AI 串流輸出 Markdown 閃爍的老問題，然後有人用原生 Web API 做了零依賴的型別安全事件系統，最後有人站出來說 JS 工具鏈碎片化已經到了該正視的程度。三篇都不是在追新，而是在解決真問題。

---

## 🔧 今日硬菜

### [Markdown 预解析：别等全文完了再渲染，如何流式增量渲染代码块和公式？](https://juejin.cn/post/7611549704817098778)

做過 AI 聊天介面的都知道那個痛：每來一個 token 就 `marked(fullText)` 重跑一次，結果代碼塊在「純文字」和「高亮態」之間反覆橫跳，公式還沒打完就噴紅叉。這篇提出了狀態機驅動的增量預解析方案——把解析器拆成 TEXT、CODE、MATH 三個狀態，在閉合標籤到達之前先「假裝」它已經閉合了，用佔位節點做增量更新。從 O(N²) 降到 O(1)，不是在吹牛。

**重點：**
- 三狀態機（TEXT/CODE/MATH）切換，碰到 ` ``` ` 或 `$$` 就轉態，不用等閉合
- 代碼塊用 Prism.js 而非 Shiki，搭配 requestAnimationFrame 每 50ms 批量高亮一次
- 但是... Auto-Close 機制是必要的——AI 斷線時 ` ``` ` 永遠不來，你的狀態機會卡死在 CODE 模式

### [Type-Safe CustomEvents: Better Messaging with Native APIs](https://dev.to/link2twenty/type-safe-customevents-better-messaging-with-native-apis-2dol)

又裝 EventEmitter 套件了？等等，瀏覽器原生的 `EventTarget` 其實就夠了。這篇用 TypeScript 泛型把 `CustomEvent` 的 `detail` 型別鎖死，做出了一個零執行時開銷的型別安全事件匯流排。重點不只是少裝一個依賴，而是用 `Record<string, unknown>` 的泛型映射讓 `addEventListener` 自動推導 payload 型別——dispatch 時傳錯型別直接編譯報錯，不用等到 runtime 才炸。

**重點：**
- 用 `TypedEventTarget<M>` 泛型類別包裝原生 `EventTarget`，事件名稱和 payload 一對一綁定
- 購物車範例展示了 `item-added`、`item-removed`、`cart-cleared` 三種事件的完整型別推導
- 但是... 這是 class-based 設計，跟 React/Vue 的響應式系統整合時得自己橋接——作者也承認這塊還沒展開講

### [JavaScript's Fragmentation Crisis: Innovation vs. Interoperability](https://dev.to/pratikmathur279/javascripts-fragmentation-crisis-innovation-vs-interoperability-4ak9)

有人終於把大家心裡的話說出來了。Oxfmt 比 Prettier 快 30 倍？讚。Biome 整合 lint + format？讚。TypeScript 6.0 要 breaking change？好吧。Node.js 25.7.0、Deno 2.7、Bun——每個都要你關注、設定、可能重構。問題是：我們花在「跟上工具鏈」的時間，已經快比寫功能的時間多了。這篇點出了 Rust-based 工具帶來的認知負擔、安全研究門檻被無意間提高、以及大公司合作可能形成的圍牆花園。

**重點：**
- Rust-based 工具（Oxfmt、Biome）快是快，但 JS 開發者現在還得理解 Rust 工具鏈，認知負擔是真的
- Node.js 的 HackerOne signal requirement 本意是過濾低品質漏洞報告，但也擋住了新手安全研究者
- 但是... 文章提出問題比解決方案多，「社群需要在創新和互通之間取得平衡」說了等於沒說——不過至少有人開了這個頭

---

## ⚡ 一句話帶過

- **[打字机效果优化：用 requestAnimationFrame 缓冲高频文字更新](https://juejin.cn/post/7611715859572801582)** — 跟上面的 Markdown 渲染是姊妹篇，用 rAF 把每秒幾十次的 DOM 更新壓到每幀一次，AI 聊天介面的效能救星
- **[How to Generate Images Using LLM Gateway and the Vercel AI SDK](https://dev.to/smakosh/how-to-generate-images-using-llm-gateway-and-the-vercel-ai-sdk-4e69)** — 用一個 OpenAI-compatible API 統一多家圖片生成服務，Vercel AI SDK 的整合範例值得參考
- **[How I Built 1,182 Pages of Free Time Tools with Next.js 16](https://dev.to/cyrilye/how-i-built-1182-pages-of-free-time-tools-with-nextjs-16-1b75)** — Next.js 16 靜態生成 1182 頁做 programmatic SEO，數字很漂亮但 SEO 效果才是重點
- **[RGGrid — a workflow-ready React data grid](https://dev.to/damodarraju/rggrid-a-workflow-ready-react-data-grid-with-rules-audit-logs-workflow-states-k2m)** — React 資料表格加上 audit trail、workflow states、rule engine，內部工具開發者可以看看
- **[I Built a Voice-to-Code VS Code Extension That Runs Entirely On-Device](https://dev.to/agentic_engineer/i-built-a-voice-to-code-vs-code-extension-that-runs-entirely-on-device-16gc)** — 用語音下指令寫程式碼，全部在本機跑不上雲，概念有趣但實用性待驗證
- **[HookLab — Watch your Claude Code hooks in real time](https://dev.to/felipeelias/hooklab-watch-your-claude-code-hooks-in-real-time-42n3)** — Claude Code 的 HTTP hooks 即時監控面板，能看到每個工具呼叫的參數和回傳值
- **[waves/cn — our own shadcn package](https://dev.to/mouad_sadik_ab26b70d42c84/wavescn-our-own-shadcn-package-4fo9)** — 找不到 shadcn/ui 的波浪元件就自己做一個，開源精神讚但生態碎片化又 +1
- **[Your README Is Already a Website](https://dev.to/davorg/your-readme-is-already-a-website-dg7)** — 一個 GitHub Action 把 README.md 直接轉成 styled index.html，不用 Jekyll 不用 Ruby
- **[Claude Code Remote Control: 3 Methods Compared](https://dev.to/_46ea277e677b888e0cd13/claude-code-remote-control-3-methods-compared-moshi-vs-rc-vs-openclaw-2iih)** — Moshi + tmux + Tailscale vs Remote Control vs OpenClaw，結論是 Moshi 方案最穩
- **[How I Built a Production-Ready Reviews System](https://dev.to/freerave/how-i-built-a-production-ready-reviews-system-mongodb-aggregation-custom-toasts-css-variables-2a84)** — MongoDB aggregation pipeline + CSS Variables 做評價系統，架構設計比較紮實

---

## 📚 慢慢啃

- **[The Windows 95 user interface: A case study in usability engineering (1996)](https://dl.acm.org/doi/fullHtml/10.1145/238386.238611)** — 30 年前的 UX 研究論文，讀完你會發現現代前端做的很多 UX 決策，微軟在 1996 年就用可用性測試驗證過了
- **[Verified Spec-Driven Development (VSDD)](https://gist.github.com/dollspace-gay/d8d3bc3ecf4188df049d7a4726bb2a00)** — HN 上 140 分的開發方法論，強調先寫可驗證的規格再動手寫程式碼，AI 輔助開發時代特別值得思考
- **[I'm Building a Programming Language From Scratch](https://dev.to/ericdacoder/im-building-a-programming-language-from-scratch-heres-what-thats-actually-like-4aji)** — 一個人五週寫了 36 萬行 Rust、17 個 compiler crate，包含完整的 Hindley-Milner 型別推導——不一定實用但絕對硬核
