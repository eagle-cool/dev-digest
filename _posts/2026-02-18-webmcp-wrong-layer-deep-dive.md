---
title: "WebMCP 深度解析：當 Google 和 Microsoft 試圖在錯誤的層級解決正確的問題"
date: 2026-02-18
description: "Chrome 146 搶先預覽的 WebMCP 要讓每個網站變成 AI Agent 的工具介面，但這真的是正確的架構嗎？從 HATEOAS 的被遺忘、AMP 的前車之鑑、到 Accessibility Tree 的現有能力，深度拆解為什麼 WebMCP 可能是 Web 標準史上又一次方向錯誤。"
tags: [deep-dive, frontend, systems]
---

Chrome 146 Canary 剛拿到一個新 API：`navigator.modelContext`。Google 和 Microsoft 的工程師聯手提案，要讓每個網站都能對 AI Agent 暴露結構化的「工具合約」——你的登入表單、搜尋功能、結帳流程，全部變成 Agent 可以直接呼叫的函式。

這個提案叫 WebMCP（Web Model Context Protocol），已經進入 W3C Web Machine Learning 社群群組孵化。早期試驗數據很漂亮：token 用量降低 67.6%、延遲改善 25-37%、任務成功率 97.9%。如果你是 PM，這些數字足以讓你在下次會議上拍桌子說「我們要支援這個」。

但如果你是寫了十年前端的工程師，你會先問一個問題：**這層東西，為什麼要放在瀏覽器裡？**

---

## 問題是真的，方向可能是錯的

先把立場講清楚：AI Agent 將會以遠超人類的流量消費網頁，為此做最佳化是正確的投資。這一點毫無爭議。

爭議在於**怎麼做**。

WebMCP 的提案有兩種 API：

**宣告式（Declarative）**——在既有的 HTML `<form>` 上加標記，瀏覽器自動把表單欄位轉成工具 schema：

```html
<form toolname="search-flights"
      tooldescription="Search available flights">
  <input name="origin" type="text" required>
  <input name="destination" type="text" required>
  <input name="date" type="date" required>
  <button type="submit">Search</button>
</form>
```

**命令式（Imperative）**——用 JavaScript 註冊工具合約：

```javascript
navigator.modelContext.registerTool({
  name: 'search-flights',
  description: 'Search available flights',
  inputSchema: {
    type: 'object',
    properties: {
      origin: { type: 'string' },
      destination: { type: 'string' },
      date: { type: 'string', format: 'date' }
    },
    required: ['origin', 'destination', 'date']
  },
  execute: async (params) => {
    const results = await searchFlights(params);
    return { flights: results };
  }
});
```

看起來很合理，對吧？但停下來想一秒——那個 `execute` 裡呼叫的 `searchFlights`，最終不還是打一個 API 到後端？Agent 透過瀏覽器這個中間人讀一份「工具說明書」，然後觸發一個最終還是跑在伺服器上的動作。

為什麼不直接讓 Agent 和伺服器對話？

---

## 三條路，WebMCP 選了最尷尬的那條

讓 Web 對 AI Agent 友善，本質上有三條路：

**路線一：伺服器端 MCP。** Anthropic 在 2024 年底推出的 Model Context Protocol 已經是事實標準。伺服器自己暴露工具——Postgres MCP server 知道自己的 schema、Stripe MCP server 知道自己的 API。工具合約和工具本身是同一個東西，由同一個團隊維護，住在同一個 codebase 裡。Agent 直接跟資料的源頭對話，沒有中間人。

**路線二：瀏覽器自己變聰明。** 瀏覽器已經知道很多東西了——HTML 結構、ARIA 語意、Schema.org 標記、表單 label、連結關係。把這些資訊合成一個更豐富的機器可讀層級，開發者不用做任何額外工作。瀏覽器更新一次，全世界的網站同時受益。

**路線三：WebMCP。** 每個網站開發者都要建置並維護一份瀏覽器端的工具合約。瀏覽器只是一個被動的管道。

WebMCP 是路線三，而它卡在兩頭不到岸的尷尬位置。

願意投資的網站（SaaS 平台、企業工具、API-first 的公司）應該走路線一。伺服器端 MCP 已經存在且運作良好，Agent 拿到的是 source of truth——真正的 API、真正的商業邏輯、真正的驗證規則。WebMCP 給 Agent 的是這些東西的一份**副本**，獨立撰寫、獨立維護，而且 UI 一改就可能和現實脫節。

不願意投資的網站（部落格、小商家、新聞網站、個人頁面）不會為了 AI Agent 建置任何東西。二十年的 metadata 採用史已經證明了這一點。這些網站更適合路線二——讓瀏覽器自己讀懂它們。

