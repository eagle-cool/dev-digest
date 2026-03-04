---
title: "Next.js App Router 路由解析機制：從檔案系統到 LoaderTree 的編譯全過程"
date: 2026-03-04
description: "深入 Next.js App Router 原始碼，解析檔案系統如何編譯成 LoaderTree、FlightRouterState 如何驅動 Server-Centric Routing、Layout 嵌套的實作原理，以及 Parallel Routes 和 Intercepting Routes 的內部機制。"
tags: [deep-dive, nextjs, react, frontend]
---

你有沒有想過，當你在 `app/` 資料夾裡新增一個 `page.tsx`，Next.js 是怎麼知道它對應哪個 URL 的？

「不就是檔案系統路由嗎？資料夾名稱等於路徑，有什麼好研究的。」

如果你的理解停留在這裡，那你大概也解釋不了：為什麼 `@modal` 資料夾不會出現在 URL 裡？為什麼 `(marketing)` 這種括號資料夾可以用來分組但不影響路由？為什麼 `error.tsx` 抓不到同層 `layout.tsx` 的錯誤？

這些行為不是魔法，是 Next.js 在 build time 做的一連串精密轉換。今天我們就從原始碼出發，把整個流程拆開來看。

---

## 一切的起點：LoaderTree

App Router 的核心資料結構叫 `LoaderTree`。整個路由系統——從 build time 的檔案掃描到 runtime 的元件渲染——都圍繞這個結構運轉。它的定義藏在 `packages/next/src/server/lib/app-dir-module.ts`：

```typescript
export type LoaderTree = [
  segment: string,
  parallelRoutes: { [parallelRouterKey: string]: LoaderTree },
  modules: AppDirModules,
  staticSiblings: readonly string[] | null,
]
```

一個四元組（4-tuple）。夠簡潔吧？但別被它的簡潔騙了，每個欄位都有講究：

- **`segment`**：URL 路徑片段。`"dashboard"`、`"[id]"`、或者特殊的 `"__PAGE__"` 表示葉節點頁面
- **`parallelRoutes`**：一個 record，key 是 parallel route 的名稱（預設是 `"children"`），value 是子 LoaderTree。這是遞迴結構的關鍵
- **`modules`**：這一層的所有特殊檔案——`layout`、`page`、`loading`、`error`、`template`、`not-found`，每個都是一個 lazy import 函式
- **`staticSiblings`**：prefetch 最佳化用的。對動態區段 `[id]` 來說，這裡記錄了同層的靜態兄弟路徑（例如 `['sale']`）

注意 `parallelRoutes` 的設計：它不是一個陣列，而是一個 **key-value 的 record**。這意味著 Parallel Routes 不是事後加上去的功能，而是從一開始就被嵌進資料結構的基因裡。每個路由節點天生就能容納多個平行的子樹。

---

## Build Time：next-app-loader 怎麼掃描檔案系統

LoaderTree 不是在 runtime 動態建構的，它在 build time 就被 webpack loader 編譯成 JavaScript 程式碼。負責這件事的是 `packages/next/src/build/webpack/loaders/next-app-loader/index.ts`。

### 第一步：解析檔案

loader 拿到一個路徑後，會嘗試所有已配置的副檔名：

```typescript
const resolver: PathResolver = async (pathname) => {
  const absolutePath = createAbsolutePath(appDir, pathname)
  const filenameIndex = absolutePath.lastIndexOf(path.sep)
  const dirname = absolutePath.slice(0, filenameIndex)
  const filename = absolutePath.slice(filenameIndex + 1)

  const checks = await Promise.all(
    extensions.map(async (ext) => {
      const absolutePathWithExtension = `${absolutePath}${ext}`
      const exists = await fileExistsInDirectory(dirname, `${filename}${ext}`)
      return exists ? absolutePathWithExtension : undefined
    })
  )
  return checks.find((result) => result)
}
```

所以當它要找 `layout`，實際上是在找 `layout.tsx`、`layout.ts`、`layout.jsx`、`layout.js`——依序嘗試，先找到的贏。

### 第二步：發現 Parallel Routes

