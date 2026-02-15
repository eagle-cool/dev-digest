---
name: deep-dive
description: 從 RSS 來源中挑選一篇最值得深度解析的前端文章，結合延伸研究撰寫 3000-5000 字的深度專題報導。發布至 GitHub Pages 並通知 Discord。
argument-hint: "[optional: entry_id or URL]"
disable-model-invocation: false
user-invocable: true
allowed-tools: Task, WebFetch, WebSearch, Read, Write, Bash(date*), Bash(ls*), Bash(node *), Bash(test *), Bash(docker *), Bash(git *)
---

# Deep Dive — RSS-Driven Frontend Deep Analysis → GitHub Pages

> **Source**: Miniflux RSS aggregator (Dev category, ~74 feeds)
> **Strategy**: RSS 選題 → Agent 挑出一篇最值得深入的前端文章 → WebSearch/WebFetch 延伸研究 → 3000-5000 字深度專題
> **Publish**: GitHub Pages (Jekyll) + Discord notification
> **Language**: Traditional Chinese (繁體中文) — 術語保留英文
> **Trigger**: 手動觸發，不定期

## Core Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     Main Agent (Orchestrator)                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│   │ Phase 1  │ →  │ Phase 2  │ →  │ Phase 3  │ →  │ Phase 4  │  │
│   │ Fetch    │    │ Select   │    │ Deep     │    │ Research │  │
│   │ RSS Data │    │ THE ONE  │    │ Read     │    │ & Expand │  │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │                │                │                │       │
│       ▼                ▼                ▼                ▼       │
│   Miniflux API    Agent picks 1    Full article     WebSearch +  │
│   → Dev category   best frontend   content via      WebFetch     │
│     (1 API call)   article         Miniflux/Web     延伸研究      │
│                                                                   │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│   │ Phase 5  │ →  │ Phase 6  │ →  │ Phase 7  │ →  │ Phase 8  │  │
│   │ Write    │    │ Mark     │    │ Publish  │    │ Discord  │  │
│   │ Article  │    │ Read     │    │ Pages    │    │ Notify   │  │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │                │                │                │       │
│       ▼                ▼                ▼                ▼       │
│   3000-5000 字     Miniflux API    git push →      Send URL +   │
│   深度專題          mark as read    GitHub Pages    summary       │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
         ↕ REST API (via client.mjs)
┌────────────────────┐
│   Miniflux         │  ← Dev category feeds (~74 sources)
│   (always running) │  ← Handles caching, dedup, parsing
└────────────────────┘
```

## Phase 0: Load SOUL

Before starting any phase, **read `SOUL.md`** from the project root to load the author persona.

```yaml
Steps:
  1. Read SOUL.md from the project root (dev-digest/SOUL.md)
  2. Internalize the persona: identity, writing style, tone, attitude, dos & don'ts
  3. ALL written content must reflect the SOUL.md persona throughout
  4. The persona nickname must NEVER appear in published content
```

## Execution Process

### Phase 1: Fetch RSS Data

Retrieve unread entries from Miniflux **Dev category only** (category_id = 4).

If user provides an entry_id or URL as argument, skip Phase 1 & 2 — jump directly to Phase 3 with that article.

```yaml
Steps:
  0. Pre-flight: ensure Miniflux is running
     docker ps --filter name=miniflux --format '{{.Names}}' | grep -q miniflux
     - If NOT running: inform user to start Miniflux first, then verify with healthcheck:
       node ~/.claude/skills/miniflux/client.mjs healthcheck
  1. Determine target date (today via `date +%Y-%m-%d`)
  2. Calculate "after" timestamp (5 days before target date for wider pool):
     # Use: date -v-5d +%s (macOS) or date -d '5 days ago' +%s (Linux)
  3. Fetch unread entries from Miniflux WITH category and time filter:
     node ~/.claude/skills/miniflux/client.mjs entries --status unread --limit 200 --category 4 --after <unix_timestamp> --direction desc
     → IMPORTANT: Always use --category 4 to filter Dev feeds only
     → IMPORTANT: Always use --after to avoid pulling old imported articles
     → Returns JSON: { total, count, entries: [{ id, title, url, feed, published, content_preview }] }
  4. If total == 0, check if Miniflux is healthy:
     node ~/.claude/skills/miniflux/client.mjs healthcheck
     - If unhealthy: report error and stop
     - If healthy but 0 entries: report "no unread dev content available"
