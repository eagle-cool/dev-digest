---
title: "Next.js Middleware 與 Edge Runtime 完全拆解：從 V8 Isolate 到 proxy.ts 的架構革命"
date: 2026-03-09
description: "深入解析 Next.js Middleware 的執行機制、Edge Runtime 的 V8 Isolate 架構、為什麼 Edge 限制讓開發者抓狂，以及 Next.js 16 為何決定用 proxy.ts 取代 middleware.ts 並回歸 Node.js Runtime。"
tags: [deep-dive, nextjs, frontend]
---

你有沒有在 Next.js Middleware 裡面試過連資料庫？

如果試過，你應該看到了那個經典的錯誤訊息：`Native Node.js APIs are not supported in Edge Runtime`。然後你就開始想——這他媽的是什麼 Runtime？我明明在寫 JavaScript，為什麼連 `fs` 都不能用？

今天我們就來拆開這層黑盒子。從 Middleware 的請求攔截流程、Edge Runtime 底層的 V8 Isolate 架構，一路講到 Next.js 16 為什麼做出了「把 middleware.ts 改名為 proxy.ts 並切回 Node.js Runtime」這個重大決定。

---

## Middleware 到底在哪裡執行？請求生命週期全解

要理解 Middleware，先搞清楚一個 HTTP 請求進入 Next.js 後走過了哪些關卡。整個執行順序是這樣的：

```
HTTP Request 進入
      │
      ▼
┌─────────────────────────────┐
│ 1. next.config.js headers   │  ← 靜態 header 設定
├─────────────────────────────┤
│ 2. next.config.js redirects │  ← 靜態重定向規則
├─────────────────────────────┤
│ 3. Middleware (proxy.ts)    │  ← ★ 你的程式碼在這裡執行
├─────────────────────────────┤
│ 4. beforeFiles rewrites     │  ← next.config.js 中的 rewrites
├─────────────────────────────┤
│ 5. 檔案系統路由匹配          │  ← public/、_next/static/、pages/、app/
├─────────────────────────────┤
│ 6. afterFiles rewrites      │
├─────────────────────────────┤
│ 7. Dynamic Routes           │  ← /blog/[slug] 等動態路由
├─────────────────────────────┤
│ 8. fallback rewrites        │
└─────────────────────────────┘
      │
      ▼
   Response 回傳
```

重點在第三步。Middleware 跑在所有路由匹配**之前**——它不是 Express 那種可以掛在特定路由上的 middleware chain，而是一個全域的請求攔截器。整個 Next.js 應用只有一個 `middleware.ts`（或現在的 `proxy.ts`），放在專案根目錄。

這意味著每一個請求——包括靜態資源、圖片最佳化、API Route——都會經過 Middleware。所以你需要 `matcher` 來精準控制哪些路徑需要攔截：

```typescript
// proxy.ts
import { NextResponse, type NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  const token = request.cookies.get('session')?.value
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}

export const config = {
  matcher: [
    // 排除靜態資源、圖片、favicon 等
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
}
```

注意 `matcher` 必須是**編譯期常量**。你不能用變數、不能動態生成——Next.js 在 build 階段就需要解析這些值來決定哪些請求需要經過 Middleware。

---

## Edge Runtime：V8 Isolate 的精簡世界

現在回到核心問題：Middleware 跑在什麼環境裡？

在 Next.js 16 之前，答案是 **Edge Runtime**。這不是 Node.js，也不是瀏覽器——它是一個基於 V8 引擎但**刻意閹割**過的輕量級執行環境。

### V8 Isolate 不是 Container

傳統的 Serverless Function（像 AWS Lambda）跑在容器裡。每次冷啟動，你需要：

1. 啟動一個 microVM 或 container
2. 載入整個 Node.js runtime
3. 初始化你的應用程式碼
4. 處理請求

這個過程動輒幾百毫秒到數秒。而 V8 Isolate 的做法完全不同：