`resolveParallelSegments` 函式負責掃描每一層目錄下的路由結構：

```typescript
const resolveParallelSegments = (pathname) => {
  const matched = {}

  for (const appPath of normalizedAppPaths) {
    if (appPath.startsWith(pathname + '/')) {
      const rest = appPath.slice(pathname.length + 1).split('/')

      // 普通頁面 → children slot
      if (rest.length === 1 && rest[0] === 'page') {
        matched.children = PAGE_SEGMENT
        continue
      }

      // @ 開頭 → parallel route slot
      const isParallelRoute = rest[0].startsWith('@')
      if (isParallelRoute) {
        matched[rest[0]] = /* ... */
        continue
      }

      // 一般子目錄 → children slot
      matched.children = rest[0]
    }
  }
  return Object.entries(matched)
}
```

看到了嗎？`@sidebar`、`@modal` 這些 `@` 開頭的資料夾，在這裡被識別出來，變成 `parallelRoutes` record 裡的獨立 key。而普通的子資料夾和 `page.tsx` 都歸到 `children` 這個預設 key 底下。

### 第三步：路徑正規化

`normalizeAppPath` 負責把檔案系統路徑轉成 URL 路徑。這個函式揭示了所有「不會出現在 URL 裡」的東西是怎麼被過濾掉的：

```typescript
export function normalizeAppPath(route: string) {
  return ensureLeadingSlash(
    route.split('/').reduce((pathname, segment) => {
      if (!segment) return pathname
      if (isGroupSegment(segment)) return pathname   // (group) → 消失
      if (segment[0] === '@') return pathname          // @slot → 消失
      if (segment === 'page' || segment === 'route')
        return pathname                                // 葉節點 → 消失
      return `${pathname}/${segment}`
    }, '')
  )
}
```

三行 `return pathname` 就決定了三個核心行為：

1. **Route Groups** `(marketing)` → 用括號包起來的資料夾被完全跳過，不影響 URL
2. **Parallel Routes** `@modal` → `@` 開頭的 slot 不產生路徑
3. **Leaf Files** `page`/`route` → 終端檔案不會變成路徑的一部分

所以 `app/(marketing)/about/page.tsx` 的 URL 是 `/about`，不是 `/(marketing)/about/page`。沒有黑魔法，就是字串處理。

### 第四步：產出程式碼

最終，loader 把整棵 LoaderTree 序列化成 JavaScript 程式碼。每個模組引用都變成 `() => import()` 的 lazy 函式：

```typescript
const header = collectedDeclarations
  .map(([varName, modulePath]) =>
    `const ${varName} = () => import(/* webpackMode: "eager" */ ${
      JSON.stringify(modulePath)
    });\n`
  )
  .join('')
```

注意 `webpackMode: "eager"`——這些 import 雖然寫成動態形式，但實際上是 eager 載入的。LoaderTree 是在 server 端使用的，不需要做 code splitting。

---

## Runtime：從 LoaderTree 到 React 元件樹

Build time 產出的 LoaderTree 是一個靜態的資料結構。到了 runtime，`createComponentTree` 函式（在 `packages/next/src/server/app-render/create-component-tree.tsx`）把它轉換成真正的 React 元件樹。

這個過程的核心邏輯是遞迴遍歷 LoaderTree，在每一層做三件事：

1. **載入這一層的特殊檔案**（layout、template、error、loading、not-found）
2. **遞迴處理所有 parallel routes**（用 `Promise.all` 並行處理）
3. **組裝巢狀結構**——layout 包 error boundary，error boundary 包 suspense，suspense 包 children

假設你有這樣的檔案結構：

```
app/
  layout.tsx
  blog/
    layout.tsx
    [slug]/
      page.tsx
      loading.tsx
      error.tsx
```

最終產出的元件樹概念上長這樣：

```jsx
<RootLayout>
  <BlogLayout>
    <ErrorBoundary fallback={<SlugError />}>
      <Suspense fallback={<SlugLoading />}>
        <SlugPage params={{ slug: 'hello-world' }} />
      </Suspense>
    </ErrorBoundary>
  </BlogLayout>
</RootLayout>
```