```

### Phase 2: Select THE ONE — 前端優先選題

Agent 從所有 unread entries 中挑出**一篇**最值得深度解析的文章。

```yaml
Input: Array of entries from Phase 1 (title, url, feed, content_preview)

Selection Criteria — 什麼樣的文章值得 deep dive:

  Priority Focus (前端為主):
    - 新的 Web 標準 / 瀏覽器 API（如 View Transitions, Popover, CSS Anchor Positioning）
    - 主流框架重大更新或架構變革（React Server Components, Svelte 5, Vue Vapor）
    - 前端效能突破或新的最佳實踐
    - 建置工具革新（Vite, Turbopack, Rspack, Bun bundler）
    - TypeScript 重大特性或型別系統深度議題
    - CSS 新功能或佈局技術突破
    - Web Components / Web Platform 重要進展
    - 前端安全漏洞或供應鏈攻擊

  Also Considered (次要但仍可選):
    - Systems 技術對前端的影響（WASM, Edge Runtime, Rust-based tooling）
    - 重大開源專案的架構設計深度文章
    - 影響廣泛的安全漏洞分析

  NOT Suitable for Deep Dive:
    - AI/ML 內容
    - 簡單的 tutorial 或 how-to（太淺）
    - 新聞公告類（沒有技術深度可挖）
    - 已經被寫爛的老話題
    - Marketing / PR 文

  Ideal Article Characteristics:
    - 有技術深度可挖 — 不只是「XXX 發布了」而是「XXX 的架構設計有什麼巧思」
    - 有爭議性或多面向 — 值得從不同角度討論
    - 有延伸空間 — 可以連結到更大的技術趨勢或歷史脈絡
    - 讀者讀完會有「原來如此」的收穫
    - 能寫出 3000-5000 字的有料分析，不是硬湊

Output:
  - selected_entry: { id, title, url, feed, content_preview }
  - selection_reason: 2-3 句說明為什麼選這篇（內部用，不發布）
  - angle: 預計切入的角度和大綱方向（內部用）
```

### Phase 3: Deep Read — 完整閱讀選定文章

```yaml
Strategy:
  - 完整讀取選定文章的全文，不是摘要

Method 1 — Miniflux fetch-content (preferred, zero-cost):
  node ~/.claude/skills/miniflux/client.mjs fetch-content <entry_id>
  → Returns original article HTML content fetched by Miniflux

Method 2 — WebFetch fallback (if Miniflux fetch-content returns empty):
  Use WebFetch to read the article URL directly
  → Agent extracts key information from the page

After reading, produce internal notes:
  - core_thesis: 文章的核心論點是什麼
  - technical_details: 關鍵技術細節列表
  - questions: 讀完後有哪些問題需要延伸研究
  - connections: 這個話題和哪些更大的趨勢/歷史有關
  - research_queries: 需要搜尋的 3-5 個具體問題
```

### Phase 4: Research & Expand — 延伸研究

這是 deep-dive 與 generate-post 最大的差異：**Agent 必須主動做延伸研究**，不能只靠原文和通用知識。

```yaml
Research Strategy:
  1. 根據 Phase 3 的 research_queries，用 WebSearch 搜尋相關資料
  2. 從搜尋結果中用 WebFetch 抓取 3-5 篇最相關的參考文章
  3. 研究方向包括但不限於：
     - 官方文件 / RFC / 提案原文
     - GitHub issue / PR 討論中的第一手資訊
     - 其他工程師對同一話題的分析或反駁
     - 相關技術的歷史演進和前車之鑑
     - 效能數據、benchmarks、實測結果
     - 競爭方案或替代方案的比較

