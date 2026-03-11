---
title: "React Compiler 終結 useMemo、Node.js 原生吃 TypeScript、百萬行匯出不當機"
date: 2026-03-11
description: "React Compiler v1.0 自動處理 memoization、Node.js 原生 type stripping 讓你直接跑 .ts、DuckDB SQL 生成百萬行 XLSX 不炸瀏覽器。"
tags: [frontend, react, typescript, nodejs, tooling]
---

今天三道硬菜都跟「少寫廢 code」有關。React Compiler 讓你把 useMemo 和 useCallback 全部刪掉，Node.js 終於原生跑 TypeScript 不用裝一堆有的沒的，然後有人用 SQL 直接吐 XML 來生成百萬行 XLSX——因為 SheetJS 和 ExcelJS 全都撐不住。工程師的日常就是這樣，能用更簡單的方式解決問題，就別搞那些花裡胡哨的。

---

## 🔧 今日硬菜

### [The React Compiler Is Here: Say Goodbye to useMemo and useCallback](https://dev.to/alexcloudstar/the-react-compiler-is-here-say-goodbye-to-usememo-and-usecallback-436g)

React Compiler v1.0 去年十月就出了，但真正開始用的團隊還不多。這東西本質上就是個 build-time 的 Babel/SWC plugin，它在編譯階段分析你的 component，自動插入 memoization——你寫乾淨的程式碼，它幫你處理 `useMemo`、`useCallback`、`React.memo` 那些儀式性的東西。踩過 dependency array 寫錯導致 stale closure 的人都知道，手動 memoization 是個多容易出錯的活。

**重點：**
- 編譯器能看到整個 component 的依賴關係，比人類手動追蹤準確得多
- Next.js 一行 config 開啟，Vite 裝個 Babel plugin 就搞定
- 支援 `'use no memo'` directive，可以逐步遷移不用一次到位

**但是：** 它不是萬能的。虛擬列表、`useRef` 操控 DOM、一些老舊第三方庫的奇葩 pattern，它都不管。而且它嚴格執行 Rules of React——如果你的 codebase 有偷偷 mutate state 的壞習慣，compiler 會直接跳過或報錯。某種程度上，它也是個代碼品質檢查工具。

### [TypeScript Without a Build Step: Native Type Stripping in Node.js](https://dev.to/alexcloudstar/typescript-without-a-build-step-native-type-stripping-in-nodejs-3caf)

Node.js v22.6+ 內建 type stripping，直接 `node index.ts` 就能跑。沒有 flag，沒有 config，沒有額外依賴。它的原理很粗暴——把所有型別標註當註解移除，底下的 JavaScript 直接執行。這意味著 `ts-node`、`tsx` 這些工具在寫腳本的場景下可以正式退休了。

**重點：**
- 大部分 TS 語法都支援：interface、generics、`as` cast、`satisfies`、enum
- 開發時 `node --watch src/index.ts` 直接熱更新，爽度破表
- 型別檢查還是要靠 `tsc --noEmit`，它只負責「跑」不負責「查」

**但是：** `const enum` 不能用（因為需要編譯期展開）、legacy decorators 不支援、production build 還是建議走正規編譯流程。另外，這只是 strip 型別，不是跑 TypeScript compiler——你傳 string 給期望 number 的函式，它照跑不誤，runtime 才爆。

### [When JS Libraries Fail at 1M Rows: Generating XLSX via DuckDB SQL](https://dev.to/yuki0510/when-js-libraries-fail-at-1m-rows-generating-xlsx-via-duckdb-sql-2gkk)

這篇的思路很野：XLSX 本質上就是 ZIP 裡面包 XML，所以作者直接用 DuckDB-WASM 的 SQL 字串操作生成 XML，再用 JSZip 打包——跳過所有 JS 中間物件的轉換。結果？ExcelJS 處理百萬行要 20 秒還可能當掉瀏覽器，他的方案 XML 生成 5.5 秒、壓縮 7.3 秒，而且匯出完瀏覽器不卡。關鍵差別在於省掉了大量 GC-eligible 的中間物件。

**重點：**
- SheetJS 和 ExcelJS 在大資料量下記憶體爆炸是已知痛點，這方案直接繞過
- DuckDB SQL 的 `CONCAT` + `REPLACE` 處理字串出奇地快
- 靜態 XML 模板 + 動態資料行 = 最小化生成邏輯

