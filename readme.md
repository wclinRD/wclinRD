<div align="center">

# wclinRD

**把 AI coding agent 當成一個真正的系統來設計。**

Agent 記憶 · MCP 工具鏈 · EDA / RTL · macOS

</div>

---

## 🧭 我在做什麼

三條主線，背後是同一個問題意識：**怎麼讓 agent 不只是「能跑」，而是能長期、可預測地工作。**

| 主線 | 想解決的問題 | 代表專案 |
|:---|:---|:---|
| 🤖 **AI Coding Agent 工具鏈** | Agent 的記憶、工具、反思應該被工程化，而不是全部塞進 system prompt | [`opencode_smart_mcp`](https://github.com/wclinRD/opencode_smart_mcp) · [`opencode-memv2`](https://github.com/wclinRD/opencode-memv2) |
| 🔬 **EDA / RTL 工具** | 把工業級 EDA 的 trace 體驗搬到 macOS，做成開源可用的桌面工具 | [`CircuitTrace`](https://github.com/wclinRD/CircuitTrace) |
| 🖥️ **macOS 常駐工具** | 原生 Swift、選單列、輕量低干擾 | [`omlxbar`](https://github.com/wclinRD/omlxbar) · [`systemMonitor`](https://github.com/wclinRD/systemMonitor) |

---

## 🚀 精選專案

### 🤖 AI Coding Agent 工具鏈

#### [opencode_smart_mcp](https://github.com/wclinRD/opencode_smart_mcp) — 洋蔥架構的 Agent 工具層

> **用最少 token 做最多事。**

為任何 MCP 相容的 agent 注入智慧工具層，提供 **88 個開發工具**。核心設計是「洋蔥架構」：外層只暴露 21 個 direct tools，搭配任務分類器與 67 個 sub-tools，收到「爬蟲」「重構」「除錯」等任務時才動態載入對應 skill — 不再把所有知識預先塞進 system prompt。

- 支援 **opencode**（主要）、**Claude Code**、**Cursor**、任何標準 MCP host
- 內含 RTL 程式碼理解、安全掃描、網頁抓取與語意搜尋等深度能力

#### [opencode-memv2](https://github.com/wclinRD/opencode-memv2) — detmem，零 LLM 的決定性記憶

> **不呼叫任何 AI Provider、不消耗 token、離線可用、結果完全可預測。**

把每個 session 的對話寫進本機 SQLite（FTS5），用固定正則抽出「決定 / TODO / 偏好 / 慣例」，並在下一次 prompt 前自動注入最相關的記憶。在 OpenCode v2.0.20 上逐項實測驗證，而非照文件推測。

#### [opencode_lmstudio_loader](https://github.com/wclinRD/opencode_lmstudio_loader) — 本機模型自動換載

讓 opencode 使用 LM Studio 模型時**自動卸載舊模型、載入目標模型**，保證記憶體只留單一模型。

- **就緒驗證**：load 後用 `max_tokens=1` 的 chat completion 確認引擎真的可推論才放行請求
- **並發序列化**：不同模型的並發載入排隊執行，修掉 `Model is unloaded` 的主因
- **失敗自動復原**：`retry` hook 僅針對「模型未就緒」類錯誤同步重新 ensure，成功則覆寫 `delay:0` 立即重試
- 快路徑驗證快取（instanceId + TTL）、回應式預載、卸載完成輪詢

#### [opencode-auto-reflection](https://github.com/wclinRD/opencode-auto-reflection) — 讓 agent 有自我學習能力

工具報錯時分析原因、任務成功時提取模式、自動產生新 skill 形成持續學習迴圈。

#### [cursor-workflow](https://github.com/wclinRD/cursor-workflow) — 把 Hybrid Agent 工作流移植到 Cursor

貼近 Cursor 原生工具（Read / Grep / StrReplace），Smart MCP 負責深度分析；任務追蹤以 **`workflow-state.json` 為唯一真相來源**。支援專案級與全域安裝。

#### [visualcode_mcp_tool](https://github.com/wclinRD/visualcode_mcp_tool) — VS Code File MCP

標準化的檔案操作 MCP server（stdio），提供 `read_file` / `write_file` / `edit_file` / `apply_patch` / `grep_file` / `list_dir` / `glob_file` 等工具，並延伸 `run_command`、`web_fetch`、`web_search`（Exa，無 key 自動 fallback DuckDuckGo）與 `thinking_breakdown` 結構化推理引擎。

### 🔬 EDA / RTL

#### [CircuitTrace](https://github.com/wclinRD/CircuitTrace) — macOS 上的 Verilog 追蹤工具

介面與操作邏輯致敬工業界標準 EDA 工具，用現代 Web 前端 + Python 分析後端打包成獨立 `.app`。

- **OneTrace 面板**：Trace Driver / Load / Connection，透過 `pyslang` 的 `lookupName` 支援完整階層路徑精準解析（如 `tb.u_soc.u_cpu.clk`）
- **波形雙向互動**：內建 Rust/WASM 的 Surfer 波形引擎，支援程式碼 `Ctrl+W` 加入信號，波形雙擊反查 Driver（Cross-Probing）
- **Monaco Editor**：Verilog 語法高亮、多頁籤、智慧跳轉至檔案行號
- 模組階層檢視、可拖曳的自訂佈局、獨立彈出波形視窗
- `React 19 + Vite` · `Python 3.11 + FastAPI + pywebview` · `PyInstaller`

### 🖥️ macOS

#### [omlxbar](https://github.com/wclinRD/omlxbar)

選單列監控工具，基於 CodexBar 架構延伸。`Swift` · `MIT`

#### [systemMonitor](https://github.com/wclinRD/systemMonitor)

輕量系統監控。`Swift`

### 🧩 其他

- **[Obsidian-Plugin-Example](https://github.com/wclinRD/Obsidian-Plugin-Example)** — Obsidian 外掛開發框架：Ribbon Icon、自訂 View、Settings Tab、獨立 utils 整合的最小完整範例
- **[smart-agent-package](https://github.com/wclinRD/smart-agent-package)** — agent 打包工具

---

## 🛠️ 技術棧

**語言**

![JavaScript](https://img.shields.io/badge/-JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![TypeScript](https://img.shields.io/badge/-TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Python](https://img.shields.io/badge/-Python-3776AB?style=flat-square&logo=python&logoColor=white)
![Swift](https://img.shields.io/badge/-Swift-F05138?style=flat-square&logo=swift&logoColor=white)
![Shell](https://img.shields.io/badge/-Shell-4EAA25?style=flat-square&logo=gnubash&logoColor=white)
![HTML5](https://img.shields.io/badge/-HTML5-E34F26?style=flat-square&logo=html5&logoColor=white)
![CSS3](https://img.shields.io/badge/-CSS3-1572B6?style=flat-square&logo=css3&logoColor=white)
![Node.js](https://img.shields.io/badge/-Node.js-5FA04E?style=flat-square&logo=nodedotjs&logoColor=white)

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
![SQLite](https://img.shields.io/badge/-SQLite_FTS5-003B57?style=flat-square&logo=sqlite&logoColor=white)

**平台 / EDA**

![macOS](https://img.shields.io/badge/-macOS-000000?style=flat-square&logo=apple&logoColor=white)
![SystemVerilog](https://img.shields.io/badge/-SystemVerilog-8B0000?style=flat-square)
![Verilog](https://img.shields.io/badge/-Verilog-8B0000?style=flat-square)
![WebAssembly](https://img.shields.io/badge/-WebAssembly-654FF0?style=flat-square&logo=webassembly&logoColor=white)

---

## 🔭 我關注的領域

從 star 紀錄看得出來，我一直在追蹤這幾條線的發展：

- **Agent 工程化** — skills 生態、harness engineering、context 管理、agent 記憶體與自我評測
- **AI 輔助開發工具** — opencode / Cursor / Claude Code 的外掛與工作流設計
- **Apple 平台開發** — SwiftUI、macOS 選單列工具、原生打包與簽章
- **EDA / 硬體** — RTL 追蹤、波形檢視、AST 與 dataflow 分析
- **知識管理** — Obsidian 外掛、個人 wiki 與筆記自動化

---

<div align="center">

### 📫 找我

有興趣一起討論 agent 工具鏈、EDA 工具或 macOS 開發，歡迎開 issue 或 PR。

**專案的每一個 README 都盡量誠實標註「實測通過」與「尚未驗證」的差別。**

<sub>README 由我自己的 agent 工作流協作撰寫 · Last updated: 2026-10-15</sub>

</div>