WebMCP 要求路線一等級的工程投入，但交付的是路線一的降級版。它聲稱服務路線二的世界，但要求的正是路線二的世界從未交付過的那種採用率。

---

## 這個劇本我們看過：HTML Form 從 1993 年就是機器可讀的

這裡有個讓人哭笑不得的歷史脈絡。

REST 架構的核心原則之一是 HATEOAS（Hypermedia As The Engine Of Application State）——簡單說就是：**伺服器的回應本身就告訴你接下來能做什麼**。一個 HTML 頁面上的 `<form>` 元素，帶著 `action`、`method`、每個 `<input>` 的 `name` 和 `type`——這就是一份結構化的工具宣告。1993 年就有了。

HTML 表單說：「這裡有一個動作，這些是它需要的輸入，送到這個 URL 處理。」WebMCP 的宣告式 API 說的，是**一模一樣的事情**，只是換了一組屬性名稱。

那為什麼 AI Agent 不直接讀 HTML 表單？因為業界花了二十年把 Web 變成了一堆 JavaScript SPA，表單提交變成了 `onClick` handler 裡的 `fetch()` 呼叫，語意結構被 CSS-in-JS 和 Virtual DOM 層層包裹。我們自己把 hypermedia 的機器可讀性搞壞了，現在再發明一個新標準來修補。

有人在 DEBEDb 的技術部落格上寫了一篇尖銳的評論，標題就叫〈WebMCP: A Solution In Search of the Problem It Created〉。核心論點很簡單：問題是產業自己創造的，不是靠疊加另一層抽象就能解決的。

這個觀點有道理，但也不完全公平。現實是 SPA 已經佔領了 Web，你不可能叫全世界退回到 server-rendered hypermedia。在這個前提下，某種形式的結構化介面確實是必要的——但那個介面該放在哪一層，才是關鍵問題。

---

## AMP 的幽靈：「加法式」標準的隱形成本

原文做了一個 AMP 類比，值得深挖。

Google AMP 在 2015 年推出，透過 Google 搜尋結果的 Top Stories 輪播位給予優先展示——本質上是脅迫式採用。出版商最終紛紛放棄 AMP，報告顯示退出後廣告收入反而**提升**。2023 年 Google 正式取消 AMP 作為排名因素。整個故事從「這是開放標準」到「其實是 Google 控制 Web 的工具」到「廢棄」，花了八年。

WebMCP 的支持者會說：「AMP 是替代架構，要求重建頁面；WebMCP 是加法式的，你保留原有網站，只是在上面疊一層工具合約。」

但這個「加法式」的框架是有誤導性的。AMP 的成本是前置且可見的——你知道自己在付什麼代價，因為你在重建頁面。WebMCP 的成本是持續且隱形的。工具合約必須永久跟 UI 保持同步，而失敗模式是**靜默漂移**而非明顯崩壞。

想像一個電商網站改了結帳流程——加了一步地址驗證、改了付款選項的順序。UI 團隊更新了 React 組件，QA 測試了使用者流程，部署完成。但誰記得去更新 WebMCP 的工具合約？沒有 build step 會抓到這個漂移，沒有 CI check 會標記不一致。Agent 繼續使用過時的合約，執行了一個不存在的動作，然後靜靜地失敗。

Stripe 管理超過 100 個破壞性 API 升級，靠的是一套自建的 DSL 來從程式碼自動生成文件。如果一家**以販賣 API 為生**的公司都需要重度自動化才能防止 metadata 腐爛，你覺得一般的新創公司有多少機率保持 WebMCP 工具定義的準確性？

---

## Accessibility Tree：那個已經存在的機器可讀層

WebMCP 暗示 AI Agent 需要一個根本不同於人類的介面。這個前提大部分是錯的。

瀏覽器已經為每個頁面生成了一個機器可讀的模型——Accessibility Tree（AX Tree）。它提供角色、名稱、狀態和互動模式。Playwright 和 Puppeteer 這些自動化框架已經透過 AX Tree 快照來驅動 Agent。

而且 multimodal model 越來越擅長直接「看」網頁。GPT-4o、Claude、Gemini 已經能純靠截圖導航許多網站。如果這個趨勢持續，不管是 WebMCP 還是 AX Tree，任何結構化介面的需求都會隨時間遞減。

但結構化介面仍然重要，原因是三個：

1. **可靠性**——視覺 Agent 會幻覺元素位置
2. **確定性**——同樣的 AX Tree 輸入產生同樣的 Agent 行為
3. **成本效率**——解析結構化樹比處理截圖便宜幾個數量級

關鍵差異在於：AX Tree **已經存在**。它的維護成本是零，因為瀏覽器自動生成。WebMCP 需要持續的人工投入，而改善中的 model 可能最終讓這份投入變得多餘。如果你要押注一個結構化層，押那個免費的。

