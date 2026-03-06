---
title: "Next.js 的靜態與動態渲染決策樹：你的頁面到底怎麼被決定命運的？"
date: 2026-03-06
description: "深入解析 Next.js App Router 如何在 build time 決定一個路由是靜態還是動態渲染。從 bailout 機制、Dynamic APIs 偵測、fetch 快取順序陷阱，到 PPR 和 Cache Components 的新模型，完整拆解這棵決策樹的每一個分支。"
tags: [deep-dive, nextjs, react, frontend, performance]
---

你有沒有遇過這種情況：明明只是在頁面裡讀了一下 `cookies()`，整個路由就從靜態變成動態，build 輸出那個漂亮的 `○`（靜態）瞬間變成 `λ`（動態）？或者更詭異的——你在某個 Server Component 裡加了一個 `fetch`，結果頁面的渲染行為完全變了，但你不知道為什麼？

Next.js App Router 最讓人又愛又恨的設計之一，就是它的**自動渲染策略選擇**。框架幫你決定頁面該靜態還是動態渲染，聽起來很美好，但「幫你決定」的前提是你得知道它怎麼決定的。否則你就是把命運交給一個黑盒子。

今天我們來把這個黑盒子拆開。

---

## 從 Pages Router 的二元世界說起

在 Pages Router 時代，事情很單純。你在頁面裡 export 了 `getStaticProps`，它就是靜態的；export 了 `getServerSideProps`，它就是動態的。沒 export 任何 data fetching 函式？那就走 Automatic Static Optimization，build 時直接生成 HTML。

```javascript
// Pages Router — 你明確告訴框架要什麼
export async function getStaticProps() {
  // 我是靜態的，build 時跑一次
  return { props: { data: await fetchData() } }
}

export async function getServerSideProps(context) {
  // 我是動態的，每個 request 都跑
  return { props: { data: await fetchData(context.req) } }
}
```

二選一，非黑即白。粗暴但清楚。

App Router 把這套推翻了。它說：「你不用自己選，我幫你判斷。」聽起來像是進步，但代價是你必須理解它的判斷邏輯——否則你連頁面為什麼突然變慢都搞不清楚。

---

## 核心決策樹：Next.js 怎麼決定你的路由命運

App Router 的渲染決策本質上是一個**排除法**：預設靜態，遇到特定條件就降級為動態。我把它畫成一棵決策樹：

```
路由渲染決策（預設：靜態）
│
├─ 有沒有設定 route segment config？
│   ├─ dynamic = 'force-dynamic'  ──→ 強制動態 ✦
│   ├─ dynamic = 'force-static'   ──→ 強制靜態（cookies/headers 回空值）
│   ├─ dynamic = 'error'          ──→ 強制靜態（碰到 Dynamic API 就報錯）
│   └─ dynamic = 'auto'（預設）    ──→ 繼續往下判斷
│
├─ revalidate 設了什麼？
│   ├─ revalidate = 0             ──→ 動態渲染 ✦
│   ├─ revalidate = N (N > 0)     ──→ ISR（靜態 + 定時重驗）
│   └─ revalidate = false（預設） ──→ 繼續往下判斷
│
├─ 有沒有使用 Dynamic APIs？
│   ├─ cookies()                  ──→ bailout，動態渲染 ✦
│   ├─ headers()                  ──→ bailout，動態渲染 ✦
│   ├─ searchParams               ──→ bailout，動態渲染 ✦
│   ├─ connection()               ──→ bailout，動態渲染 ✦
│   ├─ draftMode()                ──→ bailout，動態渲染 ✦
│   └─ 都沒用                     ──→ 繼續往下判斷
│
├─ fetch 請求的快取設定？
│   ├─ 任何 fetch 用了 cache: 'no-store'  ──→ 動態渲染 ✦
│   ├─ 任何 fetch 用了 revalidate: 0      ──→ 動態渲染 ✦
│   └─ 都是 force-cache 或有正數 revalidate ──→ 繼續判斷
│
├─ 有沒有呼叫 unstable_noStore()？
│   ├─ 有  ──→ 動態渲染 ✦
│   └─ 沒有 ──→ 繼續判斷
│
└─ 以上都沒觸發 ──→ 靜態渲染 ○
```