```
傳統 Serverless:
┌──────────────────────────────────────────┐
│ VM / Container                           │
│  ┌────────────────────────────────────┐  │
│  │ Node.js Runtime (完整)             │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │ 你的應用程式碼              │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
冷啟動：100ms - 數秒

V8 Isolate:
┌──────────────────────────────────────────────────┐
│ 一個 Runtime Process（共用）                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ Isolate  │ │ Isolate  │ │ Isolate  │  ...     │
│  │ (你的碼) │ │ (別人的) │ │ (另一個) │         │
│  └──────────┘ └──────────┘ └──────────┘         │
│  記憶體互相隔離，但共享同一個 V8 引擎             │
└──────────────────────────────────────────────────┘
冷啟動：< 5ms（接近零）
```

Cloudflare Workers 率先採用了這個架構，他們的數據是：V8 Isolate 的啟動時間**不到 5 毫秒**，記憶體消耗只有傳統 Node.js Process 的十分之一。一台機器可以同時跑成千上萬個 Isolate，而每個 Isolate 之間的記憶體完全隔離。

這就是 Edge Runtime 快的原因——它不需要啟動整個 Node.js，只要在一個已經跑著的 V8 引擎裡面開一個新的 Isolate 就好。但代價是什麼？

### 你失去的東西

V8 Isolate 只提供 Web 標準 API，你拿到的是一個像瀏覽器 Service Worker 一樣的環境。具體來說：

**可以用的：**
- `fetch`、`Request`、`Response`、`Headers`
- `URL`、`URLSearchParams`、`URLPattern`
- `TextEncoder`、`TextDecoder`
- `crypto`（Web Crypto API，不是 Node.js 的 `crypto`）
- `ReadableStream`、`WritableStream`、`TransformStream`
- `setTimeout`、`setInterval`
- `structuredClone`
- `WebSocket`

**不能用的：**
- `fs`（檔案系統）——根本沒有檔案系統
- `net`、`http`——沒有原生網路模組
- `child_process`——不能執行外部程式
- `path`——沒有檔案路徑概念
- `Buffer`——用 `Uint8Array` 替代
- `eval()`、`new Function()`——安全限制，禁止動態程式碼執行
- `require()`——只能用 ES Modules

而且在 Vercel 上，Edge Function 的程式碼大小限制是 **1-4 MB**（包含所有 import 的套件）。相比之下，普通 Serverless Function 有 50 MB 的空間。

### 這個「標準」從哪來？

這些 API 的選擇不是 Vercel 一家說了算。2022 年 W3C 成立了 **WinterCG**（Web-interoperable Runtimes Community Group），成員包括 Vercel、Cloudflare、Deno、Shopify，共同定義了一套 **Minimum Common API**——所有 Edge Runtime 都應該支援的最小 Web API 集合。

2025 年初，WinterCG 升級為 Ecma 國際的 **TC55（WinterTC）**，正式成為一個可以發布標準的技術委員會。這意味著 `fetch`、`Request`/`Response`、`TextEncoder` 這些 API 不只是 Vercel 的實驗品，而是一個跨平台的正式標準。