這裡有一個非常重要的細節：**`error.tsx` 包的是 children，不是同層的 layout**。

```
Layout
  └── ErrorBoundary    ← error.tsx 在這裡
       └── Suspense    ← loading.tsx 在這裡
            └── Page
```

這就是為什麼 `app/blog/error.tsx` **抓不到** `app/blog/layout.tsx` 裡的錯誤——Error Boundary 包在 Layout 的「裡面」，而不是「外面」。要抓 Layout 的錯誤，你需要把 `error.tsx` 放到上一層（`app/error.tsx`）。

這不是 bug，是刻意的設計。Layout 是被其父層的 Error Boundary 保護的，每一層只負責捕獲自己子樹的錯誤。想想 React 的 Error Boundary 機制就明白了——boundary 只能抓到「裡面」的錯誤。

---

## FlightRouterState：渡過網路的路由樹

LoaderTree 活在 server 端，不會直接傳給 client。傳給 client 的是它的精簡版：`FlightRouterState`。

```typescript
export type FlightRouterState = [
  segment: Segment,
  parallelRoutes: { [parallelRouterKey: string]: FlightRouterState },
  refreshState?: CompressedRefreshState | null,
  refresh?: 'refetch' | 'inside-shared-layout' | 'metadata-only' | null,
  prefetchHints?: number,
]
```

又是一個 tuple。Next.js 似乎對 tuple 情有獨鍾——理由很實際：tuple 比 object 的序列化 payload 更小。省掉 key 名稱的字串開銷，在每次導航都要傳輸路由樹的場景下，積少成多。

最有意思的是第五個欄位 `prefetchHints`——一個 bitmask：

```typescript
export const enum PrefetchHint {
  HasRuntimePrefetch        = 0b00001,
  SubtreeHasInstant         = 0b00010,
  SegmentHasLoadingBoundary = 0b00100,
  SubtreeHasLoadingBoundary = 0b01000,
  IsRootLayout              = 0b10000,
}
```

這些 flag 會在建構 FlightRouterState 時**由下往上傳播**。如果某個深層子路由有 `loading.tsx`，它的 `SegmentHasLoadingBoundary` flag 就會被設定，然後透過 bitwise OR 一路傳到父節點的 `SubtreeHasLoadingBoundary`。

為什麼要這樣做？因為 prefetch 策略需要知道子樹裡有沒有 loading boundary。如果有，client 就可以在 prefetch 時只拿到 loading boundary 以上的靜態內容，loading boundary 以下的動態內容等使用者真的導航過去再載入。這就是 Next.js 做到「instant navigation」的關鍵——先給你看得到的靜態骨架，再逐步填入動態資料。

---

## Server-Centric Routing：路由匹配在 Server 端

這是 App Router 和 Pages Router 最根本的架構差異。

Pages Router 的做法：build time 產出一份路由表，client 下載這份路由表，然後所有路由匹配都在 client 端進行。

App Router 完全反過來。**路由匹配在 server 端進行。** 導航的流程是：

1. Client 把自己當前的 `FlightRouterState` 送到 server
2. Server 比對 client 的狀態和目標路由的 LoaderTree
3. Server 算出哪些 segment 需要重新渲染
4. Server 只回傳有變化的部分（React Server Component payload）

負責這個 diff 的是 `walkTreeWithFlightRouterState` 函式。它在每一層做三個判斷：

- **Client 沒有對應狀態** → 這是新 segment，完整渲染
- **Segment 不匹配** → 路由切換了，從這裡開始重新渲染
- **Client 請求 `'refetch'`** → 即使 segment 匹配也強制重新渲染

這個 diff 機制讓 App Router 實現了 **partial rendering**——導航時，已經渲染過且沒有變化的 layout 不會被重新渲染。只有變化的子樹才會被重新計算並傳回 client。

你在 `/blog` 頁面切換到 `/blog/hello-world` 時，Root Layout 和 Blog Layout 的 React 元件 **不會重新渲染**——因為 server 端的 diff 發現這兩層的 segment 沒有變化，只有 `[slug]` 這一層以下需要更新。

