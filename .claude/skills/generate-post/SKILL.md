---
name: generate-post
description: Generates daily developer news digests from Miniflux RSS feeds (Dev category). Publishes to GitHub Pages and notifies via Discord.
argument-hint: "[optional: date]"
disable-model-invocation: false
user-invocable: true
allowed-tools: Task, WebFetch, Read, Write, Bash(date*), Bash(ls*), Bash(node *), Bash(test *), Bash(docker *), Bash(git *)
---

# Dev Digest — RSS-First Developer News → GitHub Pages

> **Source**: Miniflux RSS aggregator (Dev category, ~74 feeds)
> **Strategy**: RSS for bulk retrieval, Agent for quality evaluation, WebFetch only for deep-reading
> **Publish**: GitHub Pages (Jekyll) + Discord notification
> **Language**: Traditional Chinese (繁體中文) - titles keep original English, summaries in Traditional Chinese

## Core Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     Main Agent (Orchestrator)                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│   │ Phase 1  │ →  │ Phase 2  │ →  │ Phase 3  │ →  │ Phase 4  │  │
│   │ Fetch    │    │ Filter & │    │ Deep     │    │ Generate │  │
│   │ RSS Data │    │ Evaluate │    │ Read     │    │ Report   │  │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │                │                │                │       │
│       ▼                ▼                ▼                ▼       │
│   Miniflux API    Agent scores    WebFetch top 3    Markdown     │
│   → Dev category   title/desc    articles only     + slug       │
│     (1 API call)   (no fetch)     (optional)                     │
│                                                                   │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│   │ Phase 5  │ →  │ Phase 6  │ →  │ Phase 7  │                  │
│   │ Mark     │    │ Publish  │    │ Discord  │                  │
│   │ Read     │    │ Pages    │    │ Notify   │                  │
│   └──────────┘    └──────────┘    └──────────┘                  │
│       │                │                │                        │
│       ▼                ▼                ▼                        │
│   Miniflux API    git push →      Send URL +                    │
│   mark as read    GitHub Pages    summary text                   │
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
  3. ALL written content (editorial intro, summaries, key points, why it matters, Discord message)
     must reflect the SOUL.md persona throughout
  4. The persona nickname must NEVER appear in published content
```

## Execution Process

### Phase 1: Fetch RSS Data

Retrieve unread entries from Miniflux **Dev category only** (category_id = 4).

```yaml
Steps:
  0. Pre-flight: ensure Miniflux is running
     docker ps --filter name=miniflux --format '{{.Names}}' | grep -q miniflux
     - If NOT running: inform user to start Miniflux first, then verify with healthcheck:
       node ~/.claude/skills/miniflux/client.mjs healthcheck
  1. Determine target date (user argument or today via `date +%Y-%m-%d`)
  2. Calculate "after" timestamp (3 days before target date):
     # Use: date -v-3d +%s (macOS) or date -d '3 days ago' +%s (Linux)
  4. Fetch unread entries from Miniflux WITH category and time filter:
     node ~/.claude/skills/miniflux/client.mjs entries --status unread --limit 200 --category 4 --after <unix_timestamp> --direction desc
     → IMPORTANT: Always use --category 4 to filter Dev feeds only
     → IMPORTANT: Always use --after to avoid pulling old imported articles
     → Returns JSON: { total, count, entries: [{ id, title, url, feed, published, content_preview }] }
  5. If total == 0, check if Miniflux is healthy:
     node ~/.claude/skills/miniflux/client.mjs healthcheck
     - If unhealthy: report error and stop
     - If healthy but 0 entries: report "no new dev content today"
```

### Phase 2: Filter & Evaluate (Agent — metadata only)

The Agent evaluates entries based on title + content_preview (RSS description). No web fetching needed.

```yaml
Input: Array of entries from Phase 1 (title, url, feed, content_preview)

Agent Evaluation Criteria:
  Include (Dev-focused topics):
    - Frontend: React, Next.js, Vue, Svelte, Web standards, CSS, TypeScript, build tools, DX
    - Systems: Rust, Go, C/C++, performance engineering, compilers, OS internals
    - Security: Vulnerability research, exploit techniques, privacy, InfoSec analysis
    - DevOps: Infrastructure, containers, cloud services, CI/CD, monitoring
    - Open Source: Major releases, community news, licensing, developer tools
    - Programming: Language design, algorithms, data structures, software engineering essays
    - Retro Computing: Hardware hacking, vintage systems, reverse engineering
  Exclude:
    - AI/ML content (covered by AI Digest)
    - Marketing puff / PR without substance
    - Crypto/Blockchain
    - Job postings / hiring announcements
    - Non-tech content (politics, sports, etc.)
    - Content older than 3 days

Deduplication:
  - Handled by Miniflux read/unread status (selected articles are marked read after report)
  - Title similarity check (>80% = duplicate)

