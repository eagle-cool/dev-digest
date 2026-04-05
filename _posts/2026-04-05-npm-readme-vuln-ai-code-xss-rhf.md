---
title: "npm README 教壞人、AI 寫碼 86% 擋不住 XSS"
date: 2026-04-05
description: "npm 五大套件的 README 範例在教你寫漏洞、Veracode 實測 150+ AI 模型 45% 產出有 OWASP 漏洞、React Hook Form 搭 Next.js App Router 的表單解耦實戰。"
tags: [frontend, nodejs, ai, typescript]
---

今天有兩篇安全相關的文章撞在一起看特別有感。一篇說 npm 套件的 README 範例在「教」開發者寫漏洞，另一篇說 AI 寫的程式碼 86% 防不住 XSS。你把這兩件事加在一起想——AI 學的訓練資料裡面就包含那些有問題的 README 範例——突然就不意外了。另外還有一篇 React Hook Form 的深度教學，算是今天少數不讓人焦慮的文章。

---

## 🔧 今日硬菜

### [The Documentation Attack Surface: How npm Libraries Teach Insecure Patterns](https://dev.to/ethan_kreloff_4a7339e3d1d/the-documentation-attack-surface-how-npm-libraries-teach-insecure-patterns-2j6j)

這篇的核心論點簡單到讓人不舒服：npm 套件的程式碼是安全的，但 README 裡的範例是不安全的。作者審計了五個總計每週 1.95 億下載量的套件——axios、jsonwebtoken、cors、crypto-js、multer——每一個都有同樣的問題。axios 的 `beforeRedirect` 範例會在 HTTPS 降級成 HTTP 後重新注入憑證；cors 的 regex 範例 `/example\.com$/` 會讓 `evil-example.com` 通過；multer 的範例用 `Math.random()` 產生檔名，而套件自己的預設值用的是 `crypto.randomBytes(16)`。

踩過這個坑的人都知道，開發者不讀 source code——他們複製 README。這讓文件變成了事實上的 API，而這個 API 沒有經過 code review。

**重點：**
- axios 的 `beforeRedirect` 範例會繞過 `follow-redirects` 刻意設計的憑證剝離機制（已提交 GHSA-8wrj-g34g-4865）
- cors 套件的測試檔用的是正確的 regex，但 README 教的是有漏洞的版本
- 但是... 這不是單一套件的 bug，而是整個 npm 生態系的系統性問題——README 是最少被 review 的「程式碼」

### [Your AI-Generated Code Isn't Secure — Here's What We Find Every Time](https://dev.to/anatolysilko/your-ai-generated-code-isnt-secure-heres-what-we-find-every-time-3h63)

Veracode 實測 150+ AI 模型，45% 產出的程式碼包含 OWASP Top 10 漏洞。XSS 防禦的失敗率是 86%——而且新模型並沒有比舊模型好。DryRun Security 的研究更有趣：AI 確實寫了 rate limiting middleware，但沒有一個 agent 真正把它接上應用程式。安全網存在於程式碼裡，只是不工作。

GitGuardian 分析了 19.4 億筆 public GitHub commit，發現 AI 輔助的 commit 洩漏 secret 的比率是一般的兩倍（3.2% vs 1.5%）。Supabase 憑證洩漏暴增 992%。有個用 Cursor 建的 SaaS 創辦人，上線幾天就被找到暴露的 API key，被刷了 $14,000 的 OpenAI 帳單後直接關站。

**重點：**
- 150+ 模型、80 個任務、Java 最慘 72% 失敗率——模型大小對安全品質沒有顯著影響
- 英國 NCSC 用了「intolerable risks」這個詞來形容 vibe coding 產出的程式碼
- 但是... 問題其實不難修，因為都是基本功——硬編碼 secret、沒有參數化查詢、缺安全 headers——難的是讓用 AI 寫碼的人意識到需要修

### [Engineering Scalable Forms: Decoupling Logic with RHF and Next.js App Router](https://dev.to/devansh_prajapati_c7997a0/engineering-scalable-forms-decoupling-logic-with-rhf-and-nextjs-app-router-4h8a)

終於來一篇不讓人焦慮的。這篇用一個 Job Application Portal 的範例，示範怎麼用 React Hook Form 的 `FormProvider` + `useFormContext` + `useFieldArray` 在 Next.js App Router 裡做多步驟表單。重點在「解耦」——每個表單區塊（個人資料、工作經驗、技能）都是獨立的 client component，透過 Context 共享表單狀態，不用 prop drilling。

架構上值得注意的是 `control` 物件的角色。它不只是一個 prop，而是 RHF 內部狀態管理的橋樑——`useFieldArray` 透過它知道刪除第 3 項時該怎麼 shift 剩下的資料，而且只觸發該區塊的 re-render。對於路由切換時的狀態持久化，這個 pattern 比自己用 `useState` + `useEffect` 硬刻強太多。