看起來很多分支，但核心邏輯只有一句話：**預設靜態，碰到任何需要 request context 或明確不快取的東西就降級為動態**。

---

## Bailout 機制：框架怎麼「偵測」動態需求

這是整個決策樹最關鍵的底層機制。Next.js 不是用什麼靜態分析去掃你的程式碼——它用的是 **runtime 拋錯**。

在 build time，Next.js 會嘗試對每個路由進行預渲染（prerender）。當你的 Server Component 在渲染過程中呼叫了 `cookies()`、`headers()` 或存取 `searchParams` 這類 Dynamic API 時，這些函式內部會拋出一個**特殊的 bailout 錯誤物件**。

```javascript
// 簡化的概念虛擬碼 — Next.js 內部的 Dynamic API 偵測
function cookies() {
  if (isStaticGeneration()) {
    // 不是普通的 Error，是特殊的 bailout signal
    throw new DynamicServerError(
      'cookies() expects to have requestAsyncStorage'
    )
  }
  return getRequestCookies()
}
```

Next.js 的預渲染器會捕捉這個特殊錯誤。捕到之後，它知道：「好，這個路由沒辦法在 build time 完成，標記為動態。」

這個設計有一個非常重要的後果：**bailout 是全有或全無的**。一旦路由中任何一個地方觸發了 bailout，整個路由就變成動態渲染。不是那個元件變動態、其他保持靜態——是**整條路由**。

```javascript
// 即使 cookies() 只在一個很小的子元件裡用了
// 整個路由都會變成動態
export default async function Page() {
  return (
    <div>
      <StaticHeader />       {/* 這個本來可以靜態 */}
      <StaticContent />      {/* 這個也可以 */}
      <UserGreeting />       {/* 但這裡用了 cookies()... */}
    </div>
  )
}

async function UserGreeting() {
  const cookieStore = await cookies() // 一個 bailout，全部動態
  const name = cookieStore.get('name')?.value
  return <p>Hi, {name}</p>
}
```

這就是為什麼你在一個大頁面裡只加了一行 `cookies()` 讀取，整個頁面的效能特性就完全改變了。

### 不要 try/catch Dynamic APIs

因為 bailout 靠的是拋錯，如果你不小心把 Dynamic API 包在 `try/catch` 裡，Next.js 就攔不到那個 bailout signal，後果就是 build 直接失敗：

```javascript
// ❌ 這會造成問題
export default async function Page() {
  try {
    const cookieStore = await cookies()
    // ...
  } catch (error) {
    // 你攔截了 bailout error！
    // Next.js 無法正確判斷渲染策略
    return <FallbackUI />
  }
}
```

官方文件明確警告：如果你必須在 Dynamic API 附近使用 try/catch，要確保重新拋出原始錯誤，或者在 try/catch 之前呼叫 `unstable_noStore()` 來明確宣告動態意圖。

---

## fetch 的順序陷阱：位置決定快取行為

App Router 裡的 `fetch` 有一個極其反直覺的行為：**同一個路由裡，fetch 被快取還是不被快取，取決於它相對於 Dynamic API 呼叫的位置**。

規則是這樣的：

- 在 Dynamic API **之前**的 `fetch`：走預設的快取策略（被快取）
- 在 Dynamic API **之後**的 `fetch`：不被快取

```javascript
export default async function Page() {
  // ① 這個 fetch 在 Dynamic API 之前
  // → 預設被快取（cache: 'force-cache'）
  const posts = await fetch('https://api.example.com/posts')

  // ② Dynamic API 呼叫 — bailout 發生在這裡
  const headersList = await headers()

  // ③ 這個 fetch 在 Dynamic API 之後
  // → 預設不被快取（cache: 'no-store'）
  const comments = await fetch('https://api.example.com/comments')

  return <div>...</div>
}
```