Vercel 甚至把他們的 Edge Runtime 實現開源了——[@vercel/edge-runtime](https://edge-runtime.vercel.app/) 套件。本地開發時它用 polyfill 模擬 Web API，生產環境則跑在真正的 V8 Isolate 上。

---

## 那些讓人抓狂的 Edge 限制

理論上，Edge Runtime 很美好——接近零的冷啟動、全球分散部署、離用戶更近。但實際開發中，它讓很多人踩到坑。

### 坑一：資料庫連不上

這是最常見的痛點。傳統資料庫（PostgreSQL、MySQL、MongoDB）依賴 TCP 持久連線，而 Edge Runtime 不支援 `net` 模組。

```typescript
// ❌ 在 middleware 裡面這樣寫？直接死給你看
import { PrismaClient } from '@prisma/client'
const prisma = new PrismaClient()

export async function middleware(request: NextRequest) {
  const user = await prisma.user.findUnique({ ... }) // 爆炸
}
```

變通方案是使用 HTTP-based 的資料庫連線：
- **Neon**：提供 HTTP/WebSocket 介面的 PostgreSQL
- **PlanetScale**：HTTP API 的 MySQL
- **Prisma Data Proxy**：在你的資料庫前面加一個 HTTP 代理層

但說實話，你真的需要在 Middleware 裡查資料庫嗎？

### 坑二：Auth 套件炸裂

NextAuth.js（現在叫 Auth.js）是個經典案例。如果你用 database session strategy，它需要連資料庫查 session——在 Edge Runtime 裡做不到。

解法是拆成兩層：

```typescript
// proxy.ts — 只做 JWT 驗證，不碰資料庫
import { jwtVerify } from 'jose' // jose 是 Edge 相容的 JWT 庫

export async function proxy(request: NextRequest) {
  const token = request.cookies.get('session-token')?.value
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  try {
    const secret = new TextEncoder().encode(process.env.JWT_SECRET)
    await jwtVerify(token, secret)
    return NextResponse.next()
  } catch {
    return NextResponse.redirect(new URL('/login', request.url))
  }
}
```

注意這裡用的是 `jose` 而不是 `jsonwebtoken`——後者依賴 Node.js 的 `crypto` 模組，在 Edge 上直接爆。`jose` 用的是 Web Crypto API，Edge 完全相容。

### 坑三：無限重定向

Middleware 最陰險的 bug。你寫了一個「未登入就導向 /login」的邏輯，但忘了把 `/login` 從 `matcher` 裡排除：

```typescript
// ❌ 無限迴圈：/login 也被攔截，又導向 /login...
export function proxy(request: NextRequest) {
  if (!request.cookies.get('token')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
}

export const config = {
  matcher: ['/:path*'] // 匹配所有路徑，包括 /login
}
```

修正：

```typescript
export const config = {
  matcher: ['/((?!login|api|_next/static|_next/image|favicon.ico).*)']
}
```

### 坑四：你以為排除了，其實沒有

即使你在 matcher 裡用負向前瞻排除了 `_next/data`，Middleware 仍然會攔截 `/_next/data/*` 路由。這是 Next.js 的**刻意行為**——防止你保護了頁面但忘了保護對應的 data route。

---

## Next.js 16 的關鍵轉折：middleware.ts → proxy.ts

講完了 Edge Runtime 的限制，你應該能理解為什麼 Next.js 16 做了一個大動作。

### 改名的真正原因

Next.js 16 把 `middleware.ts` 重新命名為 `proxy.ts`，乍看只是改個名字，但底層的意圖深遠。官方給了兩個理由：

**第一，「middleware」這個詞被誤用了。** 很多人以為它是 Express.js 那種 middleware——可以鏈式呼叫、可以做任何事。但 Next.js 的 Middleware 其實更像是一個**網路代理**：它坐在 CDN 層，在請求到達你的應用之前做一些輕量處理。叫它 "proxy" 更精確地描述了它的角色。

**第二，更重要的是，`proxy.ts` 預設跑在 Node.js Runtime。** 不再是 Edge Runtime。

```typescript
// Next.js 15 (middleware.ts) — 預設 Edge Runtime
export function middleware(request: NextRequest) { ... }

// Next.js 16 (proxy.ts) — 預設 Node.js Runtime
export function proxy(request: NextRequest) { ... }
// 邏輯完全一樣，但底層 runtime 不同
```

遷移很簡單，官方甚至提供了 codemod：

```bash
npx @next/codemod@canary middleware-to-proxy .
```

它會自動把檔名從 `middleware.ts` 改成 `proxy.ts`，函式名從 `middleware` 改成 `proxy`。

### 為什麼回歸 Node.js？

這個決定反映了一個現實：**Edge Runtime 的限制對大多數應用來說太多了。**

GitHub 上有一個超過 400 條回覆的 Discussion（[#46722](https://github.com/vercel/next.js/discussions/46722)），標題就是「Allow Node.js APIs in Middleware」。開發者們列出了各種 Edge Runtime 讓他們痛苦的場景：

- 連資料庫做權限檢查
- 用現有的 Node.js Auth 套件
- 存取檔案系統讀取設定
- 使用依賴 Node.js API 的第三方套件

Next.js 15.2 開始實驗性支援 Middleware 的 Node.js Runtime，15.5 進入 stable，最終在 16.0 直接把 Node.js 設為預設。

**但這不代表 Edge Runtime 死了。** 如果你的使用場景真的適合 Edge（純粹的重定向、地理位置判斷、簡單的 header 操作），你仍然可以用 `middleware.ts`（deprecated 但還能用）。Edge 的近零冷啟動和全球邊緣部署優勢在特定場景下仍然無可取代。

### proxy.ts 的設計哲學：Thin Proxy

Next.js 團隊明確表態：`proxy.ts` 應該是一個 **Thin Proxy**——做輕量的請求攔截，不要把它當成 Express middleware 來寫。

**適合放在 proxy.ts 的：**
- URL 重定向和改寫
- 設定/讀取 cookie 和 header
- 簡單的 JWT token 驗證
- A/B 測試的路由分流
- 地理位置判斷（用 `request.geo`）

**不適合放在 proxy.ts 的：**
- 複雜的資料庫查詢
- 大量的商業邏輯
- 重度的資料處理
- 任何會顯著增加回應時間的操作

為什麼？因為 proxy.ts 跑在**每一個請求**的關鍵路徑上。任何延遲都會影響所有用戶的所有請求。它的角色是快速決策——放行、重定向、還是拒絕——然後讓路由層接手。

---

## 實戰：什麼時候用 Edge，什麼時候用 Node.js？

| 場景 | 推薦 Runtime | 原因 |
|------|-------------|------|
| 地理位置重定向 | Edge | 延遲最低，離用戶最近 |
| 簡單的 cookie/header 檢查 | Edge | 不需要 Node.js API |
| Bot 偵測與基本限流 | Edge | 在邊緣就擋掉，省後端資源 |
| 資料庫驅動的權限檢查 | Node.js | 需要資料庫連線 |
| 複雜的 Auth 邏輯 | Node.js | 大多數 Auth 套件依賴 Node.js API |
| 使用第三方 Node.js 套件 | Node.js | Edge 的套件相容性差 |
| 大部分生產應用 | Node.js（proxy.ts） | 限制少、相容性好 |

如果你在 2026 年開始一個新的 Next.js 16 專案，**直接用 `proxy.ts` + Node.js Runtime** 就對了。除非你有非常明確的延遲需求（例如全球用戶的地理位置判斷），否則 Edge 帶來的限制不值得那幾毫秒的冷啟動優勢。

---

## 結論：Edge 是好工具，但不是萬能藥

Edge Runtime 基於 V8 Isolate 的架構確實是一個工程傑作——近零冷啟動、極低記憶體消耗、全球分散部署。Cloudflare Workers 證明了這個模型在 CDN 層級的強大威力。

但 Next.js 社群用了三年多的時間證明了另一件事：**把應用層的邏輯硬塞進 Edge Runtime 的限制裡，弊大於利。** 大多數真實世界的 Middleware 使用場景——Auth、權限檢查、動態路由——都需要 Node.js 的能力。

Next.js 16 的 `proxy.ts` 是一個務實的修正。它承認了 Edge 的定位不是「取代所有 Serverless」，而是「在特定場景下的極速攔截器」。把名字從 middleware 改成 proxy，不只是語義上的精準，更是架構上的自我定位。

我的建議很簡單：用 `proxy.ts`，保持它輕量，把複雜邏輯留給 Server Component 和 Route Handler。如果你有全球延遲敏感的場景，再考慮 Edge。不要為了追求「Edge 原生」而把自己搞得綁手綁腳——技術選型永遠是 trade-off，不是信仰。

---

## 延伸閱讀

- [Next.js 16 Blog Post](https://nextjs.org/blog/next-16) — proxy.ts 的官方公告和完整 changelog
- [Next.js proxy.ts 文件](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) — API 參考、matcher 語法、完整範例
- [Next.js Edge Runtime API Reference](https://nextjs.org/docs/app/api-reference/edge) — 支援和不支援的 API 完整清單
- [Cloudflare Workers: How Workers Works](https://developers.cloudflare.com/workers/reference/how-workers-works/) — V8 Isolate 架構的最佳解釋
- [Vercel Edge Runtime（開源）](https://edge-runtime.vercel.app/) — Vercel 的 Edge Runtime 開源實現
- [GitHub Discussion #46722](https://github.com/vercel/next.js/discussions/46722) — 社群要求 Middleware 支援 Node.js Runtime 的討論串
- [WinterTC (TC55)](https://wintertc.org/faq) — Edge Runtime API 標準化的 Ecma 技術委員會