當然，AX Tree 有真實的缺口。Shadow DOM 封裝、Canvas 結構、虛擬列表——這些場景下 AX Tree 確實不夠用。但這些是**平台層級的問題，需要平台層級的修復**：

- **ElementInternals**——2024 年已在 Chrome、Edge、Firefox、Safari 全面支援，讓 Web Components 原生參與 AX Tree
- **WebDriver BiDi**——W3C 標準的跨瀏覽器自動化協定，引入標準化的 accessibility locator，2026 年初已在 Firefox、Chrome、Edge 出貨
- **Accessibility Object Model（AOM）**——WICG 提案，給 JavaScript 直接存取和修改 AX Tree 的能力。部分 spec 已出貨，完整版仍在推進中

這些技術改善瀏覽器**讀取既有內容**的能力。一次瀏覽器更新，全球每個網站同時受益。WebMCP 則需要每個網站個別投入，而且覆蓋率永遠是漸進式的。

---

## 安全的盲點：你在廣播自己的攻擊面

有一個容易被忽略的安全維度。

WebMCP 工具合約本質上是**送給不受信任客戶端的 API 文件**。它告訴每個來訪的 Agent：這裡有哪些動作可用、接受什麼參數、什麼狀態轉換是合法的。這是一張應用程式攻擊面的地圖。

一份過時的合約可能暴露已經應該被淘汰的 deprecated endpoint。一份被竄改的合約可能引導 Agent 代替使用者執行非預期的動作。WebMCP 自己的 spec 也承認存在安全疑慮——但措辭是「目前沒有人有完整的答案」，同時 API 已經在 Chrome Canary 中可用。

`destructiveHint` 這個標記？它是「建議性的，不是強制的」。也就是說，一個工具把自己標記為「破壞性操作」，但沒有任何機制阻止 Agent 無視這個標記就執行。

AX Tree 避免了這個問題，因為它是瀏覽器從 live DOM 生成的，不是一份獨立撰寫、可能被竄改或失同步的產物。

---

## 採用率的冷酷算術

Web metadata 標準的採用史是一面清楚的鏡子：

| 標準 | 2026 年採用率 | 真正驅動採用的因素 |
|------|-------------|----------------|
| Microformats (2005) | ~0.5% | 沒有激勵 |
| RDFa (2008) | ~39% | Open Graph（社群分享卡片）|
| Microdata (2011) | ~23% | Google SEO |
| JSON-LD (2011) | ~53% | Google Rich Snippets |
| Open Graph (2010) | ~70% | 社群媒體卡片 |

成功的標準有三個共通點：**簡單、即時可見的回報、幾乎零維護成本**。

`robots.txt` 解決的是開發者自己的問題（伺服器被爬蟲打爛），一個純文字檔，寫一次就忘。Open Graph 的回報是即時的——你把連結貼到 Slack，馬上看到漂亮的預覽卡片。

WebMCP 三項全敗。不簡單（工具合約要隨 UI 持續更新）。沒有可見回報（沒有「Agent Rich Snippet」讓開發者立刻看到效果）。維護成本高（合約必須和 UI 同步，否則變成負債）。

更殘酷的是 Goodhart's Law 效應。分析 Schema.org 的使用數據，61.99% 使用 Product schema 的網站只填了 `name` 和 `description`——因為那是 Google Rich Snippet 唯一獎勵的兩個欄位。開發者永遠實作最小可行版本。WebMCP 沒有對應的激勵迴路。

即便是「成功」的 ARIA，WebAIM 的年度調查發現使用 ARIA 屬性的頁面平均有 **57 個** accessibility 錯誤，而沒用 ARIA 的頁面只有 27 個。不是 ARIA 造成錯誤，而是即使有二十年的倡導、文件和瀏覽器支援，Web 規模的 metadata 維護品質依然慘不忍睹。WebMCP 會進入同樣的環境，面對同樣的結構性劣勢，而且背後的資源更少。

---

## 碎片化的陰影：又是 Chromium-only？

截至 2026 年 2 月，Mozilla 和 Apple 對 WebMCP 沒有公開表態支援。Safari 和 Firefox 都在參與 working group，但沒有出貨實作。

如果只有 Chromium 系瀏覽器支援 WebMCP，Agent 只能在 Chrome 和 Edge 裡可靠運作。這不是開放 Web 標準，這是 Chromium 功能。

而且 WebMCP 創造了一個雙層系統：有能力維護工具合約的大企業（Salesforce、Amazon）和沒能力的長尾網站。一個人的部落格和一家兆級企業原本遵守同一套 HTML 規則。WebMCP 打破了這個契約。

AMP 的教訓應該夠清楚了。Google 用搜尋排名做為棍子和胡蘿蔔，強迫出版商採用一個最終被證明對出版商有害的技術。WebMCP 的棍子和胡蘿蔔是什麼？目前看不到。這可能反而是好事——沒有脅迫式採用就沒有 AMP 式的傷害。但沒有激勵也意味著沒有採用率。

