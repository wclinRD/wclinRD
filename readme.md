<div align="center">

# wclinRD

**I build the tooling layer that makes AI coding agents reliable — and I verify it against real systems, not documentation.**

Agent tooling & deterministic memory · MCP · EDA / RTL · macOS native

<!-- ── 個人資訊欄位（保留）：填入內容後刪除這行註解即可顯示 ──
**Name / 職稱** — e.g. Software Engineer · EDA & AI Infrastructure
────────────────────────────────────────────────────────────── -->

<sub>11 original projects · 6 upstream integrations · 423 commits on the largest one</sub>

</div>

---

> **繁中** — 我把 AI coding agent 當成一個真正的系統來設計：工具層要能用最少 token 做最多事，記憶要可預測且零成本，錯誤要能被反省修正，而每一項行為都要在真實環境上實測，而不是照著文件推測。這份 README 只寫可被驗證的工程指標。

## 🎯 Engineering Impact

I work where AI agent tooling meets systems that cannot afford guessing. My largest project is an **MCP server that exposes 88 tools to any MCP-compatible agent** through a thin onion-architecture core, so the agent loads capability by task ("crawl", "refactor", "debug") instead of carrying it in the system prompt. Alongside it sits a **zero-LLM memory plugin** that costs no tokens and gives identical output for identical input, a **model-lifecycle automation** that makes switching local LLM models deterministic, and a **macOS Verilog tracer** that packages an industrial-style trace workflow into a double-clickable `.app`.

**Recent shipped:** onion-architecture MCP tool layer (88 tools), deterministic SQLite/FTS5 agent memory, LM Studio auto load/unload with readiness verification, three-layer agent self-reflection, and a `pyslang`-based RTL cross-probing tracer.

> **繁中** — 我專注在「AI agent 工具鏈」與「不能靠猜的系統」的交界。最大的專案是把 88 個工具用洋蔥架構包成一層 MCP server，讓 agent 依任務動態載入能力，而不是把知識塞進 system prompt；搭配零 LLM、零 token 的決定性記憶、可驗證的本機模型換載流程，以及用 `pyslang` 做跨波形雙向追蹤的 macOS Verilog 工具。

## 💡 Architecture & Tradeoffs

### [opencode_smart_mcp](https://github.com/wclinRD/opencode_smart_mcp) — 88 tools behind an onion-architecture router

- **Problem** — An agent either gets a bare toolset or one enormous system prompt. Either way, context is burned before real work starts, and tool selection stays coarse.
- **Architecture** — Onion design: a thin core of **21 direct tools** plus a task classifier, with **67 sub-tools** behind a single `smart_run` router. **8 domain skills load on demand** by task type (crawl / debug / git / refactor / security / test / lang / report), plus 16 optional companion skills. Server is JSON-RPC 2.0 over stdio; a `loader.mjs` auto-scans and registers plugins.
- **Tradeoffs** — Dynamic routing adds a hop and depends on the classifier being right; a 88-tool catalog is real documentation debt. I chose token efficiency and a small always-loaded surface over a flat tool list, and made the router fall back to `hybrid_router` / `agent_recommend` when the task is ambiguous.
- **Impact** — **423 commits** (the largest project I maintain). Documented per-mechanism token savings: `smart_read` progressive loading 60–80%, `smart_edit_chain` 40–60%, `smart_exa_search` caveman compression 50–70%. Core features need **no API key** and run fully local. Vertical coverage (EDA/RTL via `smart_rtl_analyze` + `smart_eda_search`, medical literature via `smart_medical_search`) that general-purpose coding agents do not ship.

<sub>**繁中** — 問題：agent 的工具不是太少就是全塞進 system prompt；架構：21 個 direct tools + 任務分類器 + 67 個經 router 的 sub-tools + 8 個按需載入 skill；取捨：多一層路由與分類器依賴，換取 token 效率與極小的常駐介面；實證：423 commits、各機制省 40–80% token、核心功能免 API key、內建 EDA/RTL 與醫學文獻垂直能力。</sub>

### [opencode-memv2](https://github.com/wclinRD/opencode-memv2) — `detmem`: zero-LLM deterministic memory