是的，你沒看錯。**同一個頁面裡的兩個 fetch，僅僅因為程式碼的順序不同，快取行為就不同。**

這個設計的原因是：一旦 Next.js 偵測到 Dynamic API，它認為「從這裡開始，開發者可能需要 request-specific 的資料」，所以後續的 fetch 預設不快取。但這個「聰明」的推斷對很多開發者來說是個地雷。

### 怎麼避免這個陷阱

明確設定每個 `fetch` 的快取策略，不要依賴預設值：

```javascript
// ✅ 明確設定，不管在 Dynamic API 前還是後都一樣
const posts = await fetch('https://api.example.com/posts', {
  cache: 'force-cache'  // 我要快取
})

const comments = await fetch('https://api.example.com/comments', {
  cache: 'no-store'     // 我不要快取
})
```

或者用 `fetchCache` route segment config 來全面覆蓋：

```javascript
// 整個路由的 fetch 都不快取
export const fetchCache = 'force-no-store'

// 或者整個路由的 fetch 都快取
export const fetchCache = 'force-cache'
```

---

## Route Segment Config：手動控制的四個檔位

當自動判斷不符合你的需求時，`dynamic` 這個 route segment config 就是你的手動排檔：

### `'auto'`（預設）

讓框架自動判斷。大部分情況下這就夠了，但你得知道上面那棵決策樹。

### `'force-dynamic'`

強制動態渲染。等同於 Pages Router 的 `getServerSideProps`。每個 request 都重新渲染，所有 fetch 都不快取。

```javascript
export const dynamic = 'force-dynamic'

export default async function Dashboard() {
  // 每個 request 都是新鮮的
  const data = await fetchDashboardData()
  return <DashboardComponent data={data} />
}
```

**用途**：快速原型開發、高度動態的 dashboard、你懶得想快取策略的時候。

### `'force-static'`

強制靜態渲染。最狠的一招——就算你用了 `cookies()` 或 `headers()`，它也不會 bailout，而是讓這些函式**回傳空值**。

```javascript
export const dynamic = 'force-static'

export default async function Page() {
  const cookieStore = await cookies()
  // cookieStore 是空的！不會拿到任何 cookie
  // 但頁面會被靜態渲染
  return <div>...</div>
}
```

**用途**：你確定這個頁面不需要 request-specific 的資料，但程式碼裡恰好引入了會觸發 bailout 的 API。小心使用，因為你會拿到空的 cookies/headers。

### `'error'`

最嚴格的靜態保護。如果頁面中任何地方使用了 Dynamic API 或未快取的 fetch，**build 直接報錯**。

```javascript
export const dynamic = 'error'

export default async function Page() {
  // 如果這裡不小心用了 cookies()，build 會直接失敗
  // 適合那些「這個頁面永遠不該是動態的」的情境
  const data = await fetch('https://api.example.com/data', {
    cache: 'force-cache'
  })
  return <div>...</div>
}
```

**用途**：關鍵的靜態頁面（landing page、文件頁），你想確保沒有人不小心加了動態邏輯進來。

---

## 靜態到動態的那條紅線：Static to Dynamic Error

Next.js 有一個硬性限制：一個路由在 build time 被判定為靜態之後，runtime 時不能突然變成動態。如果你的程式碼在 build time 走靜態路徑，但在 runtime 的 ISR revalidation 或 fallback 路徑中存取了 Dynamic API，你會看到這個錯誤：

```
Error: Page changed from static to dynamic at runtime
```

這通常發生在條件式使用 Dynamic API 的情況：

```javascript
// ❌ 這會炸
export default async function Page({ searchParams }) {
  if (someCondition) {
    // build time 沒走到這個分支 → 判定靜態
    // runtime 走到這個分支 → 嘗試動態 → 報錯
    const cookieStore = await cookies()
    return <DynamicContent cookies={cookieStore} />
  }
  return <StaticContent />
}
```

