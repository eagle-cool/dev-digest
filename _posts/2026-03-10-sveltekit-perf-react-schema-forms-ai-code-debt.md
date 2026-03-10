---
title: "SvelteKit 效能調校實戰、React Schema 表單引擎、AI 寫的 code 你敢信？"
date: 2026-03-10
description: "SvelteKit Lighthouse 行動版從 70 分衝到 95+、用 JSON Schema 驅動 React 表單不用重新部署、AI 生成的程式碼正在默默堆技術債——前端老司機今天帶你看這三件事。"
tags: [frontend, react, tooling, ai]
---

今天三道硬菜都蠻值得聊的。一個是 SvelteKit 效能調校的完整實戰紀錄，從 Lighthouse 行動版 70 分一路幹到 95+，過程中踩的坑比結果還有料。另一個是 React 表單的老問題有了新解法——用 JSON Schema 驅動，不用重部署就能改欄位。最後，有人終於把大家心裡那句話說出來了：AI 寫 code 很快，但那些 code 真的能信嗎？

---

## 🔧 今日硬菜

### [How I optimized my SvelteKit website from 70 to 95+ on mobile Lighthouse tests](https://dev.to/metehandesign/how-i-optimized-my-sveltekit-website-from-70-to-95-on-mobile-lighthouse-tests-1ba3)

這篇是少見的「完整調校日記」，不是那種「加個 lazy loading 就收工」的水文。作者從 Lighthouse 行動版 65-80 分開始，一步步拆解每個瓶頸：CSS 檔案造成 render blocking、LCP 元素竟然不是圖片而是文字、字型檔肥到 269KB、28 個 JS chunk 在窄頻寬下互搶資源。最狠的一招是用 SvelteKit 的 `hooks.server.ts` 把非必要 JS 的 preload 拿掉，讓 LCP 圖片優先載入——這一步就把分數從 92-96 穩定拉到 95-98。

**重點：**
- CSS inlining（`inlineStyleThreshold`）消滅 render blocking，但要注意 Svelte 看的是磁碟大小不是壓縮後大小——作者在這裡就踩了一坑
- 行動版選 progressive JPEG 而不是 AVIF，因為 AVIF 在 throttled CPU 上解碼太慢，檔案小但分數反而更低
- 但是... 這些 SvelteKit 特定的優化技巧（CSS inlining、chunk 合併）會犧牲快取效率，SaaS 應用要三思

### [Stop Hardcoding React Forms: Build Dynamic Schema-Driven Forms with Formitiva](https://dev.to/yanggmtl/stop-hardcoding-react-forms-build-dynamic-schema-driven-forms-with-formitiva-5d14)

表單這個東西，寫了十年還是在寫。Formitiva 的切入點不是要取代 React Hook Form 或 Formik——那些處理靜態表單已經夠好了。它瞄準的是「表單結構本身就是資料」的場景：多租戶 SaaS、CMS 驅動的後台、需要非工程師改欄位的平台。把表單定義成 JSON Schema，存資料庫、透過 API 下發、不用重新部署就能改結構。概念不新，但把 registry 機制（自訂元件、驗證器、提交 handler）做進去算是想清楚了。

**重點：**
- 核心架構：Definition（JSON Schema）→ Renderer（React 元件）→ Registries（擴充點），關注點分離做得乾淨
- 附帶一個拖拉式 Builder，讓非前端人員也能組表單——對 B2B SaaS 來說這是真需求
- 但是... 又一個表單庫。生態系、社群、長期維護都是問號，production 要用之前先看看 GitHub 星數和 issue 回覆速度

### [We're Measuring AI Velocity, but We're Ignoring AI Gravity](https://dev.to/rajanp/were-measuring-ai-velocity-but-were-ignoring-ai-gravity-488d)

「GitHub Copilot 和 Claude 是我見過最快製造技術債的方式。」——開頭這句話就夠嗆了，但說的是實話。作者提出一個概念叫「AI Gravity」：AI 寫 code 飛快，但每一行「差不多對」的程式碼都在幫你的 repo 加重量，最終把創新速度拖到停擺。解方是三層防禦：SAST 靜態分析（Sonar）當確定性閘門、CodeScene 做行為健康度監控、加上工程治理框架提供可稽核性。短文，但戳到痛點。

