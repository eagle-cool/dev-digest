---
title: "Reactivity 不只是 UI、圖片格式終極對決、npm 套件內戰"
date: 2026-03-17
description: "深入探討 Reactivity 在資料層的應用挑戰、2026 年 AVIF vs WebP vs JPEG XL 實測數據對決、以及 npm 套件版本衝突時該果斷回歸系統工具的血淚經驗。"
tags: [frontend, react, web-platform, tooling, ai]
---

今天有一篇文章讓我眼睛一亮——有人把 Reactivity 從 UI 拉到了資料層，講的是那些 Solid.js 和 Vue 不會告訴你的事。然後是 2026 年圖片格式大亂鬥的真實數據（劇透：JPEG XL 回來了），再來是一個經典的 npm 套件打架故事，結局是回歸 `apt-get install`。

---

## 🔧 今日硬菜

### [Reactivity beyond React](https://dev.to/chronograph/reactivity-beyond-react-4hgc)

這篇不是又一個「React vs Vue vs Solid」的框架戰文。作者把 Reactivity 的概念從 UI 渲染拉到了**資料層**——用 signal 和 computed 來處理業務邏輯、交易回滾、循環依賴這些硬核問題。

他用 ChronoGraph 這個 reactive library 實作了一個甘特圖排程引擎，過程中發現 UI reactivity 的假設在資料層完全不適用。比如：你需要 transaction 能 commit 也能 reject（UI 不需要這個）；你的 signal 可能同時是可寫又是可計算的（想想行事曆的 startDate、endDate、duration 三者互相連動）；linked list 超過 2000-3000 個節點就會爆 stack（他用 generator 做了 stack inversion 解決）。

最讓我印象深刻的是「cyclic computation」的處理——表單裡一堆欄位互相依賴又都要可編輯，這在實務上超常見，但現有的 reactive library 幾乎都假設 DAG（無環圖）。踩過這個坑的人都知道，一旦有環，整個心智模型就崩了。

**重點：**
- Reactivity 在資料層需要 transaction、commit/reject 語義——UI 框架完全沒處理這塊
- Signal 的「可寫又可計算」雙重身份能把節點數砍半（從 2M 降到 1M），GC 壓力直接減半
- 但是... 這目前還是研究階段，ChronoGraph 只在甘特圖引擎驗證過，通用場景還有很長的路

### [AVIF vs WebP vs HEIC vs JPEG XL: which image format should you use in 2026?](https://dev.to/serhii_kalyna_730b636889c/avif-vs-webp-vs-heic-vs-jpeg-xl-which-image-format-should-you-use-in-2026-4gn0)

終於有人用真實數據把這四個格式攤開來比了，不是那種「A 比 B 好」的廢話，而是具體到一張 2000×2000 的產品照：JPEG 540KB、WebP 350KB、JPEG XL 260KB、AVIF 210KB。

但 AVIF 也不是沒有代價——編碼時間是 JPEG 的 400-800 倍（1-2 秒 vs 2.5ms），解碼也慢了 5.4 倍。JPEG XL 在高品質場景反而比 AVIF 小 20%，而且支援 progressive decoding，大圖在慢網路上先顯示預覽。最大的新聞是：**Google 撤回了 2022 年從 Chrome 移除 JPEG XL 的決定**，Rust 解碼器已經進了 Chromium Canary 145，預計 2026 下半年進 stable。

目前的部署現實：WebP 19.4% 網站在用，AVIF 才 1.3%。HEIC 在 web 上基本宣告死亡（只有 Safari 支援，專利問題無解）。

**重點：**
- 2026 年最佳策略：AVIF → WebP → JPEG fallback chain，用 `<picture>` 或 CDN auto-negotiation
- JPEG XL 回歸 Chrome 是大事——progressive decoding 對大圖體驗是質的飛躍
- 但是... AVIF 編碼慢到你會懷疑人生，批次處理請備好耐心（或用 CDN 自動轉檔）

### [When Two npm Packages Fight Over pdfjs-dist: Drop to System Binaries](https://dev.to/agent_paaru/when-two-npm-packages-fight-over-pdfjs-dist-drop-to-system-binaries-145a)

經典場景：在 Next.js 裡同時用 `unpdf` 和 `pdf-to-img`，兩個套件各自 bundle 了不同版本的 `pdfjs-dist`（5.4.296 vs 5.4.624），Worker 版本衝突，`npm dedupe` 救不了你，因為兩邊都是 bundle 進去的，不是 peer dep。

作者花了四小時排查，最後的解法讓人拍大腿：直接 `apt-get install poppler-utils tesseract-ocr`，20 行 `execSync` 搞定，零 npm 依賴，零版本衝突。

這個思路值得推廣：**當一個 npm 套件的本質是「用 JS 包裝一個系統二進位檔」時，先問自己真的需要這層包裝嗎？** pdftoppm 做 PDF 轉圖做了幾十年，tesseract 做 OCR 也做了幾十年。有時候最好的 Node.js 套件就是 `execSync`。