解法很簡單：**不要條件式使用 Dynamic API**。要嘛整個頁面就是動態的，要嘛就別用這些 API。

---

## 從二元到光譜：PPR 和 Cache Components

看到這裡你可能覺得很困擾：一個頁面 90% 是靜態內容，就因為一個小角落讀了 `cookies()`，整個頁面就要動態渲染？這也太浪費了吧。

Vercel 團隊自己也意識到了這個問題。Sebastian Markbåge（React 核心成員、在 Vercel 工作）在官方部落格的 "Our Journey with Caching" 一文裡承認：

> 「fetch() 預設快取的設定讓效能最佳化，但快速原型開發和高度動態的應用因此受苦。我們沒有為不用 fetch() 的本地資料庫存取提供足夠的控制。」

這催生了兩個重大演進：

### Partial Prerendering (PPR)

PPR 的核心想法是：**不再整個路由非靜態即動態，而是用 `<Suspense>` 邊界把靜態和動態切開**。

```javascript
import { Suspense } from 'react'
import { cookies } from 'next/headers'

export default function Page() {
  return (
    <div>
      {/* ① 靜態殼 — build time 預渲染 */}
      <header>
        <h1>My Store</h1>
        <nav>...</nav>
      </header>

      {/* ② 靜態殼的一部分 */}
      <ProductList />

      {/* ③ 動態洞 — request time 串流填入 */}
      <Suspense fallback={<CartSkeleton />}>
        <Cart />   {/* 這裡讀 cookies()，但只影響這個 Suspense 邊界 */}
      </Suspense>
    </div>
  )
}
```

PPR 啟用後，bailout 不再是「全路由降級」，而是「降級到最近的 Suspense 邊界」。header 和 ProductList 依然是靜態殼、瞬間送到瀏覽器，Cart 的動態部分用 streaming 補上。

整個回應只需要一個 HTTP request——靜態殼和動態 chunks 在同一個 response 裡串流送出。

### Cache Components（Next.js 16）

Next.js 16 把這個模型再推一步，引入了 `use cache` 指令和 `cacheComponents` 設定。開啟後，**原本那些 route segment config（`dynamic`、`fetchCache`、`revalidate`）都不再需要了**。整個模型簡化成兩個概念：

1. **`<Suspense>`**：把需要 request time 資料的部分包起來，延遲到請求時渲染
2. **`use cache`**：把可以快取的資料或元件標記起來，納入靜態殼

```javascript
import { Suspense } from 'react'
import { cookies } from 'next/headers'
import { cacheLife } from 'next/cache'

export default function BlogPage() {
  return (
    <>
      {/* 自動靜態 — 純計算，沒有 I/O */}
      <header><h1>Our Blog</h1></header>

      {/* use cache — 動態資料被快取，納入靜態殼 */}
      <BlogPosts />

      {/* Suspense — request time 串流 */}
      <Suspense fallback={<p>Loading...</p>}>
        <UserPreferences />
      </Suspense>
    </>
  )
}

async function BlogPosts() {
  'use cache'
  cacheLife('hours')
  const res = await fetch('https://api.example.com/posts')
  const posts = await res.json()
  return <PostList posts={posts} />
}

async function UserPreferences() {
  const theme = (await cookies()).get('theme')?.value || 'light'
  return <aside>Theme: {theme}</aside>
}
```

這個模型最大的改變是：**不再有隱式的快取預設值**。在 Cache Components 模式下，如果你的元件存取了網路資源但沒有用 `use cache` 標記、也沒有被 `<Suspense>` 包住，build 時會直接報錯。你必須明確選擇——要快取還是要動態。

這比「我幫你自動判斷」誠實多了。

### `use cache` vs route segment config 對照表

| 舊模型 | 新模型 |
|--------|--------|
| `export const dynamic = 'force-dynamic'` | 不需要，預設就是動態 |
| `export const dynamic = 'force-static'` | `'use cache'` + `cacheLife('max')` |
| `export const revalidate = 3600` | `'use cache'` + `cacheLife('hours')` |
| `export const fetchCache = 'force-cache'` | `'use cache'`（scope 內的 fetch 自動快取）|
| `unstable_cache()` | `'use cache'` 函式級別 |

