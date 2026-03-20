---
title: "Zod v4 藏了一個 JIT 引擎、Vite+ 一統前端工具鏈、AI 時代怎麼寫好架構"
date: 2026-03-20
description: "Zod v4 用 new Function() 偷偷做了 JIT 編譯、Evan You 的 Vite+ 要用一個指令取代你整個工具鏈、以及一篇讓你重新思考程式碼結構的好文。"
tags: [frontend, typescript, tooling, ai]
---

今天值得你停下手邊工作看的就三件事。Zod v4 裡面藏了一個用 `new Function()` 做的 JIT 編譯器，大多數人根本不知道它存在；Evan You 的 VoidZero 推出了 Vite+，想用一個 CLI 取代你的 linter、formatter、test runner 跟 bundler；然後有人寫了一篇關於程式碼架構的文章，講得比大部分教科書都清楚。

---

## 🔧 今日硬菜

### [Why Is Zod v4 Fast — and Where Is Its Ceiling?](https://dev.to/wakita181009/why-is-zod-v4-fast-and-where-is-its-ceiling-280l)

這篇是我最近看到對 Zod v4 內部機制分析得最透徹的文章。作者直接翻了 source code，發現 Zod v4 裡面有一個叫 `$ZodObjectJIT` 的 class——它在第一次 `.parse()` 的時候，用 `new Function()` 動態生成專門針對你 schema shape 的驗證函式，之後每次 parse 都直接跑編譯過的版本。這不是什麼行銷話術，是實打實的 runtime code generation。

作者不只分析了 JIT 的原理，還做了完整的 benchmark。Object schema 上 v4 比 v3 快 1.3-4.5x，遞迴結構更是快到 4.5x。但故事沒這麼美好——帶 checks 的 primitive 型別（像 `.min().max()`）反而比 v3 慢了 27%，因為 v4 的 check pipeline 架構更複雜（換來了更好的錯誤訊息和 async 支援）。作者還基於同樣的思路做了一個 AOT 編譯器 zod-aot，build time 直接把 schema 編譯成 flat 的驗證函式，最快能比 v4 快 60 倍。

**重點：**
- Zod v4 的效能提升主要來自 `$ZodObjectJIT`，用 `new Function()` 在 runtime 生成特化驗證函式
- JIT 只作用在 object schema 上，primitive 和 collection 沒有覆蓋到
- 在 Cloudflare Workers 或嚴格 CSP 環境下 JIT 會被停用，自動 fallback 到標準路徑
- 但是... primitive 帶 checks 的場景 v4 反而比 v3 慢，這是架構升級的 trade-off

### [I Tried Vite+ and Replaced My Entire Frontend Toolchain](https://dev.to/erikch/i-tried-vite-and-replaced-my-entire-frontend-toolchain-4cgb)

VoidZero（Evan You 創辦的公司）推出了 Vite+，野心不小：一個 CLI 搞定 build（Vite 8 + Rolldown）、lint（Oxlint）、format（Oxfmt）、test（Vitest）、甚至 Node.js 版本管理。不用再裝 ESLint、Prettier、husky、lint-staged，全部塞進一個 `vite.config.ts`。

作者實際用 `vp create vue` 建了專案，體驗是：`vp check` 在大型 codebase 上不到一秒跑完（底層用了 Rust 寫的 Oxc 工具鏈），pre-commit hooks 開箱即用，格式化和 linting 都整合進同一個設定檔。但也踩了坑——Nuxt 的 config 還沒完全整合、Next.js 模板有問題、`vp` 指令會覆蓋 `package.json` 的 scripts（Theo Browne 在直播上直接罵了這個設計）。

**重點：**
- Oxlint 比 ESLint 快 50-100x，Oxfmt 比 Prettier 快 30x，Vite 8 + Rolldown 生產建置快 1.6-7.7x
- Type checking 用了 tsgolint（基於微軟官方的 tsgo——TypeScript 的 Go 移植版）
- 還在 alpha 階段，Nuxt 和 Next.js 整合尚未完善
- 但是... `vp` 覆蓋 package.json scripts 的設計很有爭議，而且目前不支援 Bun

### [Be intentional about how AI changes your codebase](https://aicode.swerdlow.dev)

這篇不是在講什麼新框架，而是在講一個更根本的問題：怎麼寫出讓人（和 AI）都能理解的程式碼結構。作者把函式分成兩類——Semantic Functions（語意函式，像 `quadratic_formula()`，職責明確、無副作用、可重複使用）和 Pragmatic Functions（務實函式，像 `handle_user_signup_webhook()`，是一堆語意函式的組合，預期會頻繁變動）。