**重點：**
- `FormProvider` + `useFormContext` 實現跨元件的表單狀態共享，元件可以在不同 wizard step 間自由移動
- `useFieldArray` 搭配 Object Path Notation（`experience.${index}.years`）處理動態列表，效能優於手動管理陣列 state
- 但是... 文章沒有提到 Server Action 整合和表單資料的 server-side validation，在 App Router 裡這塊其實才是真正的痛點

---

## ⚡ 一句話帶過

- **[MCP Connector Poisoning: How Compromised npm Packages Hijack Your AI Agent](https://dev.to/toniantunovic/mcp-connector-poisoning-how-compromised-npm-packages-hijack-your-ai-agent-3ha0)** — axios 被劫持那事的後續：攻擊者現在可以透過被污染的 npm 套件直接劫持你的 MCP server，AI agent 時代的 supply chain attack 又多了一個攻擊面
- **[Apple approves driver that lets Nvidia eGPUs work with Arm Macs](https://www.theverge.com/tech/907003/apple-approves-driver-that-lets-nvidia-egpus-work-with-arm-macs)** — Apple Silicon Mac 終於能外接 Nvidia GPU 了，等了五年的人可以哭了（HN 350+ 讚）
- **[Show HN: TurboQuant-WASM – Google's vector quantization in the browser](https://github.com/teamchong/turboquant-wasm)** — Google 的向量量化演算法編譯成 WASM 在瀏覽器跑，on-device ML 又近了一步（HN 134 讚）
- **[I built Material Symbols SVG](https://dev.to/k-s-h-r/material-symbols-svg-material-symbols-as-svg-components-across-frameworks-11lm)** — Google Material Symbols 的跨框架 SVG 元件版，支援 React/Vue/Svelte/Astro，終於不用再載 icon font 了
- **[Show HN: M. C. Escher spiral in WebGL](https://static.laszlokorte.de/escher/)** — 受 3Blue1Brown 啟發用 WebGL fragment shader 重現 Escher 的螺旋畫廊效果，純粹的 shader 藝術
- **[Chrome zero-day CVE-2026-5281 patched](https://dev.to/mrcomputerscience/breaking-cybersecurity-news-for-20260404-pithy-cyborg-threats-breaches-intel-bok)** — Chrome 今年第四個零日漏洞，這次在 WebGPU 的 Dawn 引擎，use-after-free，趕快更新
- **[CVE-2026-32211: Azure MCP Server Flaw — CVSS 9.1](https://dev.to/michael_onyekwere/cve-2026-32211-what-the-azure-mcp-server-flaw-means-for-your-agent-security-14db)** — Azure MCP Server 完全沒有身份驗證機制，CVSS 9.1，目前沒有 patch，用的人先停一下
- **[I Put VS Code, Claude, and a Terminal Inside a File Manager](https://dev.to/kimlimjustin/i-put-vs-code-claude-and-a-terminal-inside-a-file-manager-i-built-using-react-and-rust-heres-39hi)** — React + Rust 打造的檔案管理器 Xplorer，把編輯器、AI、終端全塞進去，野心不小
- **[12,000 AI-generated blog posts added in a single commit](https://github.com/OneUptime/blog/commit/30cd2384794c897d95aca77d173db44af51ca849)** — 一個 commit 塞進 12,000 篇 AI 生成的文章，這就是 2026 年的 SEO（HN 96 讚，評論區很精彩）
- **[StackWeave — fullstack monorepo for web, mobile & backend](https://dev.to/amitgharge/stackweave-a-fullstack-monorepo-for-web-mobile-backend-and-why-i-built-it-27o0)** — 受夠了三個 repo 三個 package.json 同步 schema，做了一個 web + mobile + API 的 monorepo 模板

---

## 📚 慢慢啃

- **[Why Are We Still Using Markdown in 2026?](https://dev.to/castillodk/why-are-we-still-using-markdown-in-2026-15db)** — 從 Gruber 的設計哲學出發，分析 Markdown 為什麼在富文本編輯器百花齊放的年代還是活得好好的。讀完你會重新理解「刻意的簡單」這件事
- **[Pitfalls of Claude Code](https://dev.to/cheetah100/pitfalls-of-claude-code-1nb6)** — 從 ChatGPT 寫小函式到用 Claude Code 改整個 codebase 的實戰心得，踩坑紀錄比官方文件有用十倍
- **[AI subscriptions are subsidized. Here's what happens when that stops.](https://dev.to/dzhuneyt/ai-subscriptions-are-subsidized-heres-what-happens-when-that-stops-293f)** — OpenAI 2025 年每賺一塊錢就虧 $1.69，$200/月的 Pro plan 還是賠錢。當補貼結束，你的開發工作流程會怎樣？
- **[How We Built a Free Browser-Based Screen Recorder](https://dev.to/paweldotio/how-we-built-a-free-browser-based-screen-recorder-4m04)** — 用 MediaRecorder API、Canvas 合成、Web Audio 做的純瀏覽器錄影工具，Web API 實戰的好教材