Scoring (1-5):
  - 5: Major release, breaking news, groundbreaking research
  - 4: High-quality technical deep-dive, significant update
  - 3: Interesting content, useful tutorial or analysis
  - 2: Okay content, niche interest
  - 1: Low quality or barely relevant

Output: Tiered selection for the digest:
  Tier 1 — 🔧 今日硬菜 (3 articles, score 4-5):
    - The day's meatiest dev stories, will get deep-read + senior engineer analysis
  Tier 2 — ⚡ 一句話帶過 (8-12 articles, score 3-4):
    - Worth knowing, one-liner with attitude
  Tier 3 — 📚 慢慢啃 (3-5 articles, score 3+):
    - Long-form essays, tutorials, or deep-dives worth your weekend time

  Each entry includes:
    - title (original English)
    - url
    - feed_name
    - tier: 1 | 2 | 3
    - quality_score: 1-5
    - summary_hint: 1-sentence description from content_preview (繁體中文)
    - entry_id: Miniflux entry ID (for marking as read later)
```

### Phase 3: Deep Read (Tier 1 articles only)

Only the 3 Tier 1 (🔥 今日焦點) articles get deep-read for rich summaries.

```yaml
Strategy:
  - Deep-read ALL Tier 1 articles (3 articles)
  - Tier 2 and Tier 3 use content_preview only — no fetching
  - Maximum 3 deep-reads per run (cost control)

Method 1 — Miniflux fetch-content (preferred, zero-cost):
  node ~/.claude/skills/miniflux/client.mjs fetch-content <entry_id>
  → Returns original article HTML content fetched by Miniflux

Method 2 — WebFetch fallback (if Miniflux fetch-content returns empty):
  Use WebFetch to read the article URL directly
  → Agent extracts key information from the page

For each Tier 1 article, produce:
  - summary: 3-4 sentence analysis (繁體中文), written like a senior engineer reviewing a PR.
    Focus on: what problem does this solve? what's the trade-off? who should care?
    Drop experience-based context where natural (「這讓我想到當年...」「踩過這個坑的人都知道...」).
    Dry humor welcome. Marketing-speak forbidden.
  - key_points: 2-3 key takeaways (繁體中文), engineer-precise, no fluff
  - trade_off: 1 sentence on the catch / what they're not telling you (繁體中文)

For Tier 2 articles: 1 sentence with personality — can be blunt, sarcastic, or genuinely appreciative. Not a boring title rewrite.
For Tier 3 articles: 1 sentence on what you'll actually learn if you invest the time — be specific.
```

### Phase 4: Generate Report + Slug + SEO Title

```yaml
SEO Title Generation (繁體中文):
  - Summarize the day's dev news like a senior engineer would — direct, no marketing fluff
  - Must be specific and informative, NOT generic like "每日開發者日報"
  - Include key product names, technologies, or concepts from top articles
  - Tone: matter-of-fact with a hint of attitude, like a Slack status update
  - 20-40 characters, no date in title (date is in front matter)
  - Examples:
    - "React 19 終於出了、Rust 正式進 Linux 核心、Bun 2.0 快得離譜"
    - "CSS Nesting 全面支援、Go 泛型補完計畫、又一個 Log4j 漏洞"
    - "WWDC 對開發者到底有啥用、Web Components 回來了、Cloudflare 又搞事"

Slug Generation:
  - Based on the top 2-3 dev highlights of the day
  - Format: lowercase English, hyphen-separated, max 6 words
  - Examples:
    - "react-19-rust-linux-bun-2"
    - "css-nesting-go-generics-log4j"
    - "wwdc-web-components-cloudflare"
  - The slug is used for both the filename and the Jekyll URL

Tags (only use relevant ones from this set):
  - frontend, systems, security, devops, opensource, programming, retro

Output:
  - Directory: _posts/  (write directly to Jekyll posts directory)
  - Filename: YYYY-MM-DD-<slug>.md
  - Format: Standard Markdown with Jekyll front matter
  - Language: Traditional Chinese (titles keep original English)

Jekyll Front Matter (MUST be included at top of file):
  ---
  title: "<SEO Title>"
  date: YYYY-MM-DD
  description: "<50-80 字繁中摘要，適合 Google 搜尋結果片段>"
  tags: [frontend, systems, ...]  # only include relevant categories
  ---

Content Structure (after front matter):
  - NO H1 title — Jekyll front matter `title` already renders as the page heading
  - Editorial intro (2-3 sentences, 繁中): like a senior dev summarizing today's standup.
    Direct, opinionated, no filler. Can open with a blunt observation or a dry joke.
    Example tone: "今天值得聊的就三件事。React 又改 API 了（驚不驚喜），有個 Rust 庫快得不講道理，然後又有人被 CORS 搞到懷疑人生。"
  - 🔧 今日硬菜 (3 articles): deep technical analysis, engineer-to-engineer
  - ⚡ 一句話帶過 (8-12 articles): one-liner each, blunt and scannable
  - 📚 慢慢啃 (3-5 articles): long-form worth your weekend
  - NO mention of RSS, Miniflux, feeds, or aggregation in published content
  - NO "execution mode" or "version" info — this is editorial content, not a system report
