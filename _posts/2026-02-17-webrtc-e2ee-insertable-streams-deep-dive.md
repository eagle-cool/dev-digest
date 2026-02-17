---
title: "WebRTC 的加密是假的？Insertable Streams 如何讓瀏覽器實現真正的端對端加密"
date: 2026-02-17
description: "深入解析 WebRTC Encoded Transform API（Insertable Streams）的運作原理——為什麼標準 DTLS-SRTP 加密在 SFU 架構下形同虛設，瀏覽器如何在 encoder 和 packetizer 之間插入 AES-GCM 加密層，以及 SFrame（RFC 9605）標準化之後的產業現狀與實作取捨。"
tags: [deep-dive, frontend, security]
---

你以為 WebRTC 的通話是加密的？技術上沒錯，但那個「加密」可能跟你想的不一樣。

大部分使用 SFU（Selective Forwarding Unit）架構的視訊會議系統——包括你每天用的那些——都有一個共同的尷尬事實：你的音視訊資料在伺服器上是**明文**的。不是可能，是一定。DTLS-SRTP 保護的是傳輸通道，不是你說的話。

這就像把信件放進上鎖的信箱寄出，但郵局在分揀的時候會打開來看一眼再放進另一個鎖好的信箱轉寄。信箱確實有鎖，但郵局看過你寫了什麼。

2024 年 8 月，IETF 正式發布了 RFC 9605（SFrame），2025 年 10 月所有主流瀏覽器都支援了 WebRTC Encoded Transform API。瀏覽器終於有了在 SFU 架構下實現真正端對端加密的標準化能力。這是 Web Platform 安全能力的一次重要躍進，值得好好拆解。

---

## SFU 架構的「加密假象」

要理解為什麼需要新的 API，得先看清楚現有架構的問題。

早期 WebRTC 用的是 P2P Mesh 拓撲——每個參與者直接連接其他所有人。這種架構天然支援端對端加密（DTLS-SRTP 的密鑰只有兩端知道），但可擴展性是災難級的：5 人通話需要 10 條連線，10 人就要 45 條。你的筆電不會答應。

所以業界轉向了 SFU 架構。SFU 是一台中心伺服器，所有參與者把媒體串流送到 SFU，由 SFU 決定把誰的串流轉發給誰。Janus、Mediasoup、LiveKit——主流的開源媒體伺服器都是這個模型。

問題出在 DTLS-SRTP 的工作方式。它是 **hop-by-hop** 加密，不是 end-to-end：

1. Client A 用和 SFU 協商的密鑰加密媒體
2. SFU **解密**成明文，讀取 RTP header、改寫 sequence number、做 simulcast 層選擇
3. SFU 用和 Client B 協商的**另一組密鑰**重新加密
4. Client B 解密

看到問題了嗎？SFU 是一個 **Privileged Decryption Point**（特權解密節點）。它手上有所有人的密鑰，它記憶體裡流過所有人的明文音視訊。如果 SFU 被入侵——無論是惡意內部人員、零日漏洞、還是政府的合法攔截令——你的通話內容就是裸奔的。

這不是理論風險。2020 年 Zoom 被踢爆聲稱提供「端對端加密」但實際上密鑰在伺服器端可存取，引發了一場信任危機和 FTC 調查。那之後，「真正的 E2EE」從行銷話術變成了必須解決的工程問題。

---

## Encoded Transform API：在正確的位置動手術

解決方案的核心思路很單純：在 SFU 的加密層**之外**再加一層只有通話雙方知道密鑰的加密。SFU 看到的是密文，它可以正常轉發封包（因為 RTP header 還是明的），但媒體內容對它來說就是一堆高熵噪音。

但要在哪裡加這層加密？WebRTC 的媒體管線是這樣的：

```
攝影機 → 編碼器(VP8/H.264) → 封包器(RTP/SRTP) → 網路
```

如果在攝影機後加密，編碼器看到的是密文，根本沒辦法壓縮。如果在封包器後加密，你得處理 RTP 分片，一個視訊幀可能被拆成幾十個封包，加解密複雜度暴增。

正確的插入點是 **編碼器之後、封包器之前**：

```
攝影機 → 編碼器 → 【加密 Hook】→ 封包器(SRTP) → 網路
```

這就是 WebRTC Encoded Transform API 做的事。它在管線的這個精確位置暴露出一個 `TransformStream`，讓你操作完整的編碼幀（一整個 VP8 keyframe 或一個 Opus audio frame），而不是碎片化的 RTP 封包。