---

## Parallel Routes：不只是 slot，是路由樹的一等公民

Parallel Routes 的語法是 `@folder`，但它的實現比大多數人想像的更深入。

```
app/
  layout.tsx
  dashboard/
    layout.tsx
    page.tsx
    @sidebar/
      page.tsx
      default.tsx
    @modal/
      default.tsx
      (.)settings/
        page.tsx
```

在 LoaderTree 裡，`@sidebar` 和 `@modal` 不會變成 `children` 底下的子路由。它們是和 `children` **平行**的 key：

```
["dashboard", {
  children: ["__PAGE__", {}, { page: dashboardPage }],
  sidebar:  ["__PAGE__", {}, { page: sidebarPage }],
  modal:    ["__DEFAULT__", {}, { defaultPage: modalDefault }]
}, { layout: dashboardLayout }]
```

當 `createComponentTree` 處理到 `dashboard` 這一層時，它會把所有 parallel route 的渲染結果作為 **props** 傳給 Layout：

```jsx
<DashboardLayout
  sidebar={<SidebarPage />}
  modal={<ModalDefault />}
>
  <DashboardPage />   {/* ← 這是 children，也就是預設 slot */}
</DashboardLayout>
```

`children` 就是預設的 slot。你寫 `{children}` 實際上就是在渲染 `@children` 這個 parallel route。

### default.tsx 的存在意義

當某個 parallel route slot 在當前 URL 下沒有匹配的頁面時會發生什麼？

- **Soft navigation**（client 端導航）：slot 保持上一次匹配的內容不變
- **Hard navigation**（重新整理頁面）：slot 會渲染 `default.tsx`。如果沒有 `default.tsx`，直接 404

這就是為什麼官方文件一直強調要在 parallel route 裡放 `default.tsx`——它是 hard navigation 時的安全網。

---

## Intercepting Routes：不是黑魔法，是 segment 計數

Intercepting Routes 的語法看起來很嚇人：`(.)`、`(..)`、`(..)(..)`、`(...)`。但理解了原理之後其實很直觀。

| 語法 | 攔截目標 | 類比 |
|------|---------|------|
| `(.)` | 同層 | `./` |
| `(..)` | 上一層 | `../` |
| `(..)(..)` | 上兩層 | `../../` |
| `(...)` | 根層 | `/` |

**關鍵注意事項**：`(..)` 計算的是**路由 segment**，不是檔案系統層級。

`@modal` 是一個 slot，不算 route segment。所以 `app/@modal/(..)photo/[id]/page.tsx` 裡的 `(..)` 只往上跳了一個 route segment，即使在檔案系統裡它其實跨了兩層目錄。

在內部，intercepting routes 被編碼進動態參數的型別系統：

```typescript
export const dynamicParamTypes = {
  'dynamic-intercepted-(.)':      'di(.)',
  'dynamic-intercepted-(..)':     'di(..)',
  'dynamic-intercepted-(..)(..)': 'di(..)(..)',
  'dynamic-intercepted-(...)':    'di(...)',
  // catchall variants...
}
```

這些短碼（`di(..)` 之類的）被存在 `FlightRouterState` 的 `DynamicSegmentTuple` 裡，讓 client 端的 router 知道這個 segment 是被攔截的，需要特殊處理。

---

## Client 端的快取結構

client 端用 `CacheNode` 管理每個 segment 的快取：

```typescript
export type CacheNode = {
  rsc: React.ReactNode           // 完整的動態 RSC 資料
  prefetchRsc: React.ReactNode   // 預取的靜態版本
  prefetchHead: HeadData | null  // 預取的 metadata
  head: HeadData                 // 動態 metadata
  slots: Record<string, CacheNode> | null  // parallel route 子快取
}
```

注意 `rsc` 和 `prefetchRsc` 的分離。App Router 的「instant navigation」就是靠這個實現的：

1. 使用者 hover 連結時 → prefetch，把靜態部分存到 `prefetchRsc`
2. 使用者點擊連結 → 立刻顯示 `prefetchRsc` 的內容
3. 背景載入完整動態資料 → 更新到 `rsc`，用 `useDeferredValue` 無縫切換

