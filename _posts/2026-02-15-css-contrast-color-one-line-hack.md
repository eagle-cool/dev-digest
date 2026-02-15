---
title: "一行 CSS 搞定自動對比色：contrast-color() 等不到，我們自己來"
date: 2026-02-15
description: "CSS Color Level 6 的 contrast-color() 遙遙無期，但用 oklch() 加 round() 就能一行搞定黑白文字自動切換。從 WCAG 到 APCA、從色彩空間到感知亮度，深度解析這個優雅 hack 背後的色彩科學。"
tags: [deep-dive, frontend, css]
---

根據背景色自動決定文字該用黑色還是白色——這件事聽起來簡單到不行，任何一個做過 design system 的人都遇過。但 CSS 規範折騰了五年，到今天還沒給我們一個跨瀏覽器能用的方案。

好消息是，現在有一行 CSS 能做到：

```css
color: oklch(from var(--bg) round(1.21 - L) 0 0);
```

這行看起來像黑魔法的程式碼，背後是色彩科學、無障礙標準演進、以及 CSS 新特性的巧妙組合。CSS-Tricks 最近一篇文章完整拆解了這個技巧，而我想帶你看得更深——不只是「怎麼用」，而是「為什麼這樣就能用」，以及這件事反映了 CSS 生態正在發生什麼變化。

---

## contrast-color()：一個五年磨不出的規範

故事要從 CSS 規範說起。CSS Working Group 很早就意識到「自動對比色」是個剛需。最初的提案叫 `color-contrast()`，出現在 CSS Color Module Level 5 的草案裡，語法長這樣：

```css
color: color-contrast(purple vs white, black);
```

意思是：拿 `purple` 去跟 `white` 和 `black` 比對比度，哪個高就用哪個。簡潔、直覺、完美。

Safari 在 2021 年 3 月的 Technology Preview 122 就實作了這個功能，WebKit 團隊甚至寫了 blog 介紹。Firefox 也跟進了。但 Chrome？沒有。到了 2024 年，規範名稱改成了 `contrast-color()`，語法也簡化了：

```css
color: contrast-color(purple);
```

不再讓你指定候選色了——就是黑或白，瀏覽器自己算。聽起來功能縮水了？其實是規範團隊的策略：先把最簡單的 case 定下來，避免日後換演算法時破壞既有網站。

但即便簡化成這樣，截至 2026 年 2 月，`contrast-color()` 的規範已經被推遲到 CSS Color Module Level 6，而且 Level 6 的 Editor's Draft 明確標註了 "Not Ready For Implementation"。換句話說——**這東西短期內不會有跨瀏覽器支援**。

為什麼這麼難產？因為核心問題不在語法，而在「對比度到底該怎麼算」。

---

## WCAG 的對比度公式：老了，但還在

目前 `contrast-color()` 的規範依賴的是 WCAG 2.1 的對比度演算法。這個演算法的核心是計算兩個顏色的相對亮度（luminance），然後套公式：

```
contrast ratio = (L1 + 0.05) / (L2 + 0.05)
```

其中 L1 是較亮色的亮度，L2 是較暗色。亮度範圍 0 到 1，所以對比度從 1（完全相同）到 21（純黑配純白）。

看起來數學上很乾淨，但實際用起來問題一堆。

WCAG 的亮度計算基於 sRGB 色彩空間的線性化轉換，公式涉及 gamma 校正：

```
L = 0.2126 × R_linear + 0.7152 × G_linear + 0.0722 × B_linear
```

這個公式反映的是「物理光的亮度」，不是「人眼感知的亮度」。差別在哪？想想看：你把一個深藍色 `#407ac2` 配上黑字和白字，WCAG 說黑字對比度比較高（4.70 vs 4.3）。但你實際看那個畫面，白字明顯更好讀。

這不是個案。WCAG 2.x 的對比度公式在中間色調（mid-tones）和深色背景上的表現特別差，它會系統性地高估深色系的對比度。如果你做過 dark mode 的 design system，應該對這個問題深有體會——按照 WCAG 規範選出來的顏色，有時候人眼看根本讀不清楚。

---

## APCA：下一代對比度標準，但還在路上

這就要提到 APCA（Advanced Perceptual Contrast Algorithm）了。APCA 是 Myndex 的 Andrew Somers 開發的新一代對比度演算法，被認為是 WCAG 3.0 的候選方案。

APCA 和 WCAG 2.x 的根本差異在於**感知模型**。WCAG 用的是物理亮度比，APCA 用的是基於人類視覺系統的空間對比敏感度曲線（Spatial Contrast Sensitivity Function）。這意味著 APCA 會考慮：