**重點：**
- 多個套件各自 bundle 同一個 library 的不同版本 = 無解的版本地獄
- 系統工具（poppler、tesseract、ffmpeg）穩定快速，Dockerfile 加兩行就搞定
- 但是... 這意味著你的 Docker image 會變大，而且 serverless 環境可能沒辦法裝系統套件

---

## ⚡ 一句話帶過

- **[Your AI Agent Has Been Coding Blind. Chrome Just Gave It Eyes.](https://dev.to/adioof/your-ai-agent-has-been-coding-blind-chrome-just-gave-it-eyes-49la)** — Chrome DevTools 終於讓 AI agent 能「看到」渲染結果了，不再是寫完 CSS 就 peace out
- **[Private-First AI: Building a Browser-Based Mental Health Classifier with WebLLM and WebGPU](https://dev.to/beck_moulton/private-first-ai-building-a-browser-based-mental-health-classifier-with-webllm-and-webgpu-4ai2)** — WebLLM + WebGPU 讓模型跑在瀏覽器裡，資料完全不離開本機，隱私敏感場景的正確打開方式
- **[零成本本地大模型！用 Next.js + Ollama + Qwen3 打造流式聊天應用](https://juejin.cn/post/7617728986828816411)** — Next.js 串 Ollama 跑本地 Qwen3，streaming 輸出打字機效果，零 API 費用的 AI 整合入門
- **[Hedystia 2.0: A New Type-Safe ORM for TypeScript with Multi-Database Support](https://dev.to/zastinian/hedystia-20-a-new-type-safe-orm-for-typescript-with-multi-database-support-2pfa)** — 又一個 TypeScript ORM，號稱結合 Drizzle 的 schema 定義和 TypeORM 的查詢 API，先觀望三個月
- **[Revolutionizing Your Frontend Workflow: A Deep Dive into VitePlus](https://dev.to/benriemer/revolutionizing-your-frontend-workflow-a-deep-dive-into-viteplus-dd3)** — Vite 的增強版 VitePlus 登場，更快的 build、更簡單的設定，但標題用了「Revolutionizing」所以先扣分
- **[告别登录中断：前端双 Token 無感刷新](https://juejin.cn/post/7617688223624314889)** — Access Token + Refresh Token 的無感續期機制拆解，前端 auth 的基本功但很多人做不好
- **[基于 AST 与 Proxy 沙箱的局部代码热验证](https://juejin.cn/post/7617682877929390131)** — 用 AST 解析只驗證修改過的程式碼區塊，不用全量編譯，開發體驗的微優化但思路值得借鑑
- **[Top 5 Structured Output Libraries for LLMs in 2026](https://dev.to/nebulagg/top-5-structured-output-libraries-for-llms-in-2026-48g0)** — TypeScript 用 Zod + zodResponseFormat、Python 用 Instructor，要 guaranteed schema 就上 Outlines
- **[Claude Code vs Cursor: What I Learned Using Both for 30 Days](https://dev.to/hugo662/claude-code-vs-cursor-what-i-learned-using-both-for-30-days-17en)** — 結論：Cursor 是加速器讓你做得更快，Claude Code 是代理人讓你做不同的事，不是同一個賽道
- **[Leanstral: Open-Source foundation for trustworthy vibe-coding](https://mistral.ai/news/leanstral)** — Mistral 推出 Leanstral，用形式化驗證讓 vibe coding 產出的程式碼可信，HN 244 分不是沒原因的

---

## 📚 慢慢啃

- **[How to explain React hooks in interviews](https://dev.to/krzysztof_fraus/how-to-explain-react-hooks-in-interviews-odo)** — 不是又一篇 useState/useEffect 教學，而是教你怎麼在面試中講出 hooks 背後的心智模型，讀完會重新理解為什麼 hooks 要按順序呼叫
- **[Trust Debt: The Production Crisis Hidden Inside AI-Generated Codebases](https://dev.to/deepak_mishra_35863517037/trust-debt-the-production-crisis-hidden-inside-ai-generated-codebases-125h)** — AI 生成的程式碼帶來了一種新型技術債：你不理解的程式碼比你寫錯的程式碼更危險，這篇把這個問題講得很透
- **[Speed at the cost of quality: Study of use of Cursor AI in open source projects](https://arxiv.org/abs/2511.04427)** — 學術論文分析 Cursor AI 在開源專案中的使用，結論是速度提升了但品質下降了，HN 68 分，值得認真讀
- **[How AST Made AI-Generated Functions Actually Reliable](https://dev.to/zarif007/how-ast-made-ai-generated-functions-actually-reliable-3kbn)** — 不讓 AI 直接寫程式碼，而是讓它描述邏輯再用 AST 生成，安全性和可預測性直接上了一個台階