聽起來像教科書？差別在於作者講到了「東西怎麼壞的」：semantic function 為了方便被改成 pragmatic function，結果依賴它的地方全部出事；Model 一開始很乾淨，然後有人加了「就一個」optional field，接著又一個，最後變成一坨什麼都有但什麼都不確定的資料結構。這在 AI 輔助開發的時代特別重要——如果你的程式碼結構不清晰，AI 生成的 code 也會跟著爛。

**重點：**
- Semantic functions 應該無副作用、所有輸入輸出都顯式宣告、不需要註解就能看懂
- Pragmatic functions 是業務邏輯的容器，應該有 doc comment 但只寫「意外行為」
- Model 的設計應該讓「不合法的狀態無法被建構」，用 brand types 避免型別混淆
- 但是... 現實中大家還是會為了方便把 semantic function 改成什麼都做的怪物

---

## ⚡ 一句話帶過

- **[Why Your Hero Section Is Killing LCP](https://dev.to/pawar-shivam7/why-your-hero-section-is-killing-lcp-24dm)** — 你的 hero image 可能是 LCP 最大的絆腳石，這篇教你怎麼修
- **[HTTP 402 Was 'Reserved for Future Use' for 29 Years. Stripe Just Gave It a Job.](https://dev.to/adioof/http-402-was-reserved-for-future-use-for-29-years-stripe-just-gave-it-a-job-3phg)** — HTTP 402 Payment Required 從 1997 年「保留」到現在，Stripe 終於讓它上班了
- **[No AI in Node.js Core](https://github.com/indutny/no-ai-in-nodejs-core)** — 有人在 GitHub 發起反對把 AI 功能塞進 Node.js 核心的倡議，值得關注後續發展
- **[Ng-News: AI & Angular, debounced() in v22, Oxidation Compiler in Analog](https://dev.to/playfulprogramming-angular/ng-news-2609-ai-angular-debounced-in-v22-oxidation-compiler-in-analog-4n3)** — Angular v22 加了 Signal-based `debounced()` API，Analog.js 開始用 Oxidation compiler
- **[Headless Storybook with Lit](https://dev.to/jamesives/headless-storybook-with-lit-511)** — 用 Lit 搭配 headless Storybook 建 design system，Web Components 的人可以看看
- **[Claude Code: Channels](https://code.claude.com/docs/en/channels)** — Claude Code 推出 Channels 功能，HN 上剛開始討論
- **[Build a Schema-Based Wizard Form in Vue 3](https://dev.to/parsajiravand/build-a-schema-based-wizard-form-in-vue-3-no-templates-needed-2h8p)** — 用 schema 驅動 Vue 3 多步驟表單，不用寫 template，思路蠻乾淨的
- **[基于 Cloudflare 生态的 AI Agent 实现](https://juejin.cn/post/7618852263628701759)** — 用 Cloudflare Workers + D1 + Vectorize 全家桶做 AI Agent，架構值得參考
- **[I Built 48 Lightweight SVG Backgrounds](https://dev.to/theawesomeblog/i-built-48-lightweight-svg-backgrounds-that-will-transform-your-web-projects-and-you-can-4of9)** — 48 個輕量 SVG 背景，copy-paste 即用，做 side project 的時候很實用
- **[$20 的 Cursor Pro 额度，这样用一个月都花不完](https://juejin.cn/post/7618873354036674596)** — Cursor Pro 的額度管理心得，核心思路是「用在刀刃上」而不是「少用」

---

## 📚 慢慢啃

- **[The Browser Is Already a Supercomputer. You Just Have to Ask.](https://dev.to/web_dev-usman/the-browser-is-already-a-supercomputer-you-just-have-to-ask-24bk)** — 盤點 10 個不用裝任何 library 就能用的 Browser API，有些你可能真的沒聽過。週末花 15 分鐘掃一遍，說不定下次就省下一個 dependency
- **[The Ultimate Guide to Modern Web Performance Optimization in 2026](https://dev.to/joaopakina/the-ultimate-guide-to-modern-web-performance-optimization-in-2026-3p1l)** — 2026 年版的 Web 效能優化全攻略，從 Core Web Vitals 到 rendering 策略都有涵蓋，適合當 checklist 用
- **[Rethinking open source mentorship in the AI era](https://github.blog/open-source/maintainers/rethinking-open-source-mentorship-in-the-ai-era/)** — GitHub 官方部落格討論 AI 時代的開源 mentorship——當 PR 看起來很漂亮但其實是 AI 生成的，maintainer 該怎麼辦？這個問題會越來越嚴重
- **[WCAG - Links and accessible text](https://dev.to/dawid_ryczko/wcag-links-and-accessible-text-5bmn)** — 你的連結文字寫的是「click here」還是有意義的描述？這篇把 WCAG 的連結無障礙規範講得很清楚，是那種「知道了就再也不會寫錯」的知識