- **Problem** — Agents lose context between sessions. LLM-based memory fixes that by spending tokens and returning different answers for the same input — unacceptable when a remembered decision must stay stable.
- **Architecture** — Every session transcript is written to a local SQLite database with an **FTS5 index**, and "decision / TODO / preference / convention" facts are extracted by **fixed regex** — no model call. Before the next prompt, top-K facts and snippets are injected into the front of it. For CJK, the index uses the **`trigram` tokenizer** with overlapping-trigram query expansion plus a LIKE fallback, ranked by bm25; writes go through WAL, dedupe on a `message_id` unique index, and facts use an expression unique index so a fact is current state, not an append-only log.
- **Tradeoffs** — Regex extraction is deliberately narrow: only marked lines are captured, and the loose "suggests using X" pattern is filtered by `looksLikeTech` because it once polluted the fact table with 135 junk rows. No embeddings, so no fuzzy semantic recall. In exchange: zero token cost, offline operation, identical output for identical input, and no `sqlite3` CLI or vector database to install.
- **Impact** — **55 unit/integration tests against real SQLite, with no mocks**, plus a Bun runtime pass and an end-to-end system test that asserts on a per-run random marker row rather than "row count went up". Self-healing index drift (`ftsDrift()`, used by both diagnosis and repair) means `inSync: false` cannot become an unfixable state. Every documented behavior was measured on OpenCode **v2.0.20**, including a V1→V2 API mismatch table (`session.idle` never fires in practice; `session.step.ended` is used instead).

<sub>**繁中** — 問題：記憶不該花 token，也不該同樣輸入得到不同結果；架構：對話寫入本機 SQLite + FTS5、固定正則抽事實、prompt 前注入 top-K，中文用 trigram + LIKE 備援；取捨：只能抓標記行、無語意模糊召回，換來零成本、離線、可完全預測；實證：55 項真實 SQLite 測試（無 mock）+ Bun + 端對端系統測試，索引漂移可自我修復，行為逐項在 v2.0.20 實測。</sub>

### [opencode_lmstudio_loader](https://github.com/wclinRD/opencode_lmstudio_loader) — Deterministic local model switching

- **Problem** — Pointing opencode at LM Studio models produced `Model is unloaded` failures: one model was being evicted while another was still being requested, and memory only allows one at a time.
- **Architecture** — A `model.request` hook (scoped to `providerID: lmstudio`) runs **unload → load → poll until the instance appears → readiness ping (`max_tokens=1`) → release the request**. All ensures for different models run through a **global serialized queue**, so concurrent requests cannot evict each other. A verified cache (instanceId + TTL) gives a zero-latency fast path; a `retry` hook synchronously recovers only for model-not-ready errors and overrides the decision to `delay: 0`; `session.execution.failed` triggers reactive background preload; async unloads are polled until the instance truly disappears.
- **Tradeoffs** — The readiness ping costs one tiny completion per cold load, and the strict single-model policy trades throughput for a predictable memory footprint. Non-`lmstudio` providers are never touched, and non-readiness errors keep opencode's own backoff instead of being overridden.
- **Impact** — **27/27 unit tests passing** using an injected fake server (no LM Studio required) plus a live system test against real LM Studio that auto-skips when it is not running; validated on opencode **2.0.20** for both cold-start `ensure` and the retry path (`delay` overridden 1605 → 0).

<sub>**繁中** — 問題：切換本機模型時新舊模型互踩，導致 `Model is unloaded`；架構：送出前卸載→載入→輪詢→以 `max_tokens=1` ping 確認可推論才放行，並以全域序列化佇列排隊；取捨：冷載入多一次極小請求、犧牲並行換取可預測的記憶體佔用；實證：27/27 單元測試（注入式 fake server）+ 真實 LM Studio 系統測試，2.0.20 冷啟動與 retry 覆寫皆實測通過。</sub>

### [opencode-auto-reflection](https://github.com/wclinRD/opencode-auto-reflection) — Self-learning loop for agents

