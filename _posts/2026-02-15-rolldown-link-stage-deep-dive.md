---
title: "Rolldown 鏈接階段深度拆解：一個 Rust 打包工具如何在符號層級解決 JavaScript 的模組混戰"
date: 2026-02-15
description: "深入解析 Rolldown 打包工具的鏈接階段（Link Stage）——如何用並查集追溯符號定義、用 Wrapper 機制搞定 CJS/ESM 互操作、用 DFS 傳播 Top-level Await 的傳染性。從資料結構到演算法，完整拆解這個即將取代 esbuild + Rollup 的 Rust 引擎核心。"
tags: [deep-dive, frontend]
---

Vite 8 Beta 已經發布，Rolldown 正式成為預設打包工具。Linear 的建置時間從 46 秒降到 6 秒，Beehiiv 減少了 64%。數字很漂亮，但「快」只是表象——真正值得聊的是，Rolldown 到底在底層做對了什麼。

最近一篇由 Atriiy 撰寫的技術文章，把 Rolldown 的鏈接階段（Link Stage）拆得非常透徹：從模組圖遍歷到符號等價性判定，從 CJS/ESM 互操作到 Top-level Await 的傳染性處理。這篇文章的技術密度之高，值得用更大的脈絡來重新理解。

今天我們就來深入聊聊：一個打包工具的「鏈接」到底在鏈接什麼？為什麼這個看似無聊的中間步驟，其實是整個打包品質的決定性瓶頸？

---

## 打包不只是「把檔案合在一起」

很多人對打包工具的理解停留在「把一堆 JS 檔案合成一個」。但現實比這複雜太多了。

現代前端專案動輒數百甚至數千個模組，形成錯綜複雜的依賴圖譜。打包工具不能只知道「A 檔案 import 了 B 檔案」，它必須回答更精確的問題：A 檔案裡的 `useState` 和 C 檔案裡的 `useState`，是不是同一個東西？

這個問題聽起來很蠢，但在實務中一點都不簡單。考慮這個場景：

```javascript
// helpers.js
export * from './api.js'
export * from './utils.js'

// main.js
import { source } from './helpers.js'
```

`main.js` 拿到的 `source` 是來自 `api.js` 還是 `utils.js`？如果兩邊都有 `source`，該怎麼辦？如果其中一個是 CommonJS 模組呢？

這就是鏈接階段要解決的問題。它把掃描階段（Scan Stage）產出的粗糙檔案級依賴圖，精煉成符號級的精確映射。做完這一步，後續的 Tree Shaking、Code Splitting 和程式碼生成才有穩固的基礎可以站。

## 三層心智模型：從宏觀到微觀

Rolldown 的鏈接階段可以用三層心智模型來理解，由粗到細：

**第一層：固有屬性傳播。** 計算那些會沿著依賴鏈「傳染」的屬性——Top-level Await（TLA）和副作用（Side Effects）。如果 `api.js` 用了 TLA，那 import 它的 `helpers.js` 也被傳染，import `helpers.js` 的 `main.js` 也逃不掉。Rolldown 用深度優先搜尋（DFS）加記憶化（Memoization）來做這件事，避免重複計算。

**第二層：模組系統標準化。** JavaScript 有 CJS 和 ESM 兩套模組系統，它們天生不互通。在做精確的符號鏈接之前，必須先把這兩套系統的差異抹平。Rolldown 透過 Wrapper 機制來處理——本質上就是用函數閉包模擬目標模組系統的運行環境。

**第三層：符號溯源。** 最細粒度的工作。把每個 import 的符號追溯到它唯一的原始定義。這裡 Rolldown 用了一個經典的資料結構——並查集（Disjoint Set Union, DSU）。

這三層不是隨便分的。它們反映了問題的本質結構：你不可能在搞清楚模組系統之前做符號鏈接，也不可能在標記 TLA 傳染性之前做正確的程式碼生成。順序就是這樣，沒有捷徑。

