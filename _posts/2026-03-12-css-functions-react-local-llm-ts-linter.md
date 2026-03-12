---
title: "CSS 變身計算機、React Hook 直連本地 LLM、TypeScript 架構守門員"
date: 2026-03-12
description: "CSS 新函式讓你少寫一半 JS、use-local-llm 讓 React 兩分鐘接上本地模型、architecture-linter 用 YAML 守住分層架構。另有 Expo 一鍵上架、WordPress 7.0 iframe 編輯器遷移指南等。"
tags: [frontend, css, react, typescript, ai, tooling]
---

今天三道硬菜都跟「少寫程式碼」有關。CSS 現在會算三角函數了（不是開玩笑），有人用 2.8 KB 解決了 Vercel AI SDK 在本地開發時的過度設計問題，還有人受不了 code review 一直抓架構違規所以寫了個 linter。嗯，工程師的懶惰果然是生產力。

---

## 🔧 今日硬菜

### [CSS Functions in 2026: Who Gave CSS a Calculator?!](https://dev.to/pixelperfect_pro/css-functions-in-2026-who-gave-css-a-calculator-4226)

還記得當年用 `calc()` 都要小心翼翼怕同事看不懂的日子嗎？現在 CSS 直接給你 `exp()`、`hypot()`、`sign()`、`anchor()`、`shape()`——這已經不是樣式語言，這是佈局計算引擎。`anchor()` 讓你不用再寫 `getBoundingClientRect()` + scroll offset + resize observer 那套組合拳就能定位 tooltip；`sign()` 基本上就是 CSS 版的 if statement。最狠的是 `anchor-size()`，dropdown 寬度等於按鈕寬度這件事終於不需要 JS 了。

**重點：**
- `anchor()` / `anchor-size()` 取代大量 JS 定位邏輯，tooltip 和 floating UI 的實作方式正在改變
- `sign()` + `exp()` + `hypot()` 讓 CSS 具備條件判斷和數學運算能力，響應式設計不再依賴 JS
- 但是... 瀏覽器支援度還是老問題，`anchor()` 目前主要在 Chromium 系，Safari 和 Firefox 還在追趕

### [use-local-llm: React Hooks for AI That Actually Work Locally](https://dev.to/pooyagolchian/use-local-llm-react-hooks-for-ai-that-actually-work-locally-4ghc)

踩過這個坑的都知道——你本地 Ollama 跑得好好的，想在 React 裡接上去，結果 Vercel AI SDK 硬要你架一層 API route。為了一個 prototype 多寫一個後端？拜託。`use-local-llm` 就是一個 2.8 KB、零依賴的 React Hook 庫，`useOllama("gemma3:1b")` 一行搞定，瀏覽器直連 `localhost:11434`，streaming 回應、中斷生成、TypeScript 型別全包。還支援 Ollama、LM Studio、llama.cpp 自動偵測。

**重點：**
- 架構極簡：Browser → fetch() → localhost，沒有中間層，prototype 兩分鐘上手
- 核心函式 `streamChat()` / `streamGenerate()` 是 async generator，React 之外也能用
- 但是... 僅適用於本地開發和隱私場景，生產環境你還是需要 Vercel AI SDK 那套認證和限流機制

### [I built a linter that enforces Clean Architecture rules in TypeScript](https://dev.to/cvalingam/i-built-a-linter-that-enforces-clean-architecture-rules-in-typescript-483g)

每個 TypeScript 專案遲早會出現 controller 直接 import repository 的那天。ESLint 抓不到這種架構層級的違規，code review 又不是每次都能抓到。`architecture-linter` 用一個 `.context.yml` 定義層級關係和禁止的 import 方向，CLI、VS Code 擴充和 GitHub Action 三管齊下。內建 NestJS、Clean Architecture、Hexagonal、Next.js、Express 預設模板，`npx architecture-linter init` 自動偵測你的分層結構。

**重點：**
- 用 YAML 宣告式定義架構規則，比在 ESLint config 裡硬寫 import 路徑限制乾淨太多
- VS Code 即時顯示紅色波浪線 + GitHub Action PR 標註，違規在 merge 之前就被攔下來
- 但是... 專案還很新，預設模板可能不完全符合你的架構變體，自訂規則的靈活度有待觀察

---