Research Rules:
  - MUST: 至少用 WebSearch 搜尋 3 次不同的查詢
  - MUST: 至少用 WebFetch 閱讀 2 篇延伸參考文章
  - MUST: 研究結果必須實質性地影響最終文章內容，不是裝飾
  - MUST NOT: 編造不存在的數據、引用、或 benchmark 結果
  - MUST NOT: 只搜尋確認自己觀點的資料（要找正反兩面）
  - SHOULD: 優先找一手資料（RFC, PR, 官方 blog）而非二手轉述

Output:
  - references: [{ title, url, key_insight }] — 實際引用的參考資料
  - additional_context: 延伸研究中發現的重要資訊
  - counter_arguments: 不同觀點或反對意見
```

### Phase 5: Write Article — 撰寫深度專題

```yaml
Article Requirements:
  Length: 3000-5000 字（繁體中文）
  Tone: 遵循 SOUL.md 人設 — 資深工程師的深度解讀
  Structure: 視主題而定（見下方結構選項）

Structure Options (Agent 根據主題自行選擇最適合的):

  Option A — 故事線型:
    原文新聞出發 → 背景脈絡 → 技術深潛 → 實際影響 → 結論與展望
    適合：重大發布、架構變革、新標準

  Option B — 問題拆解型:
    提出問題 → 現狀分析 → 方案比較 → 深入某方案 → 結論
    適合：「該用 A 還是 B」、「為什麼 X 取代了 Y」

  Option C — 技術解析 + 實作導向:
    這是什麼 → 為什麼需要 → 怎麼運作（附程式碼） → 實際應用 → 注意事項
    適合：新 API、新工具、新框架特性

  Option D — 趨勢分析型:
    現象觀察 → 歷史脈絡 → 各方觀點 → 技術分析 → 未來預測
    適合：「為什麼大家都在轉向 XXX」、生態系變化

SEO Title (繁體中文):
  - 直接、有料、不標題黨
  - 包含核心技術名詞
  - 20-50 字
  - 例：「React Server Components 深度解析：為什麼你的 SPA 該開始準備了」
  - 例：「View Transitions API 完全攻略：瀏覽器原生動畫終於能用了」
  - 例：「Bun 2.0 vs Vite vs Turbopack：2026 前端建置工具終極比較」

Slug Generation:
  - Based on the main topic, lowercase English, hyphen-separated
  - Max 8 words, descriptive
  - Examples:
    - "react-server-components-deep-analysis"
    - "view-transitions-api-complete-guide"
    - "bun-vite-turbopack-comparison-2026"

Tags:
  - MUST include: deep-dive (用來區分日報)
  - Plus relevant tech tags from: frontend, systems, security, devops, opensource, programming, retro

Jekyll Front Matter:
  ---
  title: "<SEO Title>"
  date: YYYY-MM-DD
  description: "<80-120 字繁中摘要，適合 Google 搜尋結果片段，要有具體技術內容>"
  tags: [deep-dive, frontend, ...]
  ---

Content Rules:
  - NO H1 title — Jekyll front matter `title` already renders as the page heading
  - 開場 (2-3 段): 從原文新聞/事件切入，迅速讓讀者知道「今天要聊什麼、為什麼重要」
  - 主體: 根據選定的結構展開，至少 3 個主要段落 (H2)
  - 結尾 (1-2 段): 給出有立場的結論和對讀者的具體建議
  - 參考連結: 文末附上原文和延伸閱讀，格式為 markdown 連結列表
  - 程式碼片段: 如果主題涉及具體 API 或實作，適當加入程式碼區塊
  - NO mention of RSS, Miniflux, feeds, or aggregation
  - NO "execution mode" or internal tooling references
  - 語氣一致：全文維持 SOUL.md 的資深工程師口吻

Reference Section (文末):
  ## 延伸閱讀

  - [原文標題](原文 URL) — 本文的起點
  - [參考文章標題](URL) — 一句話說明為什麼值得讀
  - [參考文章標題](URL) — 一句話說明
  ...