API 的使用方式在標準化過程中經歷了演化。早期 Chrome 實作的是 `RTCRtpSender.createEncodedStreams()`，後來 W3C 統一為 `RTCRtpScriptTransform`——一個以 Web Worker 為核心的設計：

```javascript
// 發送端：掛上加密 transform
const worker = new Worker('crypto-worker.js');
sender.transform = new RTCRtpScriptTransform(worker, {
  side: 'send',
  participantId: 'user-1234'
});
```

Worker 端透過 `rtctransform` 事件接收串流，用 `TransformStream` 串接加解密邏輯：

```javascript
// crypto-worker.js
addEventListener('rtctransform', (event) => {
  const transform = new TransformStream({
    async transform(frame, controller) {
      // frame.data 是完整的編碼幀（ArrayBuffer）
      const encrypted = await encryptFrame(frame.data);
      frame.data = encrypted;
      controller.enqueue(frame);
    }
  });

  event.transformer.readable
    .pipeThrough(transform)
    .pipeTo(event.transformer.writable);
});
```

強制使用 Worker 不是 API 設計者的潔癖——視訊每秒 30-60 幀，每一幀都要做一次 AES-GCM 加解密，如果跑在主線程上，UI 會卡到你懷疑人生。Worker 讓加密運算和畫面渲染完全隔離。

有趣的是，Google Meet 實際上是在主線程使用這個 API 的，和規範建議相悖。Google 當然有自己的理由（減少主線程和 Worker 之間的結構化克隆開銷），但這也成為標準化過程中的一個爭議點。

---

## AES-GCM + SubtleCrypto：瀏覽器原生的加密肌肉

加密本身用的是 Web Crypto API（`SubtleCrypto`）的 AES-GCM 模式。選 GCM 而不是 CBC，因為 GCM 是 AEAD（Authenticated Encryption with Associated Data）——同時提供加密和認證，確保密文沒有被篡改，而且不需要 padding。

每個加密幀的格式大致是這樣：

```
[ IV (12 bytes) ] [ 密文 (N bytes) ] [ 認證標籤 (16 bytes，GCM 自動附加) ]
```

IV（初始化向量）必須在同一把密鑰下永不重複。生產環境通常用計數器而非隨機值來確保唯一性——`crypto.getRandomValues()` 在 12 bytes 空間裡碰撞的機率雖然微乎其微，但在安全領域「微乎其微」不是一個令人安心的詞。

密鑰交換則走 ECDH（Elliptic Curve Diffie-Hellman），同樣透過 SubtleCrypto 實作。流程不複雜：

1. 每個參與者生成一對 ECDH P-256 密鑰
2. 公鑰透過信令通道交換
3. 雙方用自己的私鑰 + 對方的公鑰推導出共享密鑰
4. 共享密鑰用來加密/解密媒體幀

```javascript
// 推導共享密鑰
const sharedKey = await crypto.subtle.deriveKey(
  { name: 'ECDH', public: remotePublicKey },
  localPrivateKey,
  { name: 'AES-GCM', length: 256 },
  false,
  ['encrypt', 'decrypt']
);
```

信令伺服器只是中繼，它看到的是公鑰交換（公鑰本來就是公開的）和被接收者公鑰加密過的對稱密鑰 blob。即使信令伺服器被完全控制，攻擊者也拿不到能解密媒體的對稱密鑰——除非他同時拿到某個參與者的私鑰。

---

## SFrame：從各家土砲到 IETF 標準

2024 年 8 月，IETF 正式發布了 RFC 9605——**SFrame（Secure Frame）**。這是 WebRTC E2EE 的封裝格式標準，由 Google 的 Justin Uberti 和 Emad Omara 在 2018 年首次提出，歷經六年標準化。

SFrame 解決的核心問題是：如果每家平台都自己定義加密幀格式（IV 放哪、Key ID 怎麼編碼、認證標籤多長），跨平台互操作就是地獄。RFC 9605 統一了這些：

- **Header**：包含 Key ID（KID）和 Counter（CTR），不加密但有完整性保護
- **Payload**：加密後的媒體資料
- **認證範圍**：Header + Payload 一起認證，防止篡改

支援的 cipher suite 包括 `AES_128_CTR_HMAC_SHA256` 和 `AES_128_GCM_SHA256` 等，兼顧效能和安全性的不同需求。