## CJS 與 ESM 的世紀糾纏

前端圈最持久的痛苦之一，就是 CJS 和 ESM 的共存問題。

CommonJS 是 Node.js 的同步模組系統——`require()` 是阻塞的，`module.exports` 是在運行時動態決定的。ESM 是 ECMAScript 官方標準——`import/export` 是靜態的，設計給編譯期分析用的。這兩套東西從根本哲學上就不一樣。

但現實是殘酷的：你的現代 ESM 專案幾乎一定會依賴某個只提供 CJS 版本的老套件。Rolldown 怎麼處理？

首先，掃描階段會根據語法特徵判定每個模組的 `ExportsKind`（ESM、CommonJS 或 None）。接著，鏈接階段會看 import 方用的是什麼方式（`ImportKind`——靜態 import、動態 import() 還是 require()），再結合被 import 方的 `ExportsKind`，決定需要什麼樣的包裝器（`WrapKind`）。

核心邏輯大致是這樣：

```rust
ImportKind::Require => match importee.exports_kind {
  ExportsKind::Esm => {
    self.metas[importee.idx].wrap_kind = WrapKind::Esm;
  }
  ExportsKind::CommonJs => {
    self.metas[importee.idx].wrap_kind = WrapKind::Cjs;
  }
}
```

包裝器本質上是一個函數閉包，模擬目標模組系統的運行環境。CJS 模組會被包在一個提供 `module.exports` 物件的函數裡，這樣 ESM 模組就能安全地跟它互動。

值得一提的是，Rolldown 在這方面做得比前輩們優雅不少。Rollup 需要靠 `@rollup/plugin-commonjs` 外掛來處理 CJS，esbuild 雖然內建了 CJS 支援但行為有些特殊（例如會把 `require` 轉成 `__require`，導致二次打包困難）。Rolldown 直接內建了完整的 CJS/ESM 互操作支援，而且大致遵循 esbuild 的語意，通過了 esbuild 所有的 CJS/ESM 互操作測試。這不是小事——如果你踩過 CJS interop 的坑，你知道這有多重要。

還有一個容易被忽略的細節：包裝過程是**遞迴的**。當一個 CJS 模組被包裝後，它的行為可能改變（例如引入新的非同步特性），所有依賴它的模組也可能需要重新評估。Rolldown 會沿著整個 import 鏈向上遞迴傳播，確保一致性。

## 並查集：演算法教科書照進現實

鏈接階段最核心的資料結構是並查集（DSU），也叫 Union-Find。如果你上過演算法課，這東西應該不陌生。但看到它被用在打包工具裡，還是挺讓人會心一笑的。

問題本質是這樣的：專案裡有成千上萬個符號引用，其中很多指向同一個原始定義。`main.js` 裡的 `useState`、`featureA.js` 裡的 `useState`、`featureB.js` 裡的 `useState`——它們其實都是 `react` 模組裡那同一個函數。我們需要一個高效的方式來判定和管理這些等價關係。

這正是並查集的拿手好戲。每個符號引用是一個元素，等價的符號會被歸到同一個集合裡，集合的根節點就是這個符號的「標準代表」。

Rolldown 的實作有兩個重點：

**路徑壓縮的 `find_mut`：**

```rust
pub fn find_mut(&mut self, target: SymbolRef) -> SymbolRef {
  let mut canonical = target;
  while let Some(parent) = self.get_mut(canonical).link {
    // 路徑壓縮：把當前節點直接指向祖父節點
    self.get_mut(canonical).link = self.get_mut(parent).link;
    canonical = parent;
  }
  canonical
}
```

每次查找時，順手把經過的節點重新指向更高的祖先，逐步扁平化樹結構。這讓後續查找的時間複雜度趨近於 O(1)——準確說是 O(α(n))，其中 α 是反 Ackermann 函數，對所有實際用途來說就等於常數。

**`link` 合併操作：**