---

## 實戰：你現在該怎麼做

### 1. 搞清楚你的 Next.js 版本

- **Next.js 14-15（無 cacheComponents）**：你活在 route segment config 的世界。理解上面那棵決策樹是必要的。
- **Next.js 16+（啟用 cacheComponents）**：擁抱 `use cache` + `<Suspense>` 模型，忘掉 route segment config。

### 2. 別依賴自動判斷

不管用哪個版本，最安全的做法是**明確宣告意圖**：

```javascript
// ✅ 明確告訴 fetch 要不要快取
const data = await fetch(url, { cache: 'force-cache' })
const fresh = await fetch(url, { cache: 'no-store' })

// ✅ 明確設定路由層級的渲染策略
export const dynamic = 'force-dynamic' // 或 'force-static'
```

### 3. 小心 Dynamic API 的擴散

Dynamic API 的 bailout 是會「傳染」的。如果你在一個共用的 utility 函式裡呼叫了 `headers()`，任何引入這個函式的頁面都會變成動態渲染：

```javascript
// ❌ 危險：這個 utility 會讓所有引入它的頁面變動態
export async function getLocale() {
  const headersList = await headers()
  return headersList.get('accept-language') || 'en'
}

// 在你的 Page 裡用了它 → 整個頁面動態
export default async function Page() {
  const locale = await getLocale() // boom, 動態了
  return <Content locale={locale} />
}
```

### 4. 用 build 輸出驗證

`next build` 的輸出會清楚標示每個路由的渲染策略：

```
Route (app)                    Size    First Load JS
┌ ○ /                          1.2 kB  87.3 kB
├ ○ /about                     543 B   86.7 kB
├ λ /dashboard                 2.1 kB  88.2 kB
├ ○ /blog/[slug]               1.8 kB  87.9 kB
└ λ /profile                   890 B   87.0 kB

○  (Static)   prerendered as static content
λ  (Dynamic)  server-rendered on demand
```

如果你預期是 `○` 但看到 `λ`，回頭檢查是不是某個地方偷偷呼叫了 Dynamic API。

---

## 別讓框架替你做決定

Next.js 的靜態/動態渲染自動決策是一把雙面刃。在簡單的場景下，它幫你省了思考；在複雜的場景下，它幫你挖了坑。

理解這棵決策樹之後，你可以做到兩件事：

1. **精準控制**：知道每一行程式碼對渲染策略的影響，不再被「為什麼這個頁面突然變慢」困惑
2. **有意識地設計**：把 Dynamic API 的使用限制在最小範圍，讓大部分頁面享受靜態渲染的效能紅利

如果你用的是 Next.js 16，強烈建議開啟 `cacheComponents`，擁抱 `use cache` + `<Suspense>` 的新模型。它把「幫你猜」變成「逼你選」——聽起來更麻煩，但比踩地雷好太多了。

---

## 延伸閱讀

- [Next.js 官方文件：Cache Components](https://nextjs.org/docs/app/getting-started/cache-components) — Next.js 16 的新渲染模型完整文件
- [Next.js 官方文件：Caching in Next.js](https://nextjs.org/docs/app/guides/caching) — 四層快取機制的完整解析
- [Route Segment Config API](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config) — `dynamic`、`revalidate`、`fetchCache` 等設定的官方參考
- [Our Journey with Caching — Vercel Blog](https://nextjs.org/blog/our-journey-with-caching) — Sebastian Markbåge 談 Next.js 快取策略的演進和反思
- [Resolving "app/ Static to Dynamic Error"](https://nextjs.org/docs/messages/app-static-to-dynamic-error) — 靜態到動態切換錯誤的官方除錯指南
- [Static Bail Out Caught](https://nextjs.org/docs/messages/ppr-caught-error) — PPR 模式下 bailout 被 try/catch 攔截的處理方式