**重點：**
- 2026 年的瓶頸不是寫 code，是驗證 code——AI 產出的程式碼要「有罪推定」直到通過確定性驗證
- 把 AI 當油門、靜態分析當煞車，兩個都要有才不會翻車
- 但是... 文章推薦的工具（Sonar、CodeScene、ExtenSURE）都是商業產品，難免有業配嫌疑，核心觀點對但解方要自己判斷

---

## ⚡ 一句話帶過

- **[Why I Built a Reverse-CAPTCHA That Verifies AI Agents, Not Humans](https://dev.to/leo_pechnicki/why-i-built-a-reverse-captcha-that-verifies-ai-agents-not-humans-2jbi)** — 不問「你是不是人」，問「你是不是合法的 AI」。方向對了，Web 的訪客結構正在根本性地改變
- **[Why We Don't Use Socket Connections for Everything](https://dev.to/rajat10/why-we-dont-use-socket-connections-for-everything-3j6g)** — WebSocket 不是萬能的，HTTP 的無狀態和快取優勢在多數場景下還是贏。基礎觀念但值得新手讀
- **[How I Built an Animated Circle Progress Component Using FSCSS (No JavaScript)](https://dev.to/fscss-ttr/how-i-built-an-animated-circle-progress-component-using-fscss-no-javascript-18o4)** — 純 CSS 圓形進度條，用 custom properties 驅動動畫。CSS 能做到的事情越來越離譜
- **[I built a free Planning Poker tool with Next.js and Socket.io](https://dev.to/kodakro/i-built-a-free-planning-poker-tool-with-nextjs-and-socketio-20dd)** — Next.js + Socket.io + Redis 做的 Planning Poker，技術選型中規中矩但功能完整
- **[Why Shift+Enter doesn't work in Claude Code (and how to fix it)](https://dev.to/richardbray/why-shiftenter-doesnt-work-in-claude-code-and-how-to-fix-it-10f7)** — 原來要開 Kitty keyboard protocol。用 Claude Code 的人可以省下十分鐘 debug 時間
- **[Finding Dependency Confusion Vulnerabilities in Public GitHub Repositories](https://dev.to/sidhanta_palei_b40572bcbd/detecting-dependency-confusion-vulnerabilities-in-github-repositories-18hm)** — npm 供應鏈攻擊的老手法 dependency confusion，但 2026 了還是有人踩。你的 package.json 有私有 scope 嗎？
- **[I Found an Open-Source AI Agent That Runs on Your Laptop and Talks Through Telegram, Discord, and WhatsApp](https://dev.to/dsharma08k/i-found-an-open-source-ai-agent-that-runs-on-your-laptop-and-talks-through-telegram-discord-and-17kl)** — 本地跑的開源 AI Agent，支援多平台。不想把資料丟雲端的人可以看看
- **[I Built a Chrome Extension That Finally Makes Bookmarks Searchable](https://dev.to/jagadeeshjayachandran/i-built-a-chrome-extension-that-finally-makes-bookmarks-searchable-heres-how-4a4d)** — Chrome 書籤搜尋爛了十年，終於有人受不了自己寫了一個

---

## 📚 慢慢啃

- **[My Claude Code Skill Got Flagged by a Security Scanner. Here's What I Found and Fixed.](https://dev.to/othmanadi/my-claude-code-skill-got-flagged-by-a-security-scanner-heres-what-i-found-and-fixed-1g1a)** — 用 Manus 的 context-engineering pattern 寫 Claude Code skill，結果被安全掃描器抓到問題。讀完你會更理解 AI agent 的檔案系統存取該怎麼設計
- **[Zero Trust for AI Agents? Google Workspace CLI's Design Philosophy](https://dev.to/akari_iku/zero-trust-for-ai-agents-google-workspace-clis-design-philosophy-46k1)** — Google 的人寫了個 Workspace CLI，底層是零信任架構。AI agent 和企業 API 的整合模式正在成形，這篇幫你看清方向
- **[GitHub Copilot: Assistant for my current Python workflow](https://dev.to/srini047/github-copilot-assistant-for-my-current-python-workflow-2phm)** — 把開發流程拆成三個階段，逐一說明 Copilot 怎麼融入。雖然是 Python 場景，但工作流的思考方式前端也適用