```rust
pub fn link(&mut self, base: SymbolRef, target: SymbolRef) {
  let base_root = self.find_mut(base);
  let target_root = self.find_mut(target);
  if base_root == target_root {
    return; // 已經在同一個集合裡了
  }
  self.get_mut(base_root).link = Some(target_root);
}
```

找到兩邊的根，如果不同就合併。簡潔到沒什麼好說的。

但 Rolldown 的 DSU 不是教科書上的原始版本。它用了 `IndexVec`（帶型別索引的向量）而不是原始陣列，透過 `SymbolId` 和 `ModuleIdx` 這些型別化索引來確保編譯期的型別安全。這是 Rust 生態的一個典型優勢——你在 JavaScript 的打包工具裡很難看到這種等級的型別保證。

為什麼這個選擇很聰明？因為符號解析本質上就是一個等價類問題。你可以用哈希表做，但那樣每次查詢都是 O(1) 但合併是 O(n)；你也可以用圖遍歷做，但那樣每次都要重新走一遍。並查集在這種「大量合併 + 大量查詢」的場景下，攤提時間複雜度接近最優。

## 星號導出：看似方便，實則地獄

`export * from './module'` 這個語法在 JavaScript 裡非常常見，但對打包工具來說是個噩夢。

問題出在歧義。假設：

```javascript
// moduleA.js
export const foo = 1;

// moduleB.js
export const foo = 2;

// barrel.js
export * from './moduleA';
export * from './moduleB';
```

`barrel.js` 到底導出的 `foo` 是哪個？兩個都叫 `foo`，優先權怎麼算？

Rolldown 用一個遞迴 DFS 加回溯的演算法來處理這個問題。核心機制有兩個：

1. **遮蔽（Shadowing）**：模組自己的顯式命名導出永遠優先於星號導出帶進來的同名符號。這符合直覺——你自己寫的東西當然比間接轉手的優先。

2. **歧義檢測**：當同一個名字從多個「優先級相同」的不同來源出現時（例如兩個不同的 `export *`），Rolldown 會把它標記到 `potentially_ambiguous_symbol_refs` 欄位裡，留待後續處理。

最終的歧義消解策略是**保守的**：如果一個導出名稱確實對應到多個不同的原始定義，Rolldown 會直接忽略它，不納入模組的公共 API。寧可報錯，不可猜錯。

這個設計選擇非常務實。esbuild 和 Rollup 在這方面的處理各有細微差異，但 Rolldown 選擇了最安全的路線——在打包工具裡，「意外地選了錯誤的符號」造成的 bug 是極難追蹤的那種。

## 外部模組的特殊處理

當 import 來自外部模組（像 `react`、`lodash`），處理方式完全不同。外部模組不會被打包進來，但打包工具仍然需要確保所有引用同一外部符號的地方是一致的。

Rolldown 用了一個叫 `external_import_binding_merger` 的嵌套哈希映射來處理這件事。它的結構大致是：

```
{
  react_module_idx: {
    "useState": [sym_from_main, sym_from_featureA, sym_from_featureB],
    "useEffect": [sym_from_main, sym_from_componentX]
  }
}
```

所有指向 `react.useState` 的本地符號引用會被收集到同一個集合裡，然後 Rolldown 建立一個「門面符號（facade symbol）」，再用 DSU 的 `link` 操作把所有本地引用都指向這個門面符號。

這個「先收集、後合併」的兩階段策略很巧妙。它避免了在遍歷過程中重複處理同一個外部符號，同時保證了最終的一致性。

## Vite 8 的大局觀：為什麼這一切很重要

理解 Rolldown 鏈接階段的設計，需要放在更大的脈絡裡看。

Vite 之前的架構有一個根本矛盾：開發環境用 esbuild 做轉譯，生產環境用 Rollup 做打包。兩套不同的引擎，不同的模組解析行為，不同的 edge case。結果就是那個經典的痛苦：「開發沒問題，一上 production 就爆」。