**但是：** 不支援儲存格樣式（顏色、邊框、格式化），純資料匯出限定。如果你的需求是生成漂亮的報表，這方案不適合。但如果你只是要把資料倒出來，這比任何現有 JS 方案都快。

---

## ⚡ 一句話帶過

- **[How Hotwire Restored My Sanity in Web Development](https://dev.to/zilton7/how-hotwire-restored-my-sanity-in-web-development-1m2o)** — 又一個從 React 回歸 server-rendered 的懺悔錄，Hotwire 的 Turbo + Stimulus 確實讓某些場景回歸簡單
- **[How Core Web Vitals Impact SEO Rankings: What the Data Shows](https://dev.to/apogeewatcher/how-core-web-vitals-impact-seo-rankings-what-the-data-shows-31ce)** — CWV 到底影不影響排名？有數據的分析比猜測有用，值得 SEO 焦慮症患者一讀
- **[Stop Creating Useless Instances in TypeScript](https://dev.to/ramonsilvadev/stop-creating-useless-instances-in-typescript-3en0)** — 拜託別再 `new UtilityClass()` 了，靜態方法或純函式就夠了，TypeScript 不是 Java
- **[Dealing with flaky tests](https://dev.to/conw_y/dealing-with-flaky-tests-4d2e)** — 每個團隊都有那幾個「有時過有時不過」的測試，這篇講怎麼系統性地處理而不是假裝沒看到
- **[FCIS — Functional Core, Imperative Shell](https://dev.to/roalcantara/fcis-functional-core-imperative-shell-4haf)** — 把純函式邏輯和 I/O 副作用硬切兩層，聽起來理想，實作起來看團隊功力
- **[Process Manager Comparison 2026 — PM2, Systemd, Supervisor, Oxmgr](https://dev.to/empellio/process-manager-comparison-2026-pm2-systemd-supervisor-oxmgr-and-more-4e0p)** — Node.js 上 production 還在用 PM2 的舉手，這篇幫你看看 2026 年還有什麼選擇
- **[I built an AI-powered 3D showroom for e-commerce using Three.js](https://dev.to/mmsoftstudio/i-built-an-ai-powered-3d-showroom-for-e-commerce-using-threejs-4hf5)** — Three.js + AI 生成 3D 商品展示間，概念有趣但離 production 還有段路
- **[OpenTelemetry Production Error Debugging with Source Map Integration](https://dev.to/pavkode/opentelemetry-production-error-debugging-improved-with-source-map-integration-for-minified-stack-2dee)** — minified stack trace 看到眼睛脫窗的人有救了，OTel + source map 是正解
- **[5 Things AI Can't Do, Even in Vue.js](https://dev.to/devunionx/5-things-ai-cant-do-even-in-vuejs-16ok)** — AI 寫不好的五件事，不限於 Vue，基本上就是「需要理解脈絡」的活
- **[Mastering TypeScript's Advanced Types: Beyond the Basics](https://dev.to/midas126/mastering-typescripts-advanced-types-beyond-the-basics-1p8j)** — Conditional types、mapped types、template literal types 一次講完，適合想升級型別體操的人

---

## 📚 慢慢啃

- **[Claude Code vs Cursor vs GitHub Copilot: The 2026 AI Coding Tool Showdown](https://dev.to/alexcloudstar/claude-code-vs-cursor-vs-github-copilot-the-2026-ai-coding-tool-showdown-53n4)** — 三大 AI coding 工具的長期使用者寫的比較，不是跑個 benchmark 就下結論那種，適合正在猶豫該選哪個的你
- **[How to give your AI the eye of a designer](https://dev.to/utshull/how-to-give-your-ai-the-eye-of-a-designer-le0)** — 讓 AI 生成的 UI 不再像工程師做的，靠的不是更好的 prompt 而是餵給它設計系統的約束
- **[Why Fresh is the "Coolest" New Framework for Web Developers](https://dev.to/injamulcse15/why-fresh-is-the-coolest-new-framework-for-web-developers-15jd)** — Deno 生態的 Fresh 框架，island architecture + 零 JS by default，看看它跟 Astro 的定位差異
- **[I Built an Open-Source Prompt-to-Video Generator (React + TypeScript)](https://dev.to/sona_badalyan_90b4c977526/i-built-an-open-source-prompt-to-video-generator-react-typescript-4824)** — 從文字 prompt 生成短影片的開源專案，React + TS 全端實作，想了解 AI 影片生成 pipeline 的可以挖
