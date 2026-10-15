<div align="center">

# wclinRD

**把 AI coding agent 當成一個真正的系統來設計。**

Agent 工具鏈 · 記憶體工程 · EDA / RTL · macOS

<sub>11 個原創專案 · 6 個 fork 整合</sub>

</div>
---

## 🗺️ 專案地圖

全部 18 個 repo：**11 個原創專案** + **6 個 fork** + 本 profile repo。

| 類別 | 專案 | 語言 | 類型 |
|:---|:---|:---|:---|
| 🤖 Agent 工具鏈 | [`opencode_smart_mcp`](https://github.com/wclinRD/opencode_smart_mcp) | JavaScript | 原創 |
| 🤖 Agent 工具鏈 | [`opencode-memv2`](https://github.com/wclinRD/opencode-memv2) | JavaScript | 原創 |
| 🤖 Agent 工具鏈 | [`opencode_lmstudio_loader`](https://github.com/wclinRD/opencode_lmstudio_loader) | JavaScript | 原創 |
| 🤖 Agent 工具鏈 | [`opencode-auto-reflection`](https://github.com/wclinRD/opencode-auto-reflection) | Skill / JSON | 原創 |
| 🤖 Agent 工具鏈 | [`visualcode_mcp_tool`](https://github.com/wclinRD/visualcode_mcp_tool) | TypeScript | 原創 |
| 🤖 Agent 工具鏈 | [`cursor-workflow`](https://github.com/wclinRD/cursor-workflow) | JavaScript | 原創 |
| 🤖 Agent 工具鏈 | [`smart-agent-package`](https://github.com/wclinRD/smart-agent-package) | JavaScript | 原創 |
| 🤖 Agent 工具鏈 | [`opencode2-todo-tool`](https://github.com/wclinRD/opencode2-todo-tool) | TypeScript | Fork |
| 🔬 EDA / RTL | [`CircuitTrace`](https://github.com/wclinRD/CircuitTrace) | Python / React | 原創 |
| 🔬 EDA / RTL | [`asic-design-plugins`](https://github.com/wclinRD/asic-design-plugins) | Tcl | Fork |
| 🖥️ macOS | [`systemMonitor`](https://github.com/wclinRD/systemMonitor) | Swift | 原創 |
| 🖥️ macOS | [`omlxbar`](https://github.com/wclinRD/omlxbar) | Swift | 原創 |
| 🖥️ macOS | [`DockTamer`](https://github.com/wclinRD/DockTamer) | Swift | Fork |
| 🧠 知識管理 | [`Obsidian-Plugin-Example`](https://github.com/wclinRD/Obsidian-Plugin-Example) | JavaScript | 原創 |
| 🧠 知識管理 | [`ai-fde-vsdx-radar`](https://github.com/wclinRD/ai-fde-vsdx-radar) | TypeScript | Fork |
| 🧠 知識管理 | [`floorplan-3d`](https://github.com/wclinRD/floorplan-3d) | HTML | Fork |
| 🧪 其他 | [`ayush-opencode`](https://github.com/wclinRD/ayush-opencode) | TypeScript | Fork |

---

## 🤖 AI Coding Agent 工具鏈

這是我投入最多的方向。共同的問題意識是：**agent 的能力不該靠 prompt 硬塞，而該靠可測試的基礎設施。**

### `opencode_smart_mcp` — 洋蔥架構的 Agent 工具層

> **用最少 token 做最多事。**

為任何 MCP 相容的 agent 注入智慧工具層，提供 **88 個開發工具**。核心是「洋蔥架構」：外層只暴露 21 個 direct tools，搭配任務分類器與 67 個 sub-tools；收到「爬蟲」「重構」「除錯」等任務時才動態載入對應 skill，而不是把所有知識預先塞進 system prompt。

- 支援 **opencode**（主要平台）、**Claude Code**、**Cursor**、任何標準 MCP host
- 內含 RTL 程式碼理解、安全掃描、網頁抓取與語意搜尋
- 這是產出量最大的專案（423 次 commit），含完整 plan / todo 文件與測試目錄

### `opencode-memv2` — detmem，零 LLM 的決定性記憶

> 不呼叫任何 AI Provider、不消耗 token、離線可用、**結果完全可預測**。

把每個 session 的對話寫進本機 SQLite（FTS5），用固定正則抽出「決定 / TODO / 偏好 / 慣例」，並在下一次 prompt 前自動注入最相關的記憶。每一項行為都在 OpenCode v2.0.20 上逐項實測，而非照文件推測。安裝上刻意避開「把整個資料夾丟進 plugins/」的陷阱，改用 `plugins` 陣列指定專案根目錄。

### `opencode_lmstudio_loader` — 本機模型自動換載

讓 opencode 使用 LM Studio 模型時**自動卸載舊模型、載入目標模型**，保證記憶體只保留單一模型。

- **就緒驗證**：load 後用 `max_tokens=1` 的 chat completion 確認引擎真的能推論，才放行請求
- **並發序列化**：不同模型的並發載入排隊執行，修掉 `Model is unloaded` 的主因
- **失敗自動復原**：`retry` hook 僅針對「模型未就緒」類錯誤同步重新 ensure，成功則覆寫 `delay: 0` 立即重試；其餘錯誤沿用預設退避
- **卸載完成等待**：unload 是 async，會 poll 到 instance 真的消失才載入新模型
- 驗證快取（instanceId + TTL）走零延遲快路徑；訂閱 `session.execution.failed` 做回應式預載

### `opencode-auto-reflection` — 讓 agent 有自我學習能力

工具報錯時分析原因、任務成功時提取模式、自動產生新 skill，形成持續學習迴圈。以 `skills/self-reflection/SKILL.md` 搭配 `reflection-log.jsonl` 實作，不需要改動 agent 本身。

### `visualcode_mcp_tool` — VS Code File MCP

標準化檔案操作的 MCP server（stdio），提供 `read_file` / `write_file` / `edit_file` / `multi_edit` / `apply_patch`（支援 unified diff 多檔） / `grep_file` / `glob_file` / `list_dir` 等工具，並延伸三類能力：

- `run_command`（cwd 限制在根目錄內、含逾時）
- `web_fetch` / `web_search`（完整 Exa 語意搜尋，無 key 自動 fallback DuckDuckGo）
- `thinking_breakdown`（cit / beam / forest / structured 四模式的結構化推理引擎）

### `cursor-workflow` — 把 Hybrid Agent 工作流移植到 Cursor

貼近 Cursor 原生工具（Read / Grep / StrReplace），Smart MCP 負責深度分析；任務追蹤以 **`workflow-state.json` 為唯一真相來源**，並附 `verify-install.sh` 做安裝驗證。支援專案級與全域兩種安裝範圍。

### `smart-agent-package` — 一鍵部署的 agent 套件

把一整套 agent 環境打包成可安裝的套件：`agents/`、`skills/`、`plugins/` 目錄搭配 `install.sh`，並附上 `opencode.json`、`opencode-mem.jsonc` 等設定檔，讓新環境能一次到位。

### `opencode2-todo-tool` <sub>(Fork)</sub> — 補回 V2 的 todo 能力

> 上游：[gabparrot/opencode2-todo-tool](https://github.com/gabparrot/opencode2-todo-tool) · 狀態：**DEPRECATED**

OpenCode V2 plugin，補回被移除的 `todowrite` 工具與 session 側邊欄的即時 todo 清單。特色是**完整取代語意**（每次呼叫取代整份清單）、以 `ctx.storage` 做 per-session 持久化避免 session 互相汙染，以及每一輪對話前重新注入最新清單。

---

## 🔬 EDA / RTL

### `CircuitTrace` — macOS 上的 Verilog 追蹤工具

介面與操作邏輯致敬工業界標準 EDA 工具，用現代 Web 前端 + Python 分析後端打包成獨立 `.app`，雙擊即可開啟。

- **OneTrace 面板**：Trace Driver / Load / Connection 一鍵追蹤訊號來源與去向，透過 `pyslang` 的 `lookupName` 支援完整階層路徑精準解析（如 `tb.u_soc.u_cpu.clk`）
- **波形雙向互動**：內建 Rust / WebAssembly 的 Surfer 波形引擎，可從程式碼 `Ctrl+W` 把信號加入波形，也能在波形視窗雙擊信號反查 Driver（Cross-Probing）
- **Monaco Editor**：Verilog 語法高亮、多頁籤、點擊追蹤結果自動跳轉至檔案與行號
- **模組階層檢視** + 可拖曳的雙向 Resizer 佈局 + 波形獨立彈出視窗（智慧繼承已選觀察信號）
- 技術棧：`React 19 + Vite` · `Python 3.11 + FastAPI + pywebview` · `pyslang` · `PyInstaller`

### `asic-design-plugins` <sub>(Fork)</sub>

> 上游：[n0sig/asic-design-plugins](https://github.com/n0sig/asic-design-plugins) · `Tcl`

ASIC 設計流程的外掛集合。我 fork 它是為了在工作流程中調整與擴充，屬於自己每天在用的工具鏈。

---

## 🖥️ macOS 原生工具

### `systemMonitor`

以 SwiftPM 建構的輕量系統監控，附 `Info.plist`、`build_app.sh` 與 `system.icns`，能直接產出可用的 `SystemMonitor.app` 並放入 `release/`。

### `omlxbar`

選單列監控工具，基於 [CodexBar](https://github.com/steipete/CodexBar) 的架構延伸，用來把 AI provider 的用量與重置時間常駐在選單列。`Swift` · `MIT`

### `DockTamer` <sub>(Fork)</sub>

> 上游：[fdamianenriquez/DockTamer](https://github.com/fdamianenriquez/DockTamer) · `Swift` · `MIT`

把 Windows 風格的「點擊 Dock 圖示即最小化」帶到原生 macOS Dock 的輕量工具。

---

## 🧠 知識管理與視覺化

### `Obsidian-Plugin-Example`

Obsidian 外掛開發框架，用最小完整範例示範 Ribbon Icon、自訂 View、Settings Tab，以及如何與獨立的 `utils.js` 整合。目的是讓「想寫 Obsidian 外掛」的人不用從零摸索結構。

### `ai-fde-vsdx-radar` <sub>(Fork)</sub>

> 上游：[ueacdyy-sys/ai-fde-vsdx-radar](https://github.com/ueacdyy-sys/ai-fde-vsdx-radar) · `TypeScript`

VS Code 的 Visio VSDX 預覽工具，結合 AI 做程式品質檢查、lint 與審查證據整理。

### `floorplan-3d` <sub>(Fork)</sub>

> 上游：[wy51ai/floorplan-3d](https://github.com/wy51ai/floorplan-3d) · `HTML` · `MIT`

3D 房屋格局與平面圖工具。fork 來做一些格局資料與驗證流程的調整。

---

## 🧪 其他

### `ayush-opencode` <sub>(Fork)</sub>

> 上游：[CyberBoyAyush/ayush-opencode](https://github.com/CyberBoyAyush/ayush-opencode) · `TypeScript`

opencode 相關的整合實驗，作為參考與備份保留。

---

## 🛠️ 技術棧

**語言**

![JavaScript](https://img.shields.io/badge/-JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![TypeScript](https://img.shields.io/badge/-TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Python](https://img.shields.io/badge/-Python-3776AB?style=flat-square&logo=python&logoColor=white)
![Swift](https://img.shields.io/badge/-Swift-F05138?style=flat-square&logo=swift&logoColor=white)
![Node.js](https://img.shields.io/badge/-Node.js-5FA04E?style=flat-square&logo=nodedotjs&logoColor=white)
![Shell](https://img.shields.io/badge/-Shell-4EAA25?style=flat-square&logo=gnubash&logoColor=white)
![Tcl](https://img.shields.io/badge/-Tcl-C3B6D9?style=flat-square)

**AI Agent / 協定**

![MCP](https://img.shields.io/badge/-Model_Context_Protocol-000000?style=flat-square&logo=anthropic&logoColor=white)
![OpenCode](https://img.shields.io/badge/-OpenCode-1F1F1F?style=flat-square)
![Claude Code](https://img.shields.io/badge/-Claude_Code-D97757?style=flat-square&logo=anthropic&logoColor=white)
![Cursor](https://img.shields.io/badge/-Cursor-000000?style=flat-square)
![LM Studio](https://img.shields.io/badge/-LM_Studio-4B5563?style=flat-square)

**前端 / 後端 / 資料**

![React](https://img.shields.io/badge/-React_19-61DAFB?style=flat-square&logo=react&logoColor=black)
![Vite](https://img.shields.io/badge/-Vite-646CFF?style=flat-square&logo=vite&logoColor=white)
![FastAPI](https://img.shields.io/badge/-FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![SQLite FTS5](https://img.shields.io/badge/-SQLite_FTS5-003B57?style=flat-square&logo=sqlite&logoColor=white)
![WebAssembly](https://img.shields.io/badge/-WebAssembly-654FF0?style=flat-square&logo=webassembly&logoColor=white)

**平台 / EDA**

![macOS](https://img.shields.io/badge/-macOS-000000?style=flat-square&logo=apple&logoColor=white)
![SwiftPM](https://img.shields.io/badge/-SwiftPM-F05138?style=flat-square&logo=swift&logoColor=white)
![SystemVerilog](https://img.shields.io/badge/-SystemVerilog-8B0000?style=flat-square)
![Verilog](https://img.shields.io/badge/-Verilog-8B0000?style=flat-square)

---

## 🔭 我關注的領域

- **Agent 工程化** — skills 生態、harness engineering、context 管理、agent 記憶與自我評測
- **AI 輔助開發工具** — opencode / Cursor / Claude Code 的外掛與工作流設計
- **Apple 平台開發** — SwiftUI、選單列常駐工具、原生打包與簽章
- **EDA / 硬體** — RTL 追蹤、波形檢視、AST 與 dataflow 分析
- **知識管理** — Obsidian 外掛、筆記自動化與個人 wiki

---

<div align="center">

### 📫 找我

想一起討論 agent 工具鏈、EDA 工具或 macOS 開發，歡迎開 issue 或 PR。

**每一個專案 README 都盡量誠實標註「實測通過」與「尚未驗證」的差別。**

<sub>Last updated: 2026-10-15</sub>

</div>
