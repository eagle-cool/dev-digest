---
title: "HTTP 402 終於有人用了、Cursor 四站合一、AI SDK 多 Agent 編排實戰"
date: 2026-04-16
description: "HTTP 402 支付協議由 Linux Foundation 推動正式落地、Cursor 用 Vercel Microfrontends 統一四站提升 5% 註冊率、Vercel AI SDK 搭 open-multi-agent 實作多 Agent 協作——三道硬菜都在講架構決策怎麼影響產品"
tags: [frontend, web-platform, ai, tooling]
---

今天值得聊的就三件事。一個沉睡 27 年的 HTTP status code 突然被一群大廠搶著用（Google、AWS、Microsoft、Stripe、Visa 全到齊），Cursor 用 Vercel Microfrontends 把四個散落各處的網站合成一個還順便拉了 5% 註冊率，然後有人用 60 行膠水程式碼把 Vercel AI SDK 跟多 Agent 框架接在一起——不是 demo，是真的能跑的那種。

---

## 🔧 今日硬菜

### [HTTP 402 waited 27 years for this moment: the x402 Foundation and the agent economy](https://dev.to/aaron_schnieder_4563d5d33/http-402-waited-27-years-for-this-moment-the-x402-foundation-and-the-agent-economy-db9)

HTTP 402 Payment Required 從 1999 年就躺在 HTTP spec 裡面，但 27 年來它的描述一直是「reserved for future use」——是整個 HTTP 規範裡最尷尬的存在。直到 AI Agent 出現了：Agent 不會填表單、不會輸信用卡號、不會按「我同意」，它需要的是 programmatic、毫秒級、sub-cent 級的付款能力。x402 就是為了這個場景設計的。

x402 Foundation 上週在 Linux Foundation 下正式成立，founding members 的名單比 W3C 會議的出席率還豪華：Google、AWS、Microsoft、Cloudflare 負責運算基礎設施，Stripe、Visa、Mastercard、American Express、Adyen 負責支付管道，Coinbase 貢獻了底層協議。這些公司平常要同意一件事比通過 TC39 提案還難，但他們在 x402 上達成了共識。

流程很簡單：Agent 打 API → 收到 402 + 付款需求 → Agent 簽署付款 → 資源解鎖。單次付款可以低到 $0.003，這種顆粒度傳統支付管道根本做不到。目前已經有 1.4 億筆交易、$6 億+ 成交量、50 萬+ 活躍 Agent 錢包在跑。

**重點：**
- x402 讓 HTTP 402 從「最沒用的 status code」變成 machine-to-machine 支付的標準介面
- Agent economy 的基礎設施正在成形：身份（ERC-8004）、支付（x402）、託管（ERC-8183）
- 但是... 底層跑在 Coinbase 的 L2 鏈上用 USDC 結算，傳統金融機構加入了但 crypto 依然是核心支付軌道——這對某些企業來說是 dealbreaker

### [How Cursor built a growth iteration loop with Vercel Microfrontends and Flags](https://vercel.com/blog/how-cursor-built-a-growth-iteration-loop-with-vercel-microfrontends-and-flags)

Cursor 的 growth team 做了一件很多公司想做但不敢做的事：把四個分散在不同 repo 的網站（marketing site、docs、sign-in dashboard、help center）用 Vercel Microfrontends 整合到 cursor.com 下面。總共 100+ routes，三個階段漸進式遷移，全程零 downtime。

技術決策裡最有意思的是他們處理 experiment flicker 的方式：在 build time 就預先計算所有實驗變體，犧牲 build 時間來換取零 layout shift。這跟大部分團隊「runtime 判斷 + loading state」的做法完全相反，但對於 marketing page 這種第一印象決定轉換率的場景，這是正確的 trade-off。

另一個值得注意的是他們放棄了傳統 CMS。原因不是 CMS 不好，而是在 agent-driven workflow 裡，CMS 要求你手動登入後台操作，跟 GitHub PR 自動化流程完全不相容。他們用 200+ parallel agents 跑了一週，花了約 $260 token 費用就完成了 CMS 遷移。

**重點：**
- PLG 註冊率提升 5%（測試「Contact sales」導航按鈕時發現的），語言支援從 4 種擴展到 11 種
- Microfrontends 讓他們可以漸進遷移而不是 big bang——這是大型前端重構的正確打開方式
- 但是... 這整套架構高度依賴 Vercel 生態（Microfrontends、Flags SDK、Edge Config），vendor lock-in 的風險自己要評估

### [Adding Multi-Agent Orchestration to a Vercel AI SDK App](https://dev.to/jackchenme/adding-multi-agent-orchestration-to-a-vercel-ai-sdk-app-4536)

如果你用 Vercel AI SDK 做過 chatbot，遲早會碰到這個問題：一個 Agent 搞不定的事（例如先 research 再寫文章），你是要自己糊 `generateText` 的串接邏輯，還是用現成的 orchestration 框架？這篇文章選了後者——把 open-multi-agent（OMA）接進 Next.js API route，跟 AI SDK 的 `streamText` + `useChat` 共存。

架構分兩個 phase：Phase 1 是 OMA 的 `runTeam()` 負責任務分解和執行（coordinator 拆任務 → researcher 查資料寫進 shared memory → writer 讀 shared memory 寫文章），Phase 2 是 AI SDK 的 `streamText()` 把結果串流到前端。兩個 library 各做各的事，overlap 幾乎為零。

