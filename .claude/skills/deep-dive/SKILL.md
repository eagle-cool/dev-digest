---
name: deep-dive
description: 按預定主題清單，撰寫前端框架核心解析的深度專題報導（3000-5000 字）。發布至 GitHub Pages 並通知 Discord。
argument-hint: "[topic keyword or 'next' for auto-select]"
disable-model-invocation: false
user-invocable: true
allowed-tools: Task, WebFetch, WebSearch, Read, Write, Bash(date*), Bash(ls*), Bash(node *), Bash(test *), Bash(docker *), Bash(git *)
---

# Deep Dive — 主題式前端框架核心解析 → GitHub Pages

> **Strategy**: 預定主題清單 → Agent 研究該主題 → WebSearch/WebFetch 深度調研 → 3000-5000 字核心解析
> **Publish**: GitHub Pages (Jekyll) + Discord notification
> **Language**: Traditional Chinese (繁體中文) — 術語保留英文
> **Trigger**: 手動觸發，可指定主題或自動選下一個

## Core Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     Main Agent (Orchestrator)                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│   │ Phase 0  │ →  │ Phase 1  │ →  │ Phase 2  │ →  │ Phase 3  │  │
│   │ Load     │    │ Select   │    │ Deep     │    │ Write    │  │
│   │ SOUL     │    │ Topic    │    │ Research │    │ Article  │  │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │                │                │                │       │
│       ▼                ▼                ▼                ▼       │
│   SOUL.md 人設    從主題清單挑選    WebSearch +       3000-5000 字 │
│                   或接收指定主題    WebFetch 深度     核心解析專題   │
│                                    研究 (5+ 次)                   │
│                                                                   │
│   ┌──────────┐    ┌──────────┐                                   │
│   │ Phase 4  │ →  │ Phase 5  │                                   │
│   │ Publish  │    │ Discord  │                                   │
│   │ Pages    │    │ Notify   │                                   │
│   └──────────┘    └──────────┘                                   │
│       │                │                                          │
│       ▼                ▼                                          │
│   git push →       Send URL +                                    │
│   GitHub Pages     summary                                       │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## Topic Registry — 主題清單

以下是預定的深度解析主題。每個主題都是「不追新、但值得深挖」的框架核心知識。

### React 核心系列

| # | Topic | Slug Prefix | 核心問題 |
|---|-------|-------------|----------|
| R1 | Fiber 架構與 Reconciliation | react-fiber-reconciliation | React 為什麼要從 Stack 換成 Fiber？Reconciliation 到底怎麼運作？ |
| R2 | Hooks 的底層鏈表機制 | react-hooks-internals | 為什麼 Hooks 有使用規則限制？底層的鏈表結構長什麼樣？ |
| R3 | Concurrent Mode 排程器 | react-concurrent-scheduler | React 的 Lane 模型和時間切片怎麼做到不阻塞 UI？ |
| R4 | Server Components 序列化協議 | react-server-components-protocol | RSC 的 wire format 長什麼樣？Client 和 Server 之間怎麼通訊？ |
| R5 | React Compiler (React Forget) | react-compiler-auto-memoization | React Compiler 如何自動推斷 memoization？它的 IR 和靜態分析怎麼做的？ |
| R6 | Virtual DOM 的 Diff 算法演進 | react-vdom-diff-evolution | 從 O(n³) 到 O(n) — React 的 diff 做了哪些取捨？和 Svelte 的 no-VDOM 比呢？ |
| R7 | Suspense 與 Streaming SSR | react-suspense-streaming-ssr | Suspense 不只是 loading spinner — 它如何改變 SSR 架構？ |
| R8 | useEffect 的完整生命週期 | react-useeffect-lifecycle | cleanup、dependency array、strict mode double-invoke — 你真的理解 useEffect 嗎？ |
| R9 | Context 的效能陷阱與替代方案 | react-context-performance | 為什麼 Context 會導致不必要的 re-render？信號（Signals）是答案嗎？ |
| R10 | React 錯誤邊界與恢復機制 | react-error-boundaries | Error Boundary 怎麼攔截錯誤？為什麼只能用 class component？ |

### Next.js 核心系列