---

## 最難的場景：即使如此，伺服器端 MCP 還是更好

公平地說，確實存在 AX Tree 力有未逮的場景：

**多步驟表單精靈加條件邏輯。** 保險理賠填寫：第三步的欄位取決於第一步的選擇，驗證規則隨理賠類型改變。AX Tree 看到的是一堆扁平的表單控件，它不編碼它們之間的條件關係。

**互相依賴的 Dashboard 控件。** BI 工具裡改日期範圍會改變可用的指標欄，選資料源會重組整個視覺化面板。Agent 讀 AX Tree 看到的是當下狀態，看不到狀態機。

**跨欄位交叉驗證的複雜資料輸入。** ERP 庫存管理，SKU 觸發倉庫可用性查詢，數量必須在供應商特定閾值內。AX Tree 能告訴你 Submit 按鈕是 disabled 的，但解釋不了**為什麼**。

這些是 WebMCP 最強的 use case。但每一個場景，**伺服器端 MCP 都做得更好**。Salesforce admin panel 本來就有 API。ERP 系統本來就有後端邏輯定義合法的狀態轉換。保險理賠流程本來就有伺服器端的驗證規則。Agent 不需要讀瀏覽器裡的一份標注——它可以直接和系統對話。

伺服器端 MCP 給 Agent 的是 source of truth：真正的商業邏輯、真正的驗證規則、真正的狀態機。WebMCP 給 Agent 的是這些東西的副本，另外寫的，另外維護的，隨時可能和它描述的現實脫節。

WebMCP 的 benchmark 拿來比較的基線是原始 DOM 爬取和截圖解析——這是最差的情況。Playwright 的 AX Tree 快照已經剝離了視覺噪音，給 Agent 一棵精簡的結構化角色/名稱/狀態樹。而伺服器端 MCP 是最高效的——Agent 拿到的是直接的 API response，只有它需要的資料，零瀏覽器開銷。真正公平的比較——「WebMCP vs. 良好實作的 AX Tree」和「WebMCP vs. 伺服器端 MCP」——還沒有被發表。

---

## 那開發者現在該做什麼

兩個問題問自己就夠了。

**第一：你的 HTML、ARIA 和 Schema.org 做對了嗎？** 對大多數組織來說，答案是沒有。修好這些東西，你同時得到 accessibility 改善、SEO 提升、以及隨著瀏覽器 API 進化而來的 Agent 可讀性——一份投入，三份回報。

**第二：如果你準備進一步投資，Agent 介面該建在哪裡？** 建在你擁有工具的伺服器上，還是建在瀏覽器裡變成一份副本？答案不言自明。

WebMCP 解決的問題是真實的。AI Agent 確實需要更好的方式與 Web 互動。但這個提案選錯了層級。它要求的投入等級和伺服器端 MCP 一樣，交付的卻是降級版本。它期望的採用率是過去二十年沒有任何 Web metadata 標準達到過的。它目前只有 Chromium 支援，而 Web 平台的碎片化是我們承受不起的。

正確的路：對願意投資的組織，推 server-side MCP；對其他所有人，讓瀏覽器變得更聰明。兩條路，中間那條泥濘的小徑——那就是 WebMCP 站的位置。

---

## 延伸閱讀

- [The WebMCP False Economy: Why We Don't Need Another Layer of Abstraction](https://dev.to/manveer_chawla_64a7283d5a/the-webmcp-false-economy-why-we-dont-need-another-layer-of-abstraction-566e) — 本文分析的起點，對 WebMCP 架構缺陷的系統性批評
- [WebMCP is available for early preview — Chrome for Developers](https://developer.chrome.com/blog/webmcp-epp) — Google 官方的 WebMCP 早期預覽公告
- [WebMCP: A Solution In Search of the Problem It Created](https://blog.debedb.com/2026/02/14/webmcp-a-solution-in-search-of-the-problem-it-created/) — 從 HATEOAS 角度切入的尖銳批評，指出 HTML 表單本身就是結構化工具宣告
- [Google Chrome ships WebMCP in early preview — VentureBeat](https://venturebeat.com/infrastructure/google-chrome-ships-webmcp-in-early-preview-turning-every-website-into-a) — 業界對 WebMCP 的多角度報導
- [Model Context Protocol — Anthropic](https://modelcontextprotocol.io/) — 伺服器端 MCP 的官方文件，理解 WebMCP 的對照參考
- [WebAIM Million — Annual Accessibility Analysis](https://webaim.org/projects/million/) — Web 規模的 accessibility metadata 品質現況，理解為什麼開發者維護的 metadata 總是品質堪憂