最實用的 insight 是那張對照表：AI SDK 強在 provider 支援（60+）和 streaming UI，OMA 強在 task decomposition 和 dependency DAG。你不需要二選一，它們可以疊在一起用。一個 gotcha：`@ai-sdk/openai` v2 預設打新的 Responses API（`/responses`），大部分 provider 還不支援，要用 `@ai-sdk/openai-compatible` 才行。

**重點：**
- OMA 的 `runTeam()` 自動處理任務拆解、拓撲排序、shared memory——你只要定義 agents 和 goal
- AI SDK v6 的 `useChat` 改了不少：沒有內建 `handleSubmit`、messages 用 `parts` 不是 `content`、`isLoading` 變成 `status` 四態
- 但是... OMA 的 provider 支援比 AI SDK 窄很多（只有 Anthropic、OpenAI-compatible、Gemini、Grok），而且不是 streaming-native，batch result 要透過 AI SDK 再串流出去

---

## ⚡ 一句話帶過

- **[Signals in Vue (I): A Minimal Bridge to the Composition API](https://dev.to/luciano0322/signals-in-vue-i-a-minimal-bridge-to-the-composition-api-45cf)** — 把自製的 signal/computed primitives 接進 Vue 3 Composition API，Signals 這個模式正在跨框架擴散
- **[Counting TypeScript Escape Hatches — A Zero-Dependency CLI with a Baseline Gate](https://dev.to/sendotltd/counting-typescript-escape-hatches-a-zero-dependency-cli-with-a-baseline-gate-1c9d)** — JS → TS 遷移中 `any` / `@ts-ignore` / `as unknown` 一直累積？這工具幫你在 CI 擋住，讓遷移真的會完成
- **[Reduced pricing for Turbo build machines](https://vercel.com/changelog/reduced-pricing-for-turbo-build-machines)** — Vercel Turbo build 降價 16%，30 CPU 機器每分鐘 $0.105，少但聊勝於無
- **[How to Compress SVG Files: Tools, Techniques, Config](https://dev.to/pixotter/how-to-compress-svg-files-tools-techniques-config-2028)** — SVG 是 XML 不是像素，壓縮邏輯完全不同——這篇把 SVGO 各設定項講得夠清楚
- **[Your MCP Server Is Probably Vulnerable](https://dev.to/bobbyblaine/your-mcp-server-is-probably-vulnerable-2135)** — 60 天內 30 個 CVE、2,614 個實作中 82% 有 path traversal 漏洞——MCP 生態的安全債讓人頭皮發麻
- **[pdf-lib Is an Under-Appreciated Gem: A Tiny PDF API in TypeScript](https://dev.to/sendotltd/pdf-lib-is-an-under-appreciated-gem-a-tiny-pdf-api-in-typescript-1jpk)** — 用 Hono + pdf-lib 搭的 PDF 微服務，純 JS 零 native dependency，190MB Alpine image，兩個下午搞定
- **[I scanned every major vibe coding tool for security. None scored above 90.](https://dev.to/shecantcode/i-scanned-every-major-vibe-coding-tool-for-security-none-scored-above-90-3opg)** — 非工程師用 AI 做的 app 安全掃描只拿 20/100，包括 critical auth bypass——vibe coding 的隱藏成本
- **[How to test Stripe webhook signatures locally without breaking verification](https://dev.to/fetchsandbox/how-to-test-stripe-webhook-signatures-locally-without-breaking-verification-5ggk)** — Stripe webhook 的 signature 驗證永遠是最先卡住你的地方，這篇講怎麼在本地正確測試
- **[Claude Code Plugins for Design Systems & Agent Orchestration for Real Workflows](https://dev.to/soytuber/claude-code-plugins-for-design-systems-agent-orchestration-for-real-workflows-3d7b)** — 用 Claude Code plugin 從網站自動抽取完整 design system，AI × 設計工程的實際應用
- **[Feature flags that show you how your rollout is actually performing](https://dev.to/ozturkaburak/feature-flags-that-show-you-how-your-rollout-is-actually-performing-4hm5)** — 大部分 feature flag 工具只告訴你開或關，不告訴你到底有沒有用——觀測性才是關鍵

---

## 📚 慢慢啃

- **[React vs. Next.js - What I Learned While Building Real-World Projects](https://dev.to/wafry_ahamed/react-vs-nextjs-what-i-learned-while-building-real-world-projects-54dd)** — 不是又一篇「React 是 library、Next.js 是 framework」的廢文，是從 admin dashboard、landing page、SaaS 模組實戰中提煉出的選型心法
- **[Your AI Agent Is One Bad URL Away From Being Compromised](https://dev.to/entropy0/your-ai-agent-is-one-bad-url-away-from-being-compromised-5202)** — Agent 框架的安全模型基本上是「URL 來了就 fetch、內容直接進 context window」——零驗證零信任檢查，這篇分析了為什麼這很危險
- **[I Tried Every Claude Code Editor. Here Is What Actually Works](https://dev.to/stravukarl/i-tried-every-claude-code-editor-here-is-what-actually-works-2ok)** — Claude Code 本身不難，難的是周圍的一切：工作規劃、多 session 追蹤、diff review、branch 管理。這篇比較了各種編輯器整合的實際體驗