- **極性（Polarity）**：深底淺字和淺底深字，即使物理對比度相同，人眼感知是不一樣的。APCA 的 Lc 值用正負號表示極性——正值是深字淺底，負值是淺字深底。
- **字體大小和粗細**：小字、細字需要更高的對比度才能達到同等可讀性。APCA 針對不同字級有不同的門檻值。
- **感知均勻性**：APCA 的 Lc 值是感知均勻的——Lc 60 到 Lc 30 的視覺差異，和 Lc 90 到 Lc 60 的視覺差異，在人眼看來是一致的。WCAG 的 4.5:1 到 3:1 的差距和 7:1 到 4.5:1 的差距，在視覺上完全不等價。

APCA 的 Lc 門檻值也比 WCAG 的簡單規則精細得多：

| Lc 值 | 適用場景 |
|-------|---------|
| Lc 90 | 正文首選（14px/400+） |
| Lc 75 | 正文最低限度（18px/400+） |
| Lc 60 | 內容文字最低限度（24px 或 16px bold） |
| Lc 45 | 標題、圖示 |
| Lc 30 | placeholder、disabled |
| Lc 15 | 幾乎不可見的裝飾元素 |

APCA 原本預計會在 WCAG 3.0 中正式納入，但截至 2025 年 8 月的 WCAG 3.0 草案中，對比度要求的章節仍然是空的。規範的車輪轉得比任何人預期的都慢。

而這正是 `contrast-color()` 難產的核心原因：**規範團隊不想鎖定 WCAG 2.x 的演算法，因為 APCA 明顯更好，但 APCA 又還沒正式標準化**。WebKit 團隊在 blog 中也坦承：目前 Safari 的實作用的是 WCAG 2 演算法，但未來一定會換。

所以我們卡在這裡——規範在等標準，標準在等共識，開發者在等瀏覽器。

---

## 一行 CSS 的巧思：用 OKLCH 繞過所有這些

回到那行魔法 CSS：

```css
color: oklch(from var(--bg) round(1.21 - L) 0 0);
```

這行程式碼完全繞開了 WCAG 和 APCA 的公式，改用一個更優雅的策略：**直接利用色彩空間的感知亮度分量來做判斷**。

這裡的關鍵是 **OKLCH**。

### OKLCH 是什麼？為什麼選它？

OKLCH 是 Björn Ottosson 在 2020 年提出的色彩空間，它是 OKLab 的圓柱座標版本。三個分量是：

- **L（Lightness）**：感知亮度，0 到 1
- **C（Chroma）**：彩度
- **H（Hue）**：色相角度

OKLCH 最大的賣點是**感知均勻性**——L 值的變化和人眼感知到的亮度變化成正比。這跟 HSL 的 L 完全不同。HSL 的 `hsl(60, 100%, 50%)` 是黃色，`hsl(240, 100%, 50%)` 是藍色，兩個的 L 值一樣是 50%，但黃色看起來比藍色亮得多。OKLCH 不會有這個問題。

OKLCH 也優於它的前身 LCH（基於 CIELab）。LCH 的感知均勻性在藍紫色區域有明顯偏差——所謂的「紫色偏移」問題。Ottosson 的 OKLab/OKLCH 修正了這個問題，在整個色域範圍內都維持了更好的均勻性。

CSS-Tricks 文章的作者實際測試發現，用 LCH 做自動對比色切換時，「太暗不適合黑字、太亮不適合白字」的灰色地帶（也就是黑字和白字對比度都不夠理想的區間）在 LCH 中是 63 到 70，但在 OKLCH 中是 0.70 到 0.77——區間更窄、更集中，意味著用 OKLCH 做二元判斷更準確。

### round() 的妙用

到 2026 年，`oklch()` 搭配 relative color syntax 已經是全瀏覽器支援的特性了。語法是：

```css
oklch(from <origin-color> <L> <C> <H>)
```

`from` 關鍵字讓你可以取得原始顏色的 L、C、H 分量，然後做數學運算。所以 `oklch(from purple L C H)` 就是 `purple` 本身——你拿到了它的分量但什麼都沒改。

接下來就是 `round()` 的巧思了。CSS `round()` 函式做的是四捨五入到最近的整數（預設步長 1）。所以：

```
round(1.21 - L)
```

當 L = 0.72 時：`1.21 - 0.72 = 0.49`，`round(0.49) = 0`（黑色）
當 L = 0.71 時：`1.21 - 0.71 = 0.50`，`round(0.50) = 1`（白色）

0.72 就是那個切換點。作者通過大量計算（用 Color.js 搭配 APCA 演算法跑了所有可能的色彩組合）找到的最佳閾值——OKLCH 亮度 0.72 以上的顏色，黑字永遠比白字對比度高；0.65 以下的顏色，白字永遠比黑字對比度高；中間的灰色地帶，兩者都有 45-60 的 APCA 對比度，怎麼選都還行。

最後把 C 和 H 都設成 0，就得到了無彩色——純黑或純白。

一行。沒有 JavaScript。沒有 CSS 變數體操。跨瀏覽器。

---

## 更進一步：不只是黑與白