| # | Topic | Slug Prefix | 核心問題 |
|---|-------|-------------|----------|
| N1 | App Router 路由解析機制 | nextjs-app-router-internals | 檔案系統路由怎麼編譯成路由表？Layout 嵌套的實現原理？ |
| N2 | RSC 在 Next.js 的完整資料流 | nextjs-rsc-data-flow | 從 Server Component render 到 Client hydration，資料怎麼流動？ |
| N3 | 靜態 vs 動態渲染的決策樹 | nextjs-static-dynamic-rendering | Next.js 怎麼決定哪些頁面靜態生成、哪些動態渲染？force-dynamic 的底層機制？ |
| N4 | Middleware 與 Edge Runtime | nextjs-middleware-edge-runtime | Middleware 跑在哪裡？Edge Runtime 的限制是什麼？跟 Node.js Runtime 有何不同？ |
| N5 | 快取策略全解 | nextjs-caching-strategy | Data Cache、Full Route Cache、Router Cache — Next.js 四層快取怎麼協作？ |
| N6 | Server Actions 的實現原理 | nextjs-server-actions-internals | Server Actions 怎麼從 form 提交變成 RPC 調用？安全性怎麼保證？ |
| N7 | Turbopack 架構與增量編譯 | nextjs-turbopack-architecture | 為什麼 Turbopack 比 webpack 快？增量編譯和持久化快取怎麼做的？ |

### TypeScript 深度系列

| # | Topic | Slug Prefix | 核心問題 |
|---|-------|-------------|----------|
| T1 | 型別系統的結構化子型別 | typescript-structural-typing | 鴨子型別 vs 名義型別 — TS 的選擇帶來什麼好處和陷阱？ |
| T2 | 條件型別與 infer 的威力 | typescript-conditional-infer | 條件型別怎麼做到型別層級的模式匹配？實用案例拆解 |
| T3 | Template Literal Types | typescript-template-literal-types | 字串型別也能做型別運算？從 API route 型別到 CSS-in-TS |
| T4 | 協變與逆變 | typescript-variance | 為什麼函式參數是逆變的？React 元件 props 的型別安全怎麼保證？ |

### CSS 與 Web 平台系列

| # | Topic | Slug Prefix | 核心問題 |
|---|-------|-------------|----------|
| W1 | CSS Container Queries 完全攻略 | css-container-queries | 從 media query 到 container query — 響應式設計的典範轉移 |
| W2 | View Transitions API | view-transitions-api | 瀏覽器原生頁面轉場 — 跨文件動畫終於不用 JavaScript 了？ |
| W3 | CSS Cascade Layers (@layer) | css-cascade-layers | 如何用 @layer 解決 CSS 優先權地獄？對元件庫的影響？ |
| W4 | Web Components 2026 現狀 | web-components-2026-status | Declarative Shadow DOM、CSS Parts — WC 終於成熟了嗎？ |

### Node.js 與後端系列

| # | Topic | Slug Prefix | 核心問題 |
|---|-------|-------------|----------|
| B1 | Node.js Event Loop 深度解析 | nodejs-event-loop-deep-dive | 微任務、宏任務、I/O polling — 事件循環的六個階段真正怎麼運作？ |
| B2 | NestJS 依賴注入容器 | nestjs-dependency-injection | NestJS 的 DI 容器怎麼實現的？Reflect.metadata 和裝飾器的角色？ |
| B3 | Node.js Streams 與背壓 | nodejs-streams-backpressure | 為什麼大檔案要用 Stream？背壓機制怎麼防止記憶體爆炸？ |

### 前端工程化系列

| # | Topic | Slug Prefix | 核心問題 |
|---|-------|-------------|----------|
| E1 | Vite 的 HMR 與模組圖 | vite-hmr-module-graph | Vite 的 HMR 為什麼比 webpack 快？模組圖怎麼做到精準更新？ |
| E2 | Monorepo 工具鏈比較 | monorepo-toolchain-comparison | Turborepo vs Nx vs pnpm workspace — 各自的快取和任務排程策略 |
| E3 | Tree Shaking 的真實原理 | tree-shaking-deep-analysis | ESM 的靜態分析怎麼做到死碼消除？side effects 為什麼是陷阱？ |

## Execution Process

### Phase 0: Load SOUL

Before starting any phase, **read `SOUL.md`** from the project root to load the author persona.

```yaml
Steps:
  1. Read SOUL.md from the project root (dev-digest/SOUL.md)
  2. Internalize the persona: identity, writing style, tone, attitude, dos & don'ts
  3. ALL written content must reflect the SOUL.md persona throughout
  4. The persona nickname must NEVER appear in published content
```

### Phase 1: Select Topic — 選定主題

```yaml
Input: User argument (topic keyword, topic ID like "R1", or "next")

Case A — User specifies a topic:
  - Match against Topic Registry by keyword or ID (e.g., "R1", "fiber", "hooks")
  - If no match: treat as a custom topic (user can request any frontend/React/Node topic)
  - Confirm the topic and research angle

Case B — User says "next" or provides no argument:
  1. List existing deep-dive posts:
     ls _posts/*deep-dive* or grep for 'deep-dive' tag in _posts/
  2. Cross-reference with Topic Registry to find which topics are already covered
  3. Pick the next uncovered topic in order (R1 → R2 → ... → N1 → N2 → ...)
  4. Announce the selected topic before proceeding

Output:
  - selected_topic: { id, title, slug_prefix, core_question }
  - angle: 預計切入的角度和大綱方向（內部用）
```

