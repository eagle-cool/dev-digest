---
title: "use() 全解、FFmpeg 進瀏覽器、Next.js 多租戶"
date: 2026-04-06
description: "React 19 的 use() API 終於有人寫了完整指南、FFmpeg 編譯成 WASM 直接在瀏覽器跑轉檔、一個 Next.js 15 codebase 搞定多租戶網站"
tags: [frontend, react, nextjs, web-platform, tooling]
---

今天前端圈的菜色不錯。React 19 的 `use()` 出了一篇真正能拿來當參考文件的完整指南，有人把 FFmpeg 編譯成 WebAssembly 讓影片轉檔完全在瀏覽器裡跑（對，不用上傳），還有一位仁兄用一個 Next.js 15 codebase 硬是把多租戶網站建構器給蓋出來了。

---

## 🔧 今日硬菜

### [React 19 use() Hook: Guide to Promises and Context](https://dev.to/rahulxsingh/react-19-use-hook-guide-to-promises-and-context-395h)

這篇是目前我看過對 React 19 `use()` API 最完整的一篇實戰指南。重點不在告訴你 `use()` 長什麼樣（那三行 code 誰都會寫），而是把它跟 Suspense、Error Boundary 怎麼串在一起講清楚了。以前用 `useEffect` 抓資料的那套 boilerplate——useState 管 loading、管 error、管 cleanup——現在可以直接砍掉，component 只需要宣告「我要這個資料」，loading 跟 error 的事交給 tree 上面的 boundary 去處理。最值得注意的是 `use()` 技術上不算 hook，可以塞在 if/loop 裡面用，這在 conditional context 讀取的場景特別好用。

**重點：**
- `use()` 接受 Promise 或 Context，render 時直接讀值，搭配 Suspense 自動處理 loading
- 不受 hook rules 限制——可在條件式、迴圈中呼叫，這是設計上的刻意選擇
- 但別在 component body 裡面直接建 Promise（會無限 re-render），要用 useMemo 或從 Server Component / route loader 傳入

### [How I Built a Browser-Based Video Converter With FFmpeg & WebAssembly — No Server Required](https://dev.to/ali_salame/how-i-built-a-browser-based-video-converter-with-ffmpeg-webassembly-no-server-required-1bl8)

把 FFmpeg 編譯成 WebAssembly 跑在瀏覽器裡——檔案不用上傳、不用後端、不用花錢。架構很乾淨：File API 讀檔進 ArrayBuffer → ffmpeg.wasm 處理 → Blob URL 下載。整個流程零 server round-trip。作者把幾個關鍵的坑都講了：SharedArrayBuffer 需要 COOP/COEP header 才能用（這個會擋掉 cross-origin iframe，但對工具站來說不是問題），容器轉換可以用 `-c copy` stream copy 秒完成，以及 Safari 對 SharedArrayBuffer 的支援一直不太穩。最有意思的是他還開放了自訂 FFmpeg CLI 參數，等於把瀏覽器變成了一個完整的 FFmpeg GUI。

**重點：**
- FFmpeg.wasm 初載 ~30MB，之後有 cache，轉檔完全在 client 端完成
- 必須設 `Cross-Origin-Opener-Policy: same-origin` 和 `Cross-Origin-Embedder-Policy: require-corp` 才能啟用 SharedArrayBuffer
- 但這代表瀏覽器真的在變成一個通用運算環境——同樣的模式適用於 ImageMagick、SQLite、Python (Pyodide)

### [I Built a Multi-Tenant Website Builder with One Next.js App. Here's the Architecture.](https://dev.to/zenpage/i-built-a-multi-tenant-website-builder-with-one-nextjs-app-heres-the-architecture-35gn)

一個 Next.js 15 + React 19 的 codebase 同時跑 dashboard 和所有使用者的公開網站——用 middleware hostname routing 在同一個 deployment 裡切流量。技術選型很實在：Drizzle ORM + PostgreSQL、tRPC + SuperJSON、Cloudflare R2 存圖片。作者踩了幾個好坑值得記住：Tailwind v4 的 `@layer` 會吃掉 `dangerouslySetInnerHTML` 渲染內容的預設樣式（解法是用 inline style 繞過 PostCSS），composite slug 在多租戶下需要 scoped unique index，以及他自己寫 markdown parser 後悔了（建議直接用 unified/remark）。這種「一個 codebase 打天下」的架構很誘人，但前提是你要想清楚 route group 的邊界。

**重點：**
- Middleware hostname routing + route groups 讓 dashboard 和 tenant sites 在同一個 app 裡完全隔離
- Tailwind v4 的 CSS `@layer` 會導致 raw HTML 的預設樣式被 Preflight reset 吃掉——用 inline style 是目前最可靠的 workaround
- 自己寫 markdown parser 是個好練習但別在 production 搞——用 unified + remark + rehype