原文還展示了一個進階技巧——如果你不想在白色和黑色之間切換，而是想在白色和你的 base text color 之間切換呢？

```css
--white-or-black: oklch(from var(--bg) round(1.21 - L) 0 0);
color: rgb(
  from color-mix(in srgb, var(--white-or-black), var(--base-color))
  calc(2*r) calc(2*g) calc(2*b)
);
```

這裡用了 `color-mix()` 的一個數學特性：

- 當 `--white-or-black` 是白色（255, 255, 255）時，混合後每個通道至少是 127.5，乘以 2 就是 255——結果還是白色。
- 當 `--white-or-black` 是黑色（0, 0, 0）時，混合後每個通道是 base color 的一半，乘以 2 就回到 base color 的原始值。

巧妙。但這個技巧有個限制：Safari 18 以下不支援在 `rgb()` 的 relative color syntax 中使用 `color-mix()` 的結果。所以實務上需要搭配 fallback。

至於 CSS Custom Functions（`@function`），原文也示範了用它來封裝這些邏輯：

```css
@function --white-black(--color) {
  result: oklch(from var(--color) round(1.21 - l) 0 0);
}
```

但截至 2026 年 2 月，`@function` 只在 Chrome 141+ 支援。Firefox 和 Safari 都還沒跟上。所以這更像是一個「未來可以這樣寫」的展示，而不是今天就能部署的方案。

---

## 這件事告訴我們什麼

表面上這是一個 CSS 技巧，但如果你往後退一步看，這裡有幾個值得思考的趨勢：

**CSS 正在變成一門真正的程式語言。** `round()`、`pow()`、`from` 關鍵字、relative color syntax——這些不是「樣式」，這是數學運算。CSS 曾經被嘲笑「不是程式語言」，但現在你可以在 CSS 裡做 gamma 校正、感知亮度計算、條件分支（透過 `round()` 實現的二元判斷）。`@function` 的加入更是直接補上了最後一塊拼圖。

**色彩空間不再是學術概念。** 五年前跟前端工程師聊 OKLCH，大概會被當成在賣弄學問。但今天 `oklch()` 是全瀏覽器支援的 production-ready 特性，而且它解決了 HSL 用了二十年都解決不了的問題——感知均勻性。理解色彩空間不再是「nice to have」，而是寫出正確 CSS 的基本功。

**規範趕不上實踐。** `contrast-color()` 的故事就是一個典型案例。規範團隊有它的考量——他們不想鎖定一個可能很快被淘汰的演算法。但開發者的需求不會等。於是就出現了這種「用現有特性逼近未來規範」的 hack。這不是第一次（記得 flexbox 之前的 float hack 嗎？），也不會是最後一次。

---

## 你現在該做什麼

如果你的 design system 有「根據背景色自動切換文字顏色」的需求：

1. **直接用那行 CSS。** `oklch(from var(--bg) round(1.21 - L) 0 0)` 在所有現代瀏覽器都能跑，沒有 JavaScript 依賴，效能零成本。

2. **閾值可以調。** 0.72（也就是 `1.21 - L` 中的 1.21）是基於 APCA 計算的最佳值，但如果你的使用場景偏向某個色域，你可以自己測試調整。

3. **別忘了實際檢查。** 自動對比色只是起點，不是終點。它能保證你不會出現白底白字的災難，但 WCAG AA（4.5:1）或 APCA Lc 75 的達標與否，還是需要人眼和工具確認。

4. **追蹤 `contrast-color()` 的進度。** 總有一天它會落地，到時候一行 `contrast-color(var(--bg))` 就搞定了，比什麼 hack 都乾淨。但不是今天。

5. **學 OKLCH。** 這個色彩空間會越來越重要。無論是 design token、主題系統、還是動態配色，OKLCH 的感知均勻性都讓它成為最適合做數學運算的色彩空間。花一個下午搞懂 L、C、H 三個軸的意義，絕對值回票價。

---

## 延伸閱讀

- [Approximating contrast-color() With Other CSS Features](https://css-tricks.com/approximating-contrast-color-with-other-css-features/) — 本文的起點，完整的程式碼和 CodePen demo
- [How to have the browser pick a contrasting color in CSS](https://webkit.org/blog/16929/contrast-color/) — WebKit 團隊對 contrast-color() 的官方說明
- [The Easy Intro to the APCA Contrast Method](https://git.apcacontrast.com/documentation/APCAeasyIntro.html) — 想搞懂 APCA 看這篇就夠了
- [CSS Color Module Level 6 Draft](https://drafts.csswg.org/css-color-6/) — contrast-color() 規範的最新草案（注意標題寫著 "Not Ready For Implementation"）
- [OKLCH in CSS: why we moved from RGB and HSL](https://evilmartians.com/chronicles/oklch-in-css-why-quit-rgb-hsl) — Evil Martians 的經典文章，解釋為什麼 OKLCH 是更好的色彩空間