### Phase 2: Deep Research — 深度調研

這是主題式 deep-dive 的核心：**Agent 必須透過大量研究來構建內容**，不是從一篇文章延伸，而是從零開始研究一個技術主題。

```yaml
Research Strategy:
  1. 先搜尋該主題的官方文件和原始資料
     - React: react.dev 文件、GitHub RFC、原始 PR
     - Next.js: nextjs.org 文件、Vercel 工程 blog
     - TypeScript: typescriptlang.org handbook、GitHub issues
     - Node.js: nodejs.org 文件、libuv 原始碼相關文章
  2. 搜尋社群中該主題的優質深度文章
     - 知名工程師的 blog post
     - Conference talk 文字稿或摘要
     - GitHub 上的 discussion 和 issue
  3. 搜尋反面觀點和常見誤解
     - "X is considered harmful" 類型的文章
     - Stack Overflow 上的常見錯誤
     - 效能陷阱或反模式

Research Rules:
  - MUST: 至少用 WebSearch 搜尋 5 次不同的查詢角度
  - MUST: 至少用 WebFetch 閱讀 3 篇深度參考文章
  - MUST: 研究結果必須實質性地構成最終文章內容
  - MUST: 包含原始碼級別的分析（看框架原始碼怎麼實現的）
  - MUST NOT: 編造不存在的數據、API、或原始碼
  - MUST NOT: 只搜尋確認自己觀點的資料（要找正反兩面）
  - SHOULD: 優先找一手資料（RFC, PR, 官方 blog, 原始碼）而非二手轉述
  - SHOULD: 找到具體的程式碼範例來說明概念

Output:
  - references: [{ title, url, key_insight }] — 實際引用的參考資料
  - source_code_findings: 從框架原始碼中發現的關鍵實現細節
  - common_misconceptions: 該主題常見的誤解
  - code_examples: 用來說明概念的程式碼片段
```

### Phase 3: Write Article — 撰寫核心解析專題

```yaml
Article Requirements:
  Length: 3000-5000 字（繁體中文）
  Tone: 遵循 SOUL.md 人設 — 資深工程師帶你看框架原始碼
  Nature: 不追新，深挖基礎。目標是讓讀者讀完後對框架的「為什麼」有深刻理解。

Structure (核心解析文章的通用結構):

  開場 (2-3 段):
    - 從一個常見的實際問題或誤解切入
    - 例：「你有沒有想過，為什麼 React 規定 Hooks 不能放在 if 裡面？」
    - 讓讀者知道今天要解開什麼謎團、為什麼他該在乎

  背景/脈絡 (1 個 H2):
    - 這個機制存在的歷史原因
    - 它解決了什麼問題
    - 在它之前是怎麼做的

  核心機制解析 (1-2 個 H2):
    - 深入原始碼級別的實現分析
    - 配合程式碼片段和圖表（用 ASCII 或 markdown）
    - 解釋關鍵的資料結構和演算法
    - 用簡化的虛擬碼讓讀者能理解核心邏輯

  實際影響/應用 (1 個 H2):
    - 理解這個機制後，能怎麼寫出更好的程式碼
    - 常見的陷阱和反模式
    - 效能考量

  結論 (1 段):
    - 有立場的總結
    - 對讀者的具體建議
    - 不要「讓我們拭目以待」那種廢話結尾

  延伸閱讀:
    - 文末附上參考資料

SEO Title (繁體中文):
  - 直接、有料、不標題黨
  - 包含核心技術名詞
  - 20-50 字
  - 例：「React Fiber 架構深度解析：從 Stack 到鏈表的 Reconciliation 革命」
  - 例：「你真的懂 useEffect 嗎？React Effect 的完整生命週期拆解」
  - 例：「Next.js 的四層快取機制：從 Request 到 Full Route 完全攻略」

Slug Generation:
  - Use the slug_prefix from Topic Registry, optionally extended
  - Lowercase English, hyphen-separated, max 8 words
  - Examples: "react-fiber-reconciliation-deep-dive"

Tags:
  - MUST include: deep-dive (用來區分日報)
  - Plus relevant tech tags from: react, nextjs, typescript, css, nodejs, frontend, web-platform, tooling

Jekyll Front Matter:
  ---
  title: "<SEO Title>"
  date: YYYY-MM-DD
  description: "<80-120 字繁中摘要，適合 Google 搜尋結果片段，要有具體技術內容>"
  tags: [deep-dive, react, ...]
  ---

Content Rules:
  - NO H1 title — Jekyll front matter `title` already renders as the page heading
  - 程式碼片段是必要的 — 這是核心解析，要有 code
  - 適當使用 ASCII 圖表解釋資料結構和流程
  - NO mention of RSS, Miniflux, feeds, or aggregation
  - NO "execution mode" or internal tooling references
  - 語氣一致：全文維持 SOUL.md 的資深工程師口吻
  - 適度使用簡化的虛擬碼（pseudocode）讓複雜概念更易懂

Reference Section (文末):
  ## 延伸閱讀

  - [參考文章標題](URL) — 一句話說明為什麼值得讀
  - [官方文件](URL) — 對應的官方資料
  ...

Output:
  - Directory: _posts/ (write directly to Jekyll posts directory)
  - Filename: YYYY-MM-DD-<slug>.md
  - Format: Standard Markdown with Jekyll front matter
```