- **Problem** — Agents repeat the same failures, and hand-written skills never evolve with what actually goes wrong.
- **Architecture** — Three layers: **L1** root-causes a tool error (throttled — first occurrence is only logged), **L2** extracts reusable tool-usage patterns when a task succeeds and compares them across sessions, **L3** consolidates drafts at session end. Reaching a threshold generates a structured `SKILL.md` that is routed to `~/.agents/skills/` (agent-generic) or `~/.opencode/skills/` (OpenCode-specific) and stays in `draft` until a human promotes it.
- **Tradeoffs** — The human gate on promotion slows skill evolution but prevents the skill library from being polluted by low-value drafts. Guardrails: recursion stops at depth ≥ 2, error logs store only `error_code` (no arguments), and reflection frequency drops automatically after 10 sessions with no new learning.
- **Impact** — Design survived **8 rounds of engineering review that fixed 96 issues** (documented per round in `docs/design-review.md`). Integrates with a heartbeat plugin so a `reflecting` flag survives session resumption — the reflection is not interrupted, and the depth counter is not reset (which would otherwise allow infinite recursion).

<sub>**繁中** — 問題：agent 會重複犯錯，靜態 skill 不會演化；架構：L1 錯誤歸因、L2 成功模式提取、L3 定期彙整，達閾值自動產生 SKILL.md 並標記 draft 待人工確認；取捨：人工閘門拖慢演化但避免 skill 被垃圾汙染；實證：8 輪工程審查共修 96 項漏洞，與 heartbeat 整合讓反省狀態跨續行存活。</sub>

### [visualcode_mcp_tool](https://github.com/wclinRD/visualcode_mcp_tool) — A safe file/edit MCP surface

- **Problem** — MCP hosts need a standardized file, search and edit tool surface that cannot be talked into escaping the workspace.
- **Architecture** — stdio MCP server with handlers split into core / edit / search / dangerous / extension groups (`read_file`, `write_file`, `multi_edit`, `apply_patch` with unified diff, `grep_file`, `glob_file`, `run_command`, `web_fetch`, `web_search`, `thinking_breakdown`). Three safety layers: `path.resolve` prefix check on every handler, a **read-before-edit session state**, and host sandboxing via `filesystem.allowWrite`.
- **Tradeoffs** — Read-before-edit adds friction to legitimate first writes and `run_command` is jailed to the root with a timeout, which limits its power. Chosen because blind overwrites and path escapes are worse failure modes than an extra read.
- **Impact** — Integration test covers handshake, tool listing, read/write/edit, grep, **path-traversal rejection**, **read-before-edit rejection** and delete — the safety properties are asserted, not just documented.

<sub>**繁中** — 問題：MCP host 需要安全且標準化的檔案／編輯介面；架構：stdio MCP server，handler 分 core/edit/search/dangerous/extension，三重防護（root 前綴比對、read-before-edit、host sandbox）；取捨：多一次讀取、指令被鎖在根目錄換取安全性；實證：整合測試直接斷言路徑穿越與未讀先寫都會被拒絕。</sub>

### [cursor-workflow](https://github.com/wclinRD/cursor-workflow) — Hybrid agent workflow ported to Cursor

- **Problem** — Heavy agent workflows force every task — including a one-line typo fix — through the same six phases, and Cursor's native tools get bypassed in favour of an MCP server.
- **Architecture** — Risk-tiered flow: 🟢 direct / 🟡 standard / 🔴 full loop, with a single-step exemption. **`workflow-state.json` is the single source of truth** (`status` + `todos`), and a Stop Hook validates both across at most 3 automatic iterations. `smart_think` degrades to inline structured analysis when MCP is unavailable, and token automation hooks inject caveman compression and periodic `smart_compact` prompts.
- **Tradeoffs** — A Stop Hook that can block adds latency to every turn, and JSON state is more ceremony than a scratchpad. Taken because progress becomes auditable and machine-verifiable instead of prose the agent can drift from.
- **Impact** — Ships `verify-install.sh` plus a bun test for the progress hook; supports both project-level and global installation; documented migration table from the previous version (forced MCP + scratchpad.md → native tools + JSON state).

<sub>**繁中** — 問題：重型工作流讓所有任務都走同樣六階段，且繞過 Cursor 原生工具；架構：風險分級 🟢/🟡/🔴 + 單步豁免，`workflow-state.json` 為唯一真相並由 Stop Hook 驗證；取捨：每回合多一次 Hook 檢查換取可稽核的進度狀態；實證：附 verify-install.sh 與 hook 測試，支援專案級與全域安裝。</sub>