---

## ⚡ 一句話帶過

- **[dangerouslySetInnerHTML in React: Safe HTML Guide](https://dev.to/rahulxsingh/dangerouslysetinnerhtml-in-react-safe-html-guide-ole)** — 該用的時候就用，但拜託先 sanitize，這篇把 XSS 防禦講得夠清楚
- **[Privacy First: Running Llama-3 Locally in Your Browser for Medical Report Analysis via WebGPU](https://dev.to/beck_moulton/privacy-first-running-llama-3-locally-in-your-browser-for-medical-report-analysis-via-webgpu-4fdn)** — WebGPU + 本地 LLM 分析醫療報告，隱私派的夢想，但目前效能還是吃硬體
- **[How I render 10,000 live aircraft at 60fps in the browser with Rust, WASM, and raw WebGL2](https://dev.to/hao_jiang_c21b032bd6fbcfa/how-i-render-10000-live-aircraft-at-60fps-in-the-browser-with-rust-wasm-and-raw-webgl2-4360)** — Rust + WASM + WebGL2 的效能展示，萬架飛機 60fps，瀏覽器已經不是當年那個瀏覽器了
- **[Stop Losing Users to Slow Loads: A Developer's Guide to Core Web Vitals in 2026](https://dev.to/craftedmarketing/stop-losing-users-to-slow-loads-a-developers-guide-to-core-web-vitals-in-2026-4fb8)** — 2026 年了 CLS 和 INP 還是在搞人，這篇整理得算實用
- **[How We Moved Video Rendering From the Server to the Browser](https://dev.to/nareshipme/how-we-moved-video-rendering-from-the-server-to-the-browser-3p3d)** — 又一個把影片渲染搬到 client 端的案例，用 Web Workers 省掉 FFmpeg server 成本
- **[Bazel For a Frontend Monorepo](https://dev.to/mbarzeev/bazel-for-a-frontend-monorepo-da6)** — 拿 Bazel 來管前端 monorepo 的建置——學習曲線陡但 cache 命中率真香
- **[I stopped copy-pasting my lint rules into 12 different config files](https://dev.to/whitehatd/i-stopped-copy-pasting-my-lint-rules-into-12-different-config-files-13b2)** — 一份規則定義散佈在 CI、pre-commit、.cursorrules 等 12 個地方，這工具統一管理
- **[Beyond Prompts: How Git Hooks Steer AI Coding Agents in Production](https://dev.to/98lenvi/beyond-prompts-how-git-hooks-steer-ai-coding-agents-in-production-4pf9)** — 靠 instruction file 管不住 AI agent？用 Git hooks 加一層確定性的護欄
- **[I built a CLI tool that auto-detects missing environment variables — no schema needed](https://dev.to/akashguptasky/i-built-a-cli-tool-that-auto-detects-missing-environment-variables-no-schema-needed-gpn)** — 掃 code 裡的 `process.env.*` 自動檢查 .env 有沒有漏——簡單但實用
- **[Your Vulnerability Scanner Was the Vulnerability: 4 Projects Backdoored in 8 Days](https://dev.to/gabrielanhaia/your-vulnerability-scanner-was-the-vulnerability-4-projects-backdoored-in-8-days-46kd)** — Trivy 自己被植入後門，掃描器變成攻擊向量，供應鏈安全沒有終點

---

## 📚 慢慢啃

- **[Playwright vs Selenium in 2026: The Ultimate Guide for Modern Test Automation](https://dev.to/ankitaloni369/playwright-vs-selenium-in-2026-the-ultimate-guide-for-modern-test-automation-1bc6)** — 如果你還在 Selenium 跟 Playwright 之間猶豫，這篇從架構到 flaky test 處理都比較了，值得週末花 20 分鐘看完做決定
- **[Build an Autocomplete Search Bar with React](https://dev.to/rahulxsingh/build-an-autocomplete-search-bar-with-react-1kn5)** — 從 debounce 到 keyboard navigation 到 accessibility，autocomplete 該注意的細節都在這裡
- **[CSR vs SSR vs SSG vs ISR](https://dev.to/moosakhan/csr-vs-ssr-vs-ssg-vs-isr-1adn)** — 用餐廳比喻把四種渲染策略講明白——新手友善，但裡面的 trade-off 分析老手也值得複習
- **[How We Scaled Quran.com to 50M Monthly Users: Architecture Lessons From the Inside](https://dev.to/mzunain/how-we-scaled-qurancom-to-50m-monthly-users-architecture-lessons-from-the-inside-i33)** — 五千萬月活的前端架構實戰，CDN 策略和 cache 分層的經驗談比教科書好看