### Phase 4: Publish to GitHub Pages

```yaml
GitHub Pages URL base: https://eagle-cool.github.io/dev-digest

Steps:
  1. Pull, commit, and push:
     git add -A && git commit -m "deep-dive: <slug>" && git pull --rebase && git push
  2. Construct page URL:
     https://eagle-cool.github.io/dev-digest/posts/<slug>/
  3. Wait 30 seconds for GitHub Pages to build, then verify once with WebFetch:
     - If live: proceed to Phase 5
     - If not live: proceed to Phase 5 anyway
```

### Phase 5: Discord Notification

```yaml
Steps:
  1. Check .env exists (if not: skip, inform user)
  2. Compose message (繁體中文, max 1800 chars):
     - Article title + one-liner hook
     - 3 key takeaways from the deep dive
     - Page URL
  3. Send text message:
     node ~/.claude/skills/discord-send/send.mjs dev-digest --text "<message>"
  4. Best-effort: failure does not affect article
```

**Discord message template:**

```
🔍 深度專題：<SEO Title>

<一句話 hook — 為什麼你該讀這篇>

重點摘要：
• <Takeaway 1>
• <Takeaway 2>
• <Takeaway 3>

📎 <URL>
```

## Output Template

```markdown
---
title: "<SEO Title>"
date: YYYY-MM-DD
description: "<80-120 字摘要，具體提到技術名詞和關鍵洞察>"
tags: [deep-dive, react]
---

<開場：從一個常見問題或誤解切入。「你有沒有想過為什麼...？」讓讀者好奇。>

---

## <背景/脈絡標題>

<這個機制的歷史背景和存在意義。300-500 字。>

## <核心機制解析標題>

<深入原始碼級別的分析。配合程式碼：>

```javascript
// 簡化的框架內部實現
// 讓讀者看到「原來底層是這樣做的」
```

<對程式碼的逐行解釋和分析>

## <實際影響/應用標題>

<理解機制後的實用建議、陷阱、反模式>

```javascript
// 常見錯誤示範
// vs 正確寫法
```

## <結論標題>

<有立場的總結 + 對讀者的具體建議。明確說「你現在該做/不該做什麼」。>

---

## 延伸閱讀

- [參考文章](URL) — 一句話說明
- [官方文件](URL) — 對應的官方資料
- [原始碼連結](URL) — 文中提到的原始碼位置
```

## Constraints & Principles

1. **Topic-Driven, Not News-Driven**: 從預定主題清單選題，不追新聞。目標是寫出「不會過時」的核心知識。
2. **Source Code Level**: 盡可能深入到框架原始碼級別。讀者看完要有「原來底層是這樣」的收穫。
3. **Research is Mandatory**: 必須做深度研究（WebSearch + WebFetch）。至少 5 次搜尋、3 篇深度參考。
4. **Code is Mandatory**: 核心解析文章一定要有程式碼。簡化的虛擬碼、實際框架原始碼片段、正反示範。
5. **One Topic, Full Depth**: 一次只解析一個主題，但要寫出 3000-5000 字的有料分析。
6. **Facts Over Opinion**: 觀點要有，但必須建立在原始碼和文件之上。不能編造 API 或實現細節。
7. **SEO-Quality Output**: 標題、描述、slug 都要 SEO 友善。
8. **No Implementation Leaks**: 不提 RSS, Miniflux, feeds, 或任何內部工具。
9. **deep-dive Tag**: 每篇文章必須包含 `deep-dive` tag 以區分日報。
10. **SOUL.md Persona**: 全文維持打碼老濕的人設，從開場到結尾。
11. **Progressive Difficulty**: 系列文章按順序從基礎到進階，但每篇都要能獨立閱讀。
12. **Reference Section**: 文末必須附上延伸閱讀和官方資料連結。

## Error Handling

| Error | Handling |
|-------|---------|
| Topic already covered | Show existing post, ask user to pick another topic |
| WebSearch returns poor results | Try alternative queries, use available info |
| WebFetch fails for key source | Try alternative sources, note limitation |
| git push fails | Report error, article still saved locally in _posts/ |
| Discord send fails | Log error, do not affect article |