SFrame 和 MLS（Messaging Layer Security，RFC 9420）的搭配也被明確設計進來——MLS 負責群組密鑰管理和前向保密（Forward Secrecy），SFrame 負責用 MLS 推導出的密鑰做逐幀加密。當有人離開通話，密鑰必須輪換，確保離開者無法解密後續內容。這在群組通話場景中至關重要。

SFrame 還有一個務實的設計考量：它獨立於 RTP。這意味著它不只能用在 WebRTC，也能用在其他媒體傳輸協議上，降低了進入門檻。

---

## 瀏覽器支援：漫長的碎片化終於結束

這個 API 的標準化之路堪稱坎坷。

Chrome 在 2020 年率先推出了私有版的 `createEncodedStreams()` API。Safari 在 2022 年 3 月實作了標準化的 `RTCRtpScriptTransform`。Firefox 在 2023 年 8 月跟上（Firefox 117）。但因為 Chrome 的實作和標準版有差異（Chrome 支援主線程使用，標準禁止），開發者必須維護兩套程式碼或使用 shim。

到了 2025 年 10 月，MDN 正式將 WebRTC Encoded Transform 標記為 **Baseline 2025**——意味著所有主流瀏覽器的最新版本都支援。五年了。

但「都支援」不代表「用法一樣」。目前的實際狀況：

| 瀏覽器 | 標準 API | 備註 |
|---------|----------|------|
| Safari | `RTCRtpScriptTransform` | 原生支援，FaceTime Web 就在用 |
| Firefox | `RTCRtpScriptTransform` | Firefox 117+ 原生支援 |
| Chrome | 需要 shim | 底層用的是較早的私有 API，透過 adapter.js 橋接 |

Apple 的 FaceTime Web 可能是最大的幕後推手——它需要在瀏覽器裡實現和原生 FaceTime 一樣的 E2EE 保證，直接推動了 Safari 的實作和 Firefox 的跟進。

---

## SFU 不會看不到「一切」——它還是能看到夠多

一個常見誤解是：啟用 E2EE 後 SFU 就完全「瞎了」。事實沒那麼極端。

SFU 的核心工作是路由和擁塞控制，這些操作基於 **RTP header**，不是 payload。Header 裡的 SSRC（同步來源識別碼）、sequence number、timestamp 仍然是明文的（被 DTLS 的傳輸層加密保護，但 SFU 本來就能解開傳輸層）。所以 SFU 仍然知道：

- **誰在說話**（SSRC）
- **封包順序和時間戳**（序列號和時間戳）
- **頻寬使用量**（根據封包大小和頻率推算）

它**不知道**的是你說了什麼、你長什麼樣——媒體 payload 對它來說就是一團亂碼。

但有一個棘手的邊界案例：**Simulcast 層選擇**。很多 SFU 支援根據接收端頻寬動態切換視訊品質層（高解析度/低解析度）。有些編碼器（特別是 VP8）會把層依賴資訊放在 payload descriptor 裡，而 E2EE 會加密整個 payload，連同這些 descriptor。

解法是使用 **AV1 Dependency Descriptor** 這種放在 RTP header extension 裡（而非 payload 裡）的元數據機制。這讓 SFU 可以在不看 payload 的情況下做出正確的層選擇。但如果你的系統還在用 VP8 的舊式 payload descriptor，啟用 E2EE 可能會讓 simulcast 壞掉。

---

## 效能代價：不是免費的午餐

安全從來不是零成本的。

**CPU 開銷**：AES-GCM 在現代 CPU 上有硬體加速（AES-NI），加解密本身很快。但資料在主線程和 Worker 之間的結構化克隆（Structured Clone）不是零成本的——在低階行動裝置上，每幀可能增加 2-5ms 延遲。60fps 的每幀預算是 16ms，吃掉 5ms 已經是 30% 了。

**頻寬**：每個加密幀要附加 12 bytes IV + 16 bytes 認證標籤 = 28 bytes。對視訊幀（通常幾 KB 到幾十 KB）來說不痛不癢。但對音訊幀（可能只有 100 bytes 的 Opus 封包），這是接近 25% 的額外開銷。在頻寬受限的場景下，這不是可以忽略的數字。

**啟動延遲**：使用者看到畫面的前提是密鑰交換完成。如果密鑰交換的信令比媒體連線慢，使用者會先看到黑畫面。這對使用者體驗的衝擊比你想的大——人對「看到黑畫面」的耐受度遠低於「畫質差一點」。

**你失去的能力**：