## ⚡ 一句話帶過

- **[LiteLLM Gateway now supported on Vercel](https://vercel.com/changelog/litellm-gateway-now-supported-on-vercel)** — Vercel 現在能跑 LiteLLM Gateway，一個 OpenAI 相容閘道接所有 LLM provider，省得你自己寫 adapter
- **[Challenge: iOS App from Zero to a Friend's Phone in 1 Hour](https://dev.to/cathylai/challenge-ios-app-from-zero-to-a-friends-phone-in-1-hour-4gfo)** — Expo EAS Launch 號稱「行動端的 Vercel」，一鍵上架 App Store，實測看看是不是真的
- **[Building Better Country Selects](https://talysto.com/blog/building-better-country-selects/)** — 國家選擇器這種看似簡單的 UI，踩坑清單比你想像的長很多
- **[Bear UI v1.1.4: 22+ New Components](https://dev.to/john_yaghobieh_8f294091f6/bear-ui-v114-22-new-components-loc-badges-and-a-better-docs-experience-2545)** — 又一個 React + Tailwind 元件庫，TypeScript-first，零配置，看看能不能打進 shadcn 的市場
- **[dirham: The UAE Dirham Symbol (U+20C3) as Web Font & React Component](https://dev.to/pooyagolchian/dirham-the-uae-dirham-symbol-u20c3-as-web-font-css-utility-react-component-4dni)** — Unicode 18.0 還沒發布，先用 npm 包墊著，做中東市場的前端會需要
- **[How to find missing i18n keys without losing your mind](https://dev.to/dev_harry/how-to-find-missing-i18n-keys-without-losing-your-mind-or-your-job-1jff)** — 上線後按鈕顯示 `{{ settings.labels.confirm_action }}`？對，就是忘了翻譯 key
- **[WordPress 7.0 Iframed Editor: Migration Playbook](https://dev.to/victorstackai/wordpress-70-iframed-editor-migration-playbook-for-meta-boxes-plugins-and-admin-js-55pm)** — WP 7.0 Beta 1 把編輯器塞進 iframe 了，你的 meta box 和 admin JS 準備好了嗎
- **[8 GLSL Shader Systems in 6 Months](https://dev.to/joshtol/8-glsl-shader-systems-in-6-months-building-elemental-effects-for-a-character-animation-engine-2jmb)** — 開源角色動畫引擎的 WebGL shader 開發紀錄，火焰、水流、冰、電擊全手刻，視覺效果值得一看
- **[Why LLMs Suck at Calling APIs (And How Flat Schemas Fix It)](https://dev.to/docat0209/why-llms-suck-at-calling-apis-and-how-flat-schemas-fix-it-o0j)** — LLM 不會組巢狀 JSON，把 schema 攤平正確率直接上升，做 MCP / function calling 的必讀
- **[Automating Prop Schema Generation from TypeScript Interfaces](https://dev.to/jasonbiondo/automating-prop-schema-generation-from-typescript-interfaces-for-zero-config-component-registration-2ocg)** — 從 TS interface 自動生成 visual page builder 的元件 schema，少寫大量重複定義

---

## 📚 慢慢啃

- **[Many SWE-bench-Passing PRs would not be merged](https://metr.org/notes/2026-03-10-many-swe-bench-passing-prs-would-not-be-merged-into-main/)** — METR 的研究顯示 AI 生成的 PR 就算通過了 SWE-bench 測試也不見得能合併，code review 標準和「測試通過」之間的差距比你以為的大
- **[The Hardest Problems in Real-Time Front-End Engineering](https://dev.to/ricardosaumeth/--1fk8)** — 十年經驗的工程師談即時前端系統的真實痛點：WebSocket 不是開了就好，state 同步、背壓處理、斷線重連才是地獄
- **[The dead Internet is not a theory anymore](https://www.adriankrebs.ch/blog/dead-internet/)** — HN 291 分的熱文，AI 生成內容正在淹沒網路，這不再是陰謀論而是你每天都在體驗的現實
- **[Preliminary data from a longitudinal AI impact study](https://newsletter.getdx.com/p/ai-productivity-gains-are-10-not)** — DX 的縱向研究數據：AI 帶來的生產力提升可能是 10% 而不是大家嘴上說的 50%，值得冷靜看看