Output:
  - Directory: _posts/ (write directly to Jekyll posts directory)
  - Filename: YYYY-MM-DD-<slug>.md
  - Format: Standard Markdown with Jekyll front matter
```

### Phase 6: Mark Read

```yaml
Steps:
  1. Mark the selected entry as read in Miniflux:
     node ~/.claude/skills/miniflux/client.mjs mark-read <entry_id>
  2. Only mark the one article that was deep-dived, not all unread entries
```

### Phase 7: Publish to GitHub Pages

```yaml
GitHub Pages URL base: https://eagle-cool.github.io/dev-digest

Steps:
  1. Pull, commit, and push:
     git add -A && git commit -m "deep-dive: <slug>" && git pull --rebase && git push
  2. Construct page URL:
     https://eagle-cool.github.io/dev-digest/posts/<slug>/
  3. Wait 30 seconds for GitHub Pages to build, then verify once with WebFetch:
     - If live: proceed to Phase 8
     - If not live: proceed to Phase 8 anyway
```

### Phase 8: Discord Notification

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
tags: [deep-dive, frontend]
---

<開場：2-3 段，從原文事件切入。像資深工程師在技術分享會開場——先講結論，再說為什麼你該在乎。不要「今天我們來聊聊」那種廢話開頭。>

---

## <第一個主要段落標題>

<深度分析，至少 300-500 字。帶入技術細節、歷史脈絡、或問題背景。>

## <第二個主要段落標題>

<核心技術解析。如果涉及具體 API 或實作：>

```javascript
// 具體的程式碼範例（如適用）
const example = "show, don't just tell";
```

<對程式碼的解釋和分析>

## <第三個主要段落標題>

<延伸討論：影響、比較、或不同觀點。這裡要用到 Phase 4 的研究成果。>

## <結論段落標題（可以是「所以呢」「我的看法」等口語標題）>

<有立場的結論 + 對讀者的具體建議。不要「讓我們拭目以待」那種廢話結尾。要明確說「你現在該做/不該做什麼」。>

---

## 延伸閱讀

- [原文標題](URL) — 本文的起點
- [參考文章](URL) — 一句話為什麼值得讀
- [參考文章](URL) — 一句話
```

## Constraints & Principles

1. **RSS-First Selection**: 從 Miniflux 選題，不是憑空想題目。
2. **Frontend Priority**: 選題以前端技術為最高優先，但不排除影響前端的系統/安全議題。
3. **Research is Mandatory**: 必須做延伸研究（WebSearch + WebFetch），不能只靠原文 + 通用知識。至少 3 次搜尋、2 篇延伸閱讀。
4. **One Article, Full Depth**: 只選一篇，但要寫出 3000-5000 字的有料分析。寧可深不可淺。
5. **Facts Over Opinion**: 觀點要有，但必須建立在事實和研究之上。引用要準確，數據不能編。
6. **Mark Only Selected**: 只標記被深度分析的那一篇為已讀，不動其他 entries。
7. **SEO-Quality Output**: 標題、描述、slug 都要 SEO 友善。
8. **No Implementation Leaks**: 不提 RSS, Miniflux, feeds, 或任何內部工具。
9. **deep-dive Tag**: 每篇文章必須包含 `deep-dive` tag 以區分日報。
10. **SOUL.md Persona**: 全文維持打碼老濕的人設，從開場到結尾。
11. **Flexible Structure**: 不強制固定結構，根據主題選擇最適合的敘事方式。
12. **Reference Section**: 文末必須附上原文連結和延伸閱讀資料。

## Error Handling

| Error | Handling |
|-------|---------|
| Miniflux unreachable | Run healthcheck, report error, stop |
| 0 unread entries | Report "no unread dev content available", stop |
| No frontend-worthy article | Report "no article suitable for deep dive today", stop |
| fetch-content empty | Fallback to WebFetch for the article |
| WebFetch fails | Use content_preview, note limitation |
| WebSearch returns poor results | Try alternative queries, use available info |
| git push fails | Report error, article still saved locally in _posts/ |
| Discord send fails | Log error, do not affect article |