```

### Phase 5: Mark Read

```yaml
Steps:
  1. Mark all selected entries as read in Miniflux:
     node ~/.claude/skills/miniflux/client.mjs mark-read <id1,id2,id3,...>
```

### Phase 6: Publish to GitHub Pages

This skill runs inside the dev-digest Jekyll repo. Posts are written directly to `_posts/`.

```yaml
GitHub Pages URL base: https://eagle-cool.github.io/dev-digest

Steps:
  1. Pull, commit, and push (report is already in _posts/):
     git add -A && git commit -m "YYYY-MM-DD: <slug>" && git pull --rebase && git push
  2. Construct page URL:
     https://eagle-cool.github.io/dev-digest/posts/<slug>/
  5. Wait 30 seconds for GitHub Pages to build, then verify once with WebFetch:
     - If live: proceed to Phase 7
     - If not live: proceed to Phase 7 anyway (typically builds within 30s)
```

### Phase 7: Discord Notification

```yaml
Steps:
  1. Check .env exists (if not: skip, inform user)
  2. Compose message (繁體中文, max 1800 chars):
     - Page URL
     - Top 3 highlights + 3 notable items
  3. Send text message (no file attachment):
     node ~/.claude/skills/discord-send/send.mjs dev-digest --text "<message>"
  4. Best-effort: failure does not affect report
```

**Discord message template:**

```
🔧 <SEO Title>

今日硬菜
• [標題] — 一句話點評（帶態度）
• [標題] — 一句話點評
• [標題] — 一句話點評

⚡ 另有 N 則一句話帶過 + N 篇慢慢啃

📎 <URL>
```

## Output Template

```markdown
---
title: "<SEO Title>"
date: YYYY-MM-DD
description: "<50-80 字摘要，用老司機口吻，適合搜尋結果片段>"
tags: [frontend, systems]
---

今天值得聊的就這幾個...(2-3 句開場，像資深同事在茶水間跟你講。直接、有料、可以帶一句乾式幽默。不要「各位讀者大家好」那套。)

---

## 🔧 今日硬菜

### [Article Title](URL)

3-4 句技術分析，像在做 code review 一樣。重點是：解決了什麼問題？trade-off 是什麼？值不值得在你的 stack 裡用？可以帶入經驗（「踩過這個坑的都知道...」），吐槽但要有建設性。

**重點：**
- 技術重點一（精確，不含糊）
- 技術重點二
- 但是...（trade-off / 沒說的部分）

### [Article Title](URL)

（同上格式，共 3 篇硬菜）

---

## ⚡ 一句話帶過

- **[Article Title](URL)** — 直接講結論，可以毒舌可以讚賞，但要中肯
- **[Article Title](URL)** — 工程師看了會「嗯」一聲的那種摘要
- **[Article Title](URL)** — 不廢話
- ...（共 8-12 則）

---

## 📚 慢慢啃

- **[Article Title](URL)** — 具體說讀完能學到什麼，不是重複標題
- **[Article Title](URL)** — 為什麼這篇值得你週末花 20 分鐘
- ...（共 3-5 則）
```

## Constraints & Principles

1. **RSS-First**: Always use Miniflux API for data retrieval. Never scrape source websites for listing.
2. **Dev Category Only**: Always filter by `--category 4` to get Dev feeds only.
3. **Agent for Evaluation Only**: Agent reads title/description to score and filter. No web fetching for listing.
4. **Deep Read = Selective**: Only top 3 articles with score >= 4 get full content fetch. Use Miniflux `fetch-content` first, WebFetch as fallback.
5. **Cost Control**: Target < 10K agent tokens per run.
6. **Mark as Read**: Always mark selected entries as read in Miniflux after report generation.
7. **SEO-Quality Output**: Every published article must have a compelling title, meta description, and meaningful URL slug. Write in the voice defined by SOUL.md, not like a bot.
8. **No Implementation Leaks**: Published content must NEVER mention RSS, Miniflux, feeds, aggregation, or any internal tooling.
9. **Meaningful Slugs**: Generate descriptive English slugs from the day's top dev highlights for readable URLs.
10. **Tiered Structure**: 3 硬菜 + 8-12 一句話帶過 + 3-5 慢慢啃。硬菜要有技術深度、一句話要夠嗆、慢慢啃要讓人想收藏。
11. **Language Consistency**: Summaries in 繁體中文, titles and keywords in English.

## Error Handling

| Error | Handling |
|-------|---------|
| Miniflux unreachable | Run healthcheck, report error, stop |
| 0 unread entries | Report "no new dev content", skip report generation |
| fetch-content empty | Fallback to WebFetch for that article |
| WebFetch fails | Use content_preview only, note in report |
| git push fails | Report error, report still saved locally in _posts/ |
| GitHub Pages not live after 2 min | Send Discord with URL anyway (will be live shortly) |
| Discord send fails | Log error, do not affect report |