這就是為什麼 App Router 的頁面切換感覺很快——你看到的第一幀是預取的靜態骨架，動態內容稍後填入，而 React 的 Concurrent Features 讓這個過渡幾乎感覺不到。

---

## 容易踩的坑

理解了內部機制，很多常見錯誤就有了解釋：

**1. 在 Server Component 裡呼叫 Route Handler**

兩者都跑在 server 上。從 Server Component 去 `fetch('/api/data')` 等於自己打自己——多了一次完全不必要的 HTTP round trip。直接呼叫函式就好。

**2. 在 try/catch 裡用 `redirect()`**

`redirect()` 的實作是 **throw 一個特殊的 error**。你用 try/catch 包住它，catch 會把這個 redirect 當成錯誤攔截掉。把 `redirect()` 放在 try/catch 外面。

**3. 以為 `loading.tsx` 隨時都會顯示**

`loading.tsx` 依賴 `<Suspense>` 機制。如果你的 page 裡沒有任何 async 操作會觸發 suspend，`loading.tsx` 根本不會出現。它不是「進入頁面就顯示的 splash screen」。

**4. Layout 裡 await 阻塞了 streaming**

如果父層 Layout 裡做了 `await fetchHeavyData()`，整個回應都會被阻塞。Streaming SSR 的前提是讓 Layout 快速返回，把耗時的資料抓取留給子元件透過 Suspense 處理。

**5. 到處加 `'use client'`**

把每個檔案都標上 `'use client'` 等於放棄了 Server Components 的所有好處。`'use client'` 標記的是 **邊界**——只在真正需要 client 功能（hooks、event handlers）的地方標記。透過 `{children}` 傳入的子元件仍然可以是 Server Component。

---

## 結論：資料結構決定了一切

回頭看 App Router 的整個架構，你會發現它的設計哲學非常清晰：**用一個精心設計的遞迴資料結構（LoaderTree）統一所有路由行為。**

檔案系統的掃描、特殊檔案的處理、parallel routes、layout 嵌套、error/loading 邊界的包裹順序——全部都是對這個 4-tuple 的操作。不需要配置檔、不需要路由表、不需要裝飾器。你的檔案結構就是你的路由定義，LoaderTree 就是它的編譯產物。

這個設計帶來的最大好處是 **consistency**——所有路由行為都可以從資料結構推導出來。但它的代價是 **implicit convention**：你必須記住 `(group)` 不產生路徑、`@slot` 是 parallel route、`(..)` 算的是 route segment 而不是目錄層級。這些規則不寫在程式碼裡，寫在檔案名稱的命名慣例裡。

我的建議是：如果你在用 App Router，花 30 分鐘看一下 `next-app-loader` 的原始碼。不需要全部看完，光是看 `resolveParallelSegments` 和 `normalizeAppPath` 這兩個函式，你就會對「為什麼我的路由行為不符合預期」這類問題有全新的理解。框架的原始碼永遠比 Stack Overflow 上的答案更可靠。

---

## 延伸閱讀

- [Next.js App Router 官方文件](https://nextjs.org/docs/app) — 起點，但只有 what，沒有 how
- [Layouts RFC](https://nextjs.org/blog/layouts-rfc) — App Router 的原始設計文件，理解「為什麼這樣設計」的最佳資料
- [Common Mistakes with the Next.js App Router](https://vercel.com/blog/common-mistakes-with-the-next-js-app-router-and-how-to-fix-them) — Vercel 官方整理的常見錯誤，值得逐條對照自己的專案
- [next-app-loader 原始碼](https://github.com/vercel/next.js/blob/canary/packages/next/src/build/webpack/loaders/next-app-loader/index.ts) — 檔案系統路由編譯的核心邏輯
- [create-component-tree.tsx 原始碼](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/app-render/create-component-tree.tsx) — LoaderTree 轉元件樹的實現
- [walk-tree-with-flight-router-state.tsx 原始碼](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/app-render/walk-tree-with-flight-router-state.tsx) — Server-centric routing 的 diff 演算法