- ❌ 伺服器端轉碼（VP8 → H.264）——SFU 看不到內容
- ❌ 伺服器端錄影——只能錄密文，解密需要額外的密鑰管理
- ❌ 伺服器端混音/合成畫面（MCU 功能）——完全不可能
- ❌ 伺服器端內容品質監控（黑畫面偵測之類）——只能靠客戶端 `getStats()` 上報

這些不是小事。很多企業客戶需要通話錄音（合規要求）、需要即時字幕（無障礙要求）、需要 AI 會議摘要（這不就是 2026 年最流行的功能嗎）。啟用 E2EE 意味著這些功能要嘛在客戶端做，要嘛做不了。

所以務實的做法是：**不要全域啟用**。用 feature flag 控制，只在敏感場景啟用——醫病視訊會診（HIPAA）、律師-客戶通話、金融機構的敏感會議。一般的團隊站會？DTLS-SRTP 夠了，省電比較實際。

---

## 產業現狀：誰在用，誰沒在用

說了這麼多技術，現實中到底有誰在生產環境用了？

**已部署 E2EE 的**：
- Apple FaceTime（包括 Web 版），是推動瀏覽器標準化的主要力量
- Google Meet 在 2024 年開始實驗性部署
- Jitsi Meet 從很早就開始支援，但之前受限於 Firefox 不支援
- Signal 的群組通話

**還沒有或部分的**：
- Zoom 有 E2EE 選項，但啟用後功能受限（不能雲端錄影、不能電話撥入）
- Microsoft Teams——至今沒有完整的 E2EE 群組通話支持
- 大部分企業視訊方案——錄音和合規需求讓他們很難全面啟用

webrtcHacks 的數據指出，Chrome 使用統計顯示 Insertable Streams API 的使用率不到所有 WebRTC 通話的 1%。技術已經就位，但產業採用率依然很低。這不是技術問題，是取捨問題——大部分場景下，人們選擇了功能完整性而非加密極端主義。

---

## 我的看法

Insertable Streams（或說 Encoded Transform API）的技術設計是漂亮的。在正確的管線位置暴露 hook，用 Web Worker 隔離效能影響，搭配 SubtleCrypto 做原生硬體加速的加解密——這是 Web Platform 在安全能力上的一次實質性進步，不是花架子。

SFrame（RFC 9605）的標準化也是正確的一步。沒有統一格式，每家平台自己造輪子，互操作性是零，安全審計的成本是 N 倍。

但我對「普及」不抱太大期待。原因很簡單：E2EE 和「功能豐富的雲端協作體驗」在架構上是根本衝突的。你不能既要伺服器看不到內容，又要伺服器幫你做即時字幕、會議摘要、雲端錄影。技術不是魔法，trade-off 是真實的。

對前端工程師來說，如果你在做涉及敏感資料的即時通訊產品（醫療、法律、金融），**現在**就該認真評估這套技術棧。所有瀏覽器都支援了，SFrame 標準已經定了，沒有再等的理由。

如果你的產品是一般的視訊會議或社交應用？先確保你的 DTLS-SRTP 設定正確，密鑰不會被 log 下來，然後把 E2EE 做成 opt-in 的進階功能。不是每通電話都需要防國家級攻擊者。

安全工程的精髓從來不是「用最強的加密」，而是「在正確的地方用正確強度的保護」。Insertable Streams 給了我們一個新的、更強的選項。要不要用，什麼時候用——那是工程判斷，不是信仰問題。

---

## 延伸閱讀

- [Trust No One: Implementing True End-to-End Encryption with Insertable Streams](https://dev.to/deepak_mishra_35863517037/trust-no-one-implementing-true-end-to-end-encryption-with-insertable-streams-2ndk) — 本文的起點，完整的實作導向教學
- [Using WebRTC Encoded Transforms (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Using_Encoded_Transforms) — W3C 標準 API 的官方文件與範例
- [RFC 9605: SFrame - Lightweight Authenticated Encryption for Real-Time Media](https://www.rfc-editor.org/rfc/rfc9605.html) — IETF 正式標準，定義了 E2EE 的幀格式
- [End-to-End Encryption in WebRTC… 4 Years Later (webrtcHacks)](https://webrtchacks.com/end-to-end-encryption-in-webrtc-4-years-later/) — 產業採用現狀的深度分析
- [End-to-end-encrypt WebRTC in All Browsers (Mozilla)](https://blog.mozilla.org/webrtc/end-to-end-encrypt-webrtc-in-all-browsers/) — Mozilla 對標準化進程和跨瀏覽器支援的觀點