### [smart-agent-package](https://github.com/wclinRD/smart-agent-package) — One-shot agent environment bootstrap

- **Problem** — A working agent environment is dozens of scattered dotfiles; rebuilding it on a new machine is an afternoon of archaeology.
- **Architecture** — A packaged `agents/` + `skills/` + `plugins/` tree with `install.sh` and ready-made `opencode.json` / `opencode-mem.jsonc` configs, so a new environment comes up in one command.
- **Tradeoffs** — An opinionated bundle is less flexible than hand-assembling configs, and it must be re-synced whenever the upstream tooling changes.
- **Impact** — Turns environment setup from a manual checklist into a single installable payload.

<sub>**繁中** — 問題：agent 環境散落成數十個設定檔，換機等於重考古；架構：打包 agents/skills/plugins 目錄 + install.sh + 現成設定檔；取捨：較不彈性且需與上游同步；實證：一鍵還原可用環境。</sub>

### [CircuitTrace](https://github.com/wclinRD/CircuitTrace) — Verilog tracing on macOS

- **Problem** — Tracing a signal's driver or load in Verilog normally means a heavy industrial EDA install. On macOS there is no lightweight option that feels like the tools engineers already use.
- **Architecture** — A modern web frontend (React 19 + Vite + Monaco) driven by a Python 3.11 backend (FastAPI + pywebview), packaged with PyInstaller into a standalone `.app`. Tracing uses `pyslang` `lookupName` for full hierarchical resolution (`tb.u_soc.u_cpu.clk`), and the waveform view embeds the Rust/WebAssembly Surfer engine with **two-way cross-probing**: `Ctrl+W` sends a signal from code to the waveform, and double-clicking a waveform signal traces back to its driver.
- **Tradeoffs** — Bundling a Python + web stack into a native app costs bundle size and startup time compared to a native rewrite. Chosen to reuse Surfer and Monaco rather than rebuild an editor and a waveform engine.
- **Impact** — A double-clickable `CircuitTrace.app` from `build_release.sh`, covering OneTrace (driver / load / connection), module hierarchy, multi-tab Monaco editing with jump-to-line, and a pop-out waveform window that inherits the selected signals.

<sub>**繁中** — 問題：macOS 上追蹤 Verilog 訊號沒有輕量選項；架構：React + Vite + Monaco 前端、Python + FastAPI + pywebview 後端，PyInstaller 打包為 .app，`pyslang` 解析完整階層路徑，內嵌 Surfer WASM 波形並支援雙向 cross-probing；取捨：打包體積與啟動時間換取重用成熟前端與波形引擎；實證：build_release.sh 產出可雙擊的 CircuitTrace.app。</sub>

### [omlxbar](https://github.com/wclinRD/omlxbar) — Menu bar monitor for AI provider limits (based on CodexBar)