Rolldown 的目標就是消除這個矛盾——一個統一的引擎，同時用於開發和生產。Vite 8 Beta 已經實現了這個目標：Rolldown 取代了 esbuild（做為 optimizer）和 Rollup（做為 bundler），搭配 Oxc 做語法轉譯和降級，Lightning CSS 做 CSS 壓縮。

整個工具鏈的三根支柱——Vite、Rolldown、Oxc——都由 VoidZero（尤雨溪的公司）維護，全部用 Rust 寫成。這種垂直整合的程度，在前端工具鏈歷史上是前所未有的。

數字說話：Rolldown 1.0 RC 在 beta 之後累計了 3,400+ commits——749 個新功能、682 個 bug 修復、109 個效能優化。它通過了 900+ Rollup 測試案例和 670+ esbuild 測試案例。在真實世界的案例中，Linear 的建置時間從 46 秒降到 6 秒（快了 7.7 倍），Beehiiv 減少了 64%。

這不是漸進式的改良，這是量級上的跳躍。而鏈接階段的設計品質——精確的符號解析、健壯的 CJS/ESM 互操作、近乎常數時間的等價類查詢——正是支撐這個跳躍的關鍵基礎設施之一。

## 我的觀點

Rolldown 鏈接階段的設計讓我印象深刻的不是什麼花俏的創新，而是**正確的工程選擇**。

並查集不是什麼新東西——它是 1964 年就被提出的資料結構。DFS 更不用說了。但在正確的場景用正確的工具，然後用 Rust 的型別系統把它們組裝得嚴絲合縫——這才是真正的工程功力。

JavaScript 的模組系統是一個歷史遺留的爛攤子，CJS 和 ESM 的共存讓所有打包工具作者都頭疼。Rolldown 沒有假裝這個問題不存在，也沒有用 hacky 的 workaround 繞過去，而是用明確的 `WrapKind` 機制正面解決了它。這比依賴外部 plugin 的方案（看看你，`@rollup/plugin-commonjs`）可靠得多。

對星號導出歧義的保守處理也值得肯定。在打包工具的世界裡，「什麼都不做」有時候是最負責任的選擇。一個猜錯的符號解析，可能導致你的 app 在某個特定的用戶路徑上靜默地讀到了錯誤的值——這種 bug 你可能追蹤幾天都找不到原因。

如果你是用 Vite 的前端開發者，現在就可以開始在 side project 裡試試 Vite 8 Beta。不需要等 1.0 正式版——RC 的 API 已經凍結，不會再有 breaking change。但生產環境？再等等。讓大型專案先踩完坑，RC 到 stable 這段路通常還有不少驚喜。

如果你是對打包工具底層感興趣的工程師，我強烈推薦去讀 Atriiy 的原文，然後 clone Rolldown 的 repo 用斷點跑一遍示範專案。理論是一回事，看著 `find_mut` 在斷點裡一步步壓縮路徑，那感覺完全不一樣。

---

## 延伸閱讀

- [Rolldown 工作原理解析：符號關聯、CJS/ESM 模組解析與導出分析](https://www.atriiy.dev/blog/rolldown-link-stage-symbol-linking-resolution) — 本文的起點，Atriiy 對 Rolldown 鏈接階段的完整技術拆解
- [Announcing Rolldown 1.0 RC](https://voidzero.dev/posts/announcing-rolldown-rc) — VoidZero 官方的 RC 公告，包含效能數據和功能清單
- [Vite 8 Beta: The Rolldown-powered Vite](https://vite.dev/blog/announcing-vite8-beta) — Vite 8 Beta 公告，詳細說明 Rolldown 如何整合進 Vite
- [Rolldown 官方文件](https://rolldown.rs/guide/introduction) — 如果你想開始使用 Rolldown，從這裡開始
- [ESM-CJS Interop Test](https://sokra.github.io/interop-test/) — Tobias Koppers（Webpack 作者）維護的 CJS/ESM 互操作測試套件，值得了解各打包工具的行為差異