- **Problem** — Local inference throughput and provider usage limits are invisible until you hit them mid-task.
- **Architecture** — Built on **CodexBar's** menu bar architecture (Swift, macOS 14+, MIT), with an added **OMLX live status widget** reporting prefill/generate TPS and request counts, so a local model's real throughput is visible at a glance.
- **Tradeoffs** — Deriving from a large upstream codebase (CodexBar's full Swift app, widget extension and docs) means fast progress but ongoing drift from upstream; a from-scratch menu bar app would be smaller but would re-solve solved problems.
- **Impact** — The widget surfaced both prefill/generate throughput and request counters that stock usage monitors do not show, acknowledging the CodexBar origin in the initial commit and preserving its MIT license.

<sub>**繁中** — 問題：本機推論吞吐與 provider 額度在使用前不可見；架構：沿用 CodexBar 的 macOS 選單列架構（Swift／MIT），新增 OMLX 即時狀態 widget 顯示 prefill/generate TPS 與請求數；取捨：繼承大型上游程式碼換取快速起步，但需持續跟上上游；實證：首個 commit 即註明 based on CodexBar，並保留 MIT 授權。</sub>

### [systemMonitor](https://github.com/wclinRD/systemMonitor) — Minimal SwiftPM system monitor

- **Problem** — A small monitoring app should not require an Xcode project to build.
- **Architecture** — Pure SwiftPM with `Info.plist`, `system.icns` and a `build_app.sh` that assembles a real `.app` into `release/`.
- **Tradeoffs** — No Xcode project means no storyboards or IB tooling; in exchange the build is a shell command and the repo stays small.
- **Impact** — `build_app.sh` produces a double-clickable `SystemMonitor.app` from a plain Swift package.

<sub>**繁中** — 問題：小型監控工具不該需要 Xcode 專案；架構：SwiftPM + Info.plist + build_app.sh 直接產出 .app；取捨：不用 Xcode 專案就沒有 IB 工具鏈；實證：一個 shell 指令產出可雙擊執行的 SystemMonitor.app。</sub>

### [Obsidian-Plugin-Example](https://github.com/wclinRD/Obsidian-Plugin-Example) — Minimal complete Obsidian plugin template

- **Problem** — Writing an Obsidian plugin means reverse-engineering the structure from other people's scattered code.
- **Architecture** — A smallest-complete example: ribbon icon, custom view, settings tab, and a separate `utils.js` module wired through an esbuild build step.
- **Tradeoffs** — Simplicity over features — it shows the wiring, not a production plugin.
- **Impact** — Gives a working plugin skeleton that can be cloned and renamed instead of assembled by guesswork.

<sub>**繁中** — 問題：寫 Obsidian 外掛得從別人的零散程式碼逆推結構；架構：最小完整範例（ribbon icon、自訂 view、settings tab、utils.js、esbuild）；取捨：只示範接線、不追求功能完整；實證：可直接 clone 改名開始開發。</sub>

## 🛠️ Technical Surface Area

**AI agent infrastructure & protocols**
- MCP (Model Context Protocol) server development over stdio / JSON-RPC 2.0
- Onion-architecture tool routing, on-demand skill loading, token-budget compression and session caching
- Deterministic agent memory: SQLite + FTS5 (`trigram` for CJK), bm25 ranking, WAL, self-healing index drift
- Agent self-reflection loops, automatic skill generation, and human-gated promotion
- Local LLM lifecycle automation (LM Studio load/unload, readiness verification, retry recovery)
- Agent workflow state machines (`workflow-state.json`), Stop Hooks, and risk-tiered execution

**EDA / RTL**
- SystemVerilog / Verilog tracing with `pyslang` AST and hierarchical `lookupName` resolution
- Waveform cross-probing with a Rust/WebAssembly engine (Surfer), module hierarchy analysis
- RTL lint / CDC / synthesis analysis exposed as agent tools

**macOS native**
- Swift, SwiftPM, menu bar and widget apps, `.app` bundle assembly and packaging scripts
- PyInstaller / pywebview packaging of Python + web stacks as native applications

**Web, data & tooling**
- React 19, Vite, Monaco Editor, TypeScript, FastAPI, Node.js
- PyInstaller, esbuild, `node:test`, bun test; shell-based build/release scripts for every project

## 📊 Engineering Metrics

I deliberately do not publish growth or ROI numbers I cannot verify. These are all countable from the repositories:

| Metric | Value |
|---|---|
| Largest project | **423 commits** (`opencode_smart_mcp`) |
| Tools exposed to agents | **88** (21 direct + 67 routed) |
| On-demand skills | **8 built-in** + 16 optional |
| Automated tests written | **55** (detmem, real SQLite) + **27** (LM Studio loader) + integration suites |
| Design review | **8 rounds, 96 issues fixed** (`opencode-auto-reflection`) |
| Documented token savings | **40–80%** per mechanism (session cache ≈100%) |
| Public repos | 18 (11 original + 6 integrations + this profile) |
| Working languages | JavaScript/Node, TypeScript, Python, Swift, Shell, Tcl |

## 🧭 Working Principles

- **No mocks in the verification path.** The memory plugin's tests run against real SQLite; the model loader's system test talks to a real LM Studio; the MCP exercise is an end-to-end `opencode run`.
- **Measure the API, don't trust the docs.** OpenCode V2's published types disagreed with its runtime, so the mismatch is documented behavior-by-behavior rather than guessed at.
- **Label what is verified and what is not.** Test suites that need external hardware or a running service auto-skip instead of pretending to pass.
- **Human gate on autonomy.** Self-generated skills stay `draft` until a person promotes them.
- **Small always-loaded surface, large on-demand surface.** Applies to tools, skills and memory alike.

> **繁中** — 原則：驗證路徑不使用 mock；以實測取代照文件推測（例如 OpenCode V2 型別與 runtime 不一致，就逐項記錄）；明確標示「已驗證」與「未驗證」，需要外部服務的測試自動 SKIP 而非假裝通過；自動產生的 skill 需人工確認才生效；常駐介面要小、按需能力要大。

## 🔀 Upstream Integrations

I fork and adapt tools I actually run in my own workflow, and I keep the upstream visible:

| Project | Upstream | Why I keep it in the loop |
|---|---|---|
| [`opencode2-todo-tool`](https://github.com/wclinRD/opencode2-todo-tool) | [gabparrot/opencode2-todo-tool](https://github.com/gabparrot/opencode2-todo-tool) | Restores V2's `todowrite` tool and a live session todo list (now DEPRECATED upstream) |
| [`asic-design-plugins`](https://github.com/wclinRD/asic-design-plugins) | [n0sig/asic-design-plugins](https://github.com/n0sig/asic-design-plugins) | Tcl ASIC flow plugins I extend for day-to-day work |
| [`ai-fde-vsdx-radar`](https://github.com/wclinRD/ai-fde-vsdx-radar) | [ueacdyy-sys/ai-fde-vsdx-radar](https://github.com/ueacdyy-sys/ai-fde-vsdx-radar) | Visio VSDX preview with lint and review-evidence collection |
| [`DockTamer`](https://github.com/wclinRD/DockTamer) | [fdamianenriquez/DockTamer](https://github.com/fdamianenriquez/DockTamer) | Windows-style click-to-minimize for the native macOS Dock |
| [`floorplan-3d`](https://github.com/wclinRD/floorplan-3d) | [wy51ai/floorplan-3d](https://github.com/wy51ai/floorplan-3d) | Adjusting layout data and validation flow |
| [`ayush-opencode`](https://github.com/wclinRD/ayush-opencode) | [CyberBoyAyush/ayush-opencode](https://github.com/CyberBoyAyush/ayush-opencode) | Reference and backup for opencode experiments |

## 🔭 Current Focus

- **Agent harness engineering** — tool routing, context management, agent memory, and self-evaluation
- **Verified agent memory** — deterministic extraction and retrieval that costs no tokens
- **AI-assisted development tooling** — opencode / Cursor / Claude Code plugins and workflow design
- **EDA tooling** — RTL tracing, waveform viewing, AST and dataflow analysis on macOS
- **Apple platform development** — SwiftUI, menu bar utilities, native packaging and signing

> **繁中** — 目前關注：agent harness 工程化、零 token 的決定性記憶、AI 開發工具外掛與工作流設計、EDA 追蹤與波形分析，以及 macOS 原生工具打包。

---

## 📫 Connect

- 🐙 **GitHub:** [github.com/wclinRD](https://github.com/wclinRD)

<!-- ── 個人資訊欄位（保留）：填入內容後刪除這段註解即可顯示 ──
- 👤 **Name:** 
- 💼 **LinkedIn:** 
- 📧 **Email:** 
- 🌐 **Website / Blog:** 
- 🐦 **X / Twitter:** 
────────────────────────────────────────────────────────── -->

Discussion about agent tooling, MCP servers, EDA tooling or macOS development is welcome — open an issue or a PR on any repository.

<sub>繁中 — 想討論 agent 工具鏈、MCP、EDA 工具或 macOS 開發，歡迎在任一 repo 開 issue 或 PR。</sub>

---

**Tech Stack:** JavaScript/Node.js, TypeScript, Python, Swift, React, SQLite FTS5, MCP, FastAPI, WebAssembly, SwiftPM, PyInstaller

*Building agent infrastructure that is measured on real systems, not on diagrams.*

<sub>Last updated: 2026-10-15</sub>
