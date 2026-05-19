# 架構說明：zed

> 由 `/map` 於 2026-05-20 產生，涵蓋提交 `ad042e5c9d`。

## 概覽

Zed 是一款以 Rust 全程式碼庫打造、GPU 加速的協作程式碼編輯器。它透過整合自訂的即時模式 UI 框架（GPUI）、基於 CRDT 的協作文字引擎，以及深度 AI 整合，解決了編輯器高延遲、低效能的痛點，並將這一切融合為單一原生應用程式。技術棧為純 Rust，搭配各平台專屬的渲染後端（Metal、Direct3D、Vulkan/Skia），以及使用 PostgreSQL 與 WebSocket 進行即時協作的伺服器元件。

---

## 技術棧

| 層次 | 技術 |
|------|------|
| 語言 | Rust（工作區版本 2024） |
| UI 框架 | GPUI（自訂、GPU 加速、即時模式） |
| 文字引擎 | 自訂 CRDT rope（`crates/text`、`crates/rope`、`crates/sum_tree`） |
| 渲染 | Metal（macOS）、Direct3D/WGPU（Windows）、Skia/Vulkan（Linux）、WebGPU（Web） |
| 擴充執行環境 | Wasmtime（WASM 沙箱） |
| 客戶端資料庫 | SQLite，透過 `sqlez`（自訂非同步包裝器） |
| 伺服器資料庫 | PostgreSQL，透過 sea-orm + sqlx |
| 伺服器儲存 | AWS S3（靜態資源）、AWS Kinesis（事件串流） |
| 網路通訊 | WebSocket（async-tungstenite + TLS）、Protobuf 序列化 |
| 語音/視訊 | LiveKit（WebRTC） |
| Git 整合 | libgit2（vendored） |
| LSP / DAP | Language Server Protocol + Debug Adapter Protocol 客戶端 |
| 測試 | Rust `#[test]` / `#[gpui::test]`、確定性非同步執行器 |
| 建置工具 | Cargo 工作區（resolver v2）、xtask、自訂 CI 腳本 |

---

## 目錄結構

```
zed/
├── crates/                  # 160+ Rust crates（主要原始碼）
│   ├── zed/                 # 主應用程式二進位（預設工作區成員）
│   ├── gpui/                # GPU 加速 UI 框架核心
│   ├── gpui_macos/          # macOS 平台後端
│   ├── gpui_windows/        # Windows 平台後端
│   ├── gpui_linux/          # Linux 平台後端
│   ├── gpui_web/            # Web/WASM 平台後端
│   ├── gpui_wgpu/           # WGPU 渲染層
│   ├── editor/              # 主要文字編輯器元件
│   ├── display_map/         # 延遲顯示轉換（自動換行、折疊、嵌入）
│   ├── multi_buffer/        # 多檔案統一緩衝區視圖
│   ├── text/                # CRDT 文字緩衝區（Lamport 時鐘）
│   ├── rope/                # 寫入時複製的 rope 資料結構
│   ├── sum_tree/            # 帶摘要的 B-tree（基礎資料結構）
│   ├── language/            # 語言抽象 + tree-sitter
│   ├── language_model/      # 抽象 LLM 介面 trait
│   ├── language_models/     # 提供者註冊表 + 初始化
│   ├── anthropic/           # Anthropic Claude 整合
│   ├── open_ai/             # OpenAI 整合
│   ├── google_ai/           # Google Gemini 整合
│   ├── bedrock/             # AWS Bedrock 整合
│   ├── ollama/              # 本地 Ollama 整合
│   ├── agent/               # 自主代理系統
│   ├── project/             # 專案模型（LSP、診斷、補全）
│   ├── workspace/           # 面板、停靠區、工作區佈局
│   ├── worktree/            # 目錄監控 + 檔案系統抽象
│   ├── collab/              # 協作伺服器（二進位 + 函式庫）
│   ├── client/              # 協作伺服器的桌面客戶端
│   ├── rpc/                 # WebSocket + Protobuf RPC 層
│   ├── proto/               # Protobuf 訊息定義（200+ 種類型）
│   ├── lsp/                 # LSP 客戶端
│   ├── dap/                 # DAP 客戶端
│   ├── git/                 # Git 整合（libgit2）
│   ├── terminal/            # 終端機模擬器（alacritty_terminal）
│   ├── settings/            # 設定系統 + 熱重載
│   ├── db/                  # SQLite 應用程式資料庫
│   ├── extension_host/      # Wasmtime WASM 擴充執行環境
│   ├── extension_api/       # 公開擴充 API 介面（Apache-2.0）
│   ├── theme/               # 主題管理
│   ├── ui/                  # 高階 UI 元件函式庫
│   └── ...                  # 100+ 額外的功能與工具 crate
├── extensions/              # 內建語言擴充（glsl、html、proto、test）
├── assets/                  # 靜態資源（字型、圖示、主題、快捷鍵、音效）
├── docs/                    # 面向使用者的文件（mdBook 格式）
├── script/                  # 建置與工具腳本（bash、PowerShell、Python）
├── tooling/                 # 開發者工具（合規檢查器、xtask、效能工具）
├── ci/                      # CI/CD 管線設定
├── nix/                     # Nix 打包與開發環境
├── legal/                   # 授權文件
├── .config/                 # 專案級設定檔
├── .github/                 # GitHub Actions 工作流程與範本
├── .zed/                    # 貢獻者用的 Zed 編輯器設定
└── Cargo.toml               # 工作區根目錄（resolver v2，edition 2024）
```

---

## 目錄角色與慣例

### `/crates`
**角色：** 整個應用程式以 160+ 個細粒度 Rust crate 的形式存放於此，每個 crate 負責一個完整的職責。
**慣例：** 任何地方均不允許 `mod.rs` 檔案。每個 crate 必須在 `Cargo.toml` 中設定 `[lib] path = "src/{crate_name}.rs"`。Crate 命名遵循明確的後綴慣例：`_ui` 用於 UI 元件，`_client` 用於 HTTP/API 客戶端，`_server` 用於伺服器二進位/函式庫，`_macros` 用於程序宏，`_settings` 用於設定結構體，`_types` 用於共用類型定義。所有 crate 繼承根工作區的 `edition.workspace = true`。

### `/crates/zed`
**角色：** 主應用程式二進位與入口點；將所有其他 crate 組裝成可執行的編輯器。
**慣例：** 這是唯一的預設工作區成員。啟動邏輯位於 `src/main.rs`。此處不放置業務邏輯——立即委派給各功能 crate。

### `/crates/gpui`
**角色：** 自訂的 GPU 加速即時模式 UI 框架，整個編輯器皆建構於其上。
**慣例：** 平台實作隔離於 `gpui_macos`、`gpui_windows`、`gpui_linux`、`gpui_web`、`gpui_wgpu`。`gpui` 核心邏輯中不得包含平台特定程式碼。工具類型放在 `gpui_util` 和 `gpui_shared_string`。

### `/crates/text`
**角色：** 底層 CRDT 文字緩衝區；以無衝突合併語義作為檔案內容的授權表示。
**慣例：** 使用 Lamport 時鐘為每個副本生成唯一操作 ID。絕不依賴更高層的編輯器概念。基於快照的讀取（`Buffer::snapshot()`）必須保持無鎖。

### `/crates/editor`
**角色：** 主要文字編輯 UI 元件；擁有游標/選取狀態、顯示轉換、輸入處理，以及 LSP 驅動的裝飾。
**慣例：** 包含 `display_map` 作為整合子模組。`Editor` 結構體包裝 `MultiBuffer`，而非原始 `Buffer`。所有位置運算在顯示空間中以 `DisplayPoint` 進行；需要時才明確轉換至緩衝區位置。

### `/crates/language_model` 與 `/crates/language_models`
**角色：** `language_model` 定義抽象 `LanguageModel` trait（串流補全、工具使用、Token 計數、思考模式）。`language_models` 是初始化並註冊所有具體提供者的提供者註冊表。
**慣例：** 每個 LLM 提供者是各自獨立的 crate（如 `anthropic`、`open_ai`、`bedrock`）。沒有共用的 HTTP 客戶端層——每個提供者自行實作。所有提供者必須支援 `api_url` 覆蓋，以便自架部署。

### `/crates/collab`
**角色：** 協作伺服器——同時包含函式庫與二進位。處理多人編輯、線上狀態、文件同步和頻道管理。
**慣例：** 這是唯一使用 PostgreSQL（透過 sea-orm + sqlx）的 crate。伺服器端會話狀態位於 `crates/collab/src/rpc.rs`。不得在此混入客戶端程式碼。

### `/crates/rpc` 與 `/crates/proto`
**角色：** `rpc` 是客戶端與伺服器共用的雙向 WebSocket + Protobuf 傳輸層。`proto` 包含 `.proto` 定義與宏自動生成的訊息類型（200+ 種）。
**慣例：** 協定版本不符回傳 HTTP 426，必須視為硬性錯誤。訊息處理器以 `TypeId` 分派，而非字串比對。

### `/crates/extension_host`
**角色：** 基於 Wasmtime 的沙箱，在執行期載入並執行第三方擴充。
**慣例：** 擴充編譯為 WASM，且只能透過 `extension_api` 介面（Apache-2.0）進行互動。第三方擴充不允許執行原生程式碼。

### `/crates/settings`
**角色：** 統一的設定系統，具備熱重載、Schema 生成和類型安全存取。
**慣例：** 使用 `RegisterSetting` 宏新增設定；不得手寫樣板程式碼。設定結構體必須衍生 `JsonSchema`。

### `/extensions`
**角色：** 隨編輯器一同發布的內建語言擴充（glsl、html、proto、test-extension）。
**慣例：** 每個擴充是指向 WASM 產物的 `extension.toml` 清單，而非 Rust crate。此目錄中的擴充在建置時打包，但遵循與第三方擴充相同的 WASM 沙箱規則。

### `/assets`
**角色：** 在執行期從磁碟載入的靜態資源——字型、圖示、主題、音效、快捷鍵，以及預設設定範本。
**慣例：** 資源不嵌入二進位檔；在執行期從相對於二進位的 `assets/` 目錄解析。不得在 crate 邏輯中硬編碼資源路徑；使用資源載入基礎設施。

### `/docs`
**角色：** mdBook 格式的面向使用者文件。
**慣例：** 架構與內部開發者文件（如本檔案）也存放於此。不得將 rustdoc 生成的 API 參考放在這裡。

### `/script`
**角色：** 建置腳本、發布自動化與開發者工具（bash、PowerShell、Python、Julia）。
**慣例：** 腳本不屬於 Cargo 建置圖。平台特定腳本使用對應的副檔名（`.sh`、`.ps1`）。

### `/tooling`
**角色：** 面向開發者的建置工具：合規檢查器、xtask 執行器、效能工具。
**慣例：** 使用 `xtask` 模式，透過 `cargo xtask <task>` 執行自訂建置任務。

### `/ci`
**角色：** CI/CD 管線定義（GitHub Actions）。

### `/nix`
**角色：** 用於可重現開發環境與打包的 Nix flake 定義。

---

## 關鍵模組

| 模組 | 路徑 | 角色 |
|------|------|------|
| 應用程式入口 | `crates/zed/src/main.rs` | CLI 參數解析、GPUI 應用程式啟動、建立第一個工作區 |
| GPUI 框架 | `crates/gpui/src/gpui.rs` | 實體系統、佈局引擎、事件分派、渲染上下文 |
| CRDT 緩衝區 | `crates/text/src/text.rs` | Lamport 時鐘 CRDT 緩衝區、編輯操作、快照 API |
| Rope | `crates/rope/src/rope.rs` | 寫入時複製的 rope，用於高效文字儲存 |
| Sum Tree | `crates/sum_tree/src/sum_tree.rs` | 帶摘要的 B-tree；支撐所有索引結構 |
| Multi Buffer | `crates/multi_buffer/src/multi_buffer.rs` | 跨多個文字緩衝區的摘錄式視圖 |
| Editor | `crates/editor/src/editor.rs` | 主要編輯器元件：選取、顯示、補全、LSP UI |
| Display Map | `crates/editor/src/display_map/` | 延遲轉換：自動換行、折疊、嵌入、Tab 展開 |
| Language | `crates/language/src/language.rs` | Tree-sitter 整合、語法高亮、折疊 |
| Language Model Trait | `crates/language_model/src/language_model.rs` | 抽象 LLM 介面（串流、工具、思考） |
| Provider Registry | `crates/language_models/src/language_models.rs` | 初始化並註冊所有 LLM 提供者 |
| Agent | `crates/agent/src/agent.rs` | 使用 LLM + ACP 工具呼叫的自主代理迴圈 |
| Project | `crates/project/src/project.rs` | LSP 生命週期、診斷、補全、Worktree 聚合 |
| Worktree | `crates/worktree/src/worktree.rs` | 目錄快照、檔案監控（`notify` crate） |
| Workspace | `crates/workspace/src/workspace.rs` | 面板/停靠區佈局、Item trait、Modal/Toast 管理 |
| RPC Peer | `crates/rpc/src/peer.rs` | 雙向訊息路由、保活機制（ping 每 1s，逾時 10s） |
| Collab Server | `crates/collab/src/rpc.rs` | WebSocket 伺服器、認證、TypeId 分派的訊息處理器 |
| Client | `crates/client/src/client.rs` | 桌面協作客戶端、認證、傳入訊息路由 |
| Proto Messages | `crates/proto/src/proto.rs` | 200+ 個宏自動生成的 Protobuf 訊息類型 |
| Extension Host | `crates/extension_host/src/extension_host.rs` | Wasmtime WASM 沙箱、擴充生命週期 |
| Settings | `crates/settings/src/settings.rs` | 類型安全設定註冊表、熱重載、Schema 生成 |
| App Database | `crates/db/src/db.rs` | 透過 sqlez 的 SQLite 客戶端端持久化 |
| Theme | `crates/theme/src/theme.rs` | 顏色主題載入與應用 |
| LSP Client | `crates/lsp/src/lsp.rs` | Language Server Protocol 客戶端 |
| DAP Client | `crates/dap/src/dap.rs` | Debug Adapter Protocol 客戶端 |
| Git | `crates/git/src/git.rs` | 基於 libgit2 的 Git 操作 |
| Terminal | `crates/terminal/src/terminal.rs` | 由 alacritty_terminal 支撐的終端機模擬器 |

---

## 資料流

### 應用程式啟動
```
main() [crates/zed/src/main.rs]
  → 解析 CLI 參數
  → Application::new(平台後端)
  → 初始化崩潰處理器、日誌、檔案系統
  → 載入 AppDatabase（SQLite）
  → 啟動背景任務（session、system_id、install_id）
  → 設定熱重載
  → initialize_workspace()
  → app.run() → GPUI 事件迴圈
```

### 開啟檔案
```
使用者開啟檔案
  → Project::open_buffer(path)
  → Worktree 尋找/建立目錄條目
  → 語言註冊表選擇語法解析器
  → Buffer::new() 使用 CRDT 引擎
  → MultiBuffer::push_excerpts()
  → Editor::new(MultiBuffer)
  → Pane::add_item(Editor)
  → EditorElement 透過 GPUI 渲染
```

### 文字編輯（本地）
```
按鍵
  → GPUI 輸入事件 → Editor::handle_input()
  → MultiBuffer::edit()
  → Buffer::apply_op() 遞增 Lamport 時鐘
  → EventEmitter 廣播 BufferEvent::Edited
  → Editor 使 DisplayMap 快取失效
  → GPUI 排程重繪 → EditorElement::paint()
```

### 協作編輯（遠端）
```
本地編輯（同上）
  → Buffer::apply_op() 生成帶唯一 replica_id + 時間戳的操作
  → Project 透過 Client → RPC Peer 發送操作
  → WebSocket 封包 → Collab 伺服器（crates/collab）
  → 伺服器廣播給其他連線的對等端
  → 遠端客戶端接收 → 透過 ProtoMessageHandlerSet 路由
  → Buffer::apply_remote_op() — CRDT 無衝突合併
```

### LLM 請求
```
使用者觸發 AI 功能（代理、補全、行內編輯）
  → LanguageModel::stream_completion(request)
  → 提供者 crate（如 anthropic）發送 HTTPS 請求
  → Server-sent events 串流回 Token
  → 透過非同步通道串流至代理/編輯器
  → GPUI 增量重渲染
```

### 協作伺服器訊息分派
```
客戶端 WebSocket 封包
  → rpc::Conn 讀取封包 → MessageStream 反序列化 Protobuf
  → Peer 路由：
      回應？ → 匹配的請求通道
      串流？ → 匹配的串流通道
      新訊息？ → 依 TypeId 廣播至 ProtoMessageHandlerSet
  → 處理器協程在 FuturesUnordered 中執行（前台）
    或分離任務（背景）
  → 回應序列化 → 傳出通道 → WebSocket 寫入
```

---

## 入口點

| 場景 | 入口點 |
|------|--------|
| 編輯器二進位 | `crates/zed/src/main.rs::main()` |
| 協作伺服器 | `crates/collab/src/main.rs::main()`（Axum HTTP + WebSocket） |
| 遠端伺服器代理 | `crates/remote_server/src/main.rs` |
| CLI 工具 | `crates/cli/src/main.rs` |
| 擴充 CLI | `crates/extension_cli/src/main.rs` |
| 編輯預測 CLI | `crates/edit_prediction_cli/src/main.rs` |
| 自動更新助手 | `crates/auto_update_helper/src/main.rs` |
| Xtask 建置執行器 | `tooling/xtask/src/main.rs` |

---

## 外部整合

| 服務 | 用途 | 模組 |
|------|------|------|
| Zed Cloud（zed.dev） | 協作、認證、頻道線上狀態 | `crates/client`、`crates/collab` |
| Anthropic | Claude LLM 補全、工具使用、思考模式 | `crates/anthropic` |
| OpenAI | GPT 補全 | `crates/open_ai` |
| AWS Bedrock | 託管 LLM 推論（IAM/SSO 認證） | `crates/bedrock` |
| Google Gemini | Gemini 補全 | `crates/google_ai` |
| Mistral | Mistral 補全 | `crates/mistral` |
| DeepSeek | DeepSeek 補全 | `crates/deepseek` |
| Ollama | 本地模型服務 | `crates/ollama` |
| LM Studio | 本地模型服務 | `crates/lmstudio` |
| OpenRouter | 模型聚合服務 | `crates/open_router` |
| Vercel AI Gateway | AI 路由閘道器 | `crates/vercel_ai_gateway` |
| X.AI（Grok） | Grok 補全 | `crates/x_ai` |
| GitHub Copilot Chat | Copilot 補全（內建） | `crates/copilot_chat`、`crates/copilot` |
| LiveKit | WebRTC 語音/視訊通話 | `crates/livekit_client`、`crates/livekit_api` |
| GitHub / GitLab / Bitbucket | 代管平台 API（PR、blame） | `crates/git_hosting_providers` |
| AWS S3 | 伺服器端靜態資源儲存 | `crates/collab`（僅伺服器） |
| AWS Kinesis | 伺服器端事件串流 | `crates/collab`（僅伺服器） |
| PostgreSQL | 伺服器端關聯式資料庫 | `crates/collab`、`crates/db` |
| 系統金鑰鏈 | 憑證儲存 | `crates/credentials_provider` |
| OAuth 提供者 | 使用者認證 | `crates/oauth_callback_server` |

---

## 注意事項

- **禁止使用 `mod.rs`：** 本專案強制規定任何 crate 均不得使用 `mod.rs`。每個 crate 入口點必須是 `src/{crate_name}.rs`。這是由 `tooling/` 中的合規工具強制執行的硬性規則。

- **資源載入是執行期行為，而非編譯期嵌入：** `assets/` 中的資源在執行期從磁碟讀取（相對於二進位的路徑），不嵌入二進位。缺少資源在執行期失敗，而非編譯期。

- **`sum_tree` 在各處都是基礎：** 文字範圍、診斷、選取和檔案樹均依賴 `crates/sum_tree`。理解其基於摘要的 B-tree 是理解程式碼庫中任何索引結構的前提。

- **`Editor` 編輯的是 `MultiBuffer`，永遠不是原始 `Buffer`：** 編輯器始終操作 `MultiBuffer`，它可能交錯來自多個檔案的摘錄片段。單檔案編輯只是只有一個摘錄的 `MultiBuffer`。這就是為什麼差異視圖和搜尋結果能以可編輯的行內視圖呈現。

- **顯示位置與緩衝區位置是不同的座標空間：** `DisplayPoint`（自動換行和折疊後的螢幕行/列）與 `Point`（緩衝區行/列）不可互換。始終透過 `DisplayMap` 明確轉換。

- **CRDT 合併以副本為作用域：** 每個 `Buffer` 有一個 `replica_id`。操作以 `(replica_id, lamport_timestamp)` 配對作為身份。兩個副本可以同時生成相同的邏輯編輯；CRDT 確保不需要鎖或伺服器往返即可收斂。

- **基於快照的讀取是無鎖的：** 幾乎所有讀取操作都接收不可變的 `Snapshot` 類型。活躍的 `Buffer` 或 `Worktree` 可在讀取快照的同時被並發修改。不要持有快照超過必要時間——它會固定記憶體。

- **Collab 的 `FuturesUnordered` 是為了防止死鎖，而非僅調整順序：** 前台訊息處理器池使用 `FuturesUnordered`，使得第 N 條訊息的緩慢處理器不會阻塞第 N+1 條訊息，明確避免透過 RPC 協定產生的 A 等待 B 等待 A 式死鎖。

- **協定版本不符是硬性錯誤（HTTP 426）：** 沒有向後相容層。客戶端與伺服器必須使用相同的協定版本。這意味著 Zed 的自動更新機制是正確性保證的必要組成部分。

- **LLM 提供者各有自己的 HTTP 客戶端與 `api_url` 設定：** AI 提供者沒有共用的 HTTP 抽象層。每個提供者 crate 自行實作 HTTP 邏輯，且所有提供者均支援在設定中覆蓋端點 URL——無需修改程式碼即可啟用自架或代理部署。

- **`collab` 同時是函式庫和伺服器二進位：** `crates/collab` 編譯為函式庫（用於整合測試）以及二進位（實際伺服器）。僅供伺服器的基礎設施（PostgreSQL、S3、Kinesis）完全限定在此處，絕不能洩漏到客戶端 crate。

- **擴充市集提供者是動態的：** 透過 Wasmtime 載入的第三方擴充可在執行期透過 `ExtensionInstalled` 事件動態註冊新的 LLM 提供者。提供者註冊表在編譯期並非靜態固定的。

- **`ZED_SERVER_URL` 和 `ZED_RPC_URL` 環境變數會覆蓋所有設定：** 這些是在啟動時一次性求值的 `LazyLock` 靜態變數。設定它們（例如在容器中）會完全重導向協作與 RPC 流量，無論使用者設定檔的內容為何。

- **PR 發布備註為強制要求：** 每個 Pull Request 必須包含「Release Notes:」章節，以一條項目符號使用以下其中之一：`Added`（新增）、`Fixed`（修復）、`Improved`（改進）或 `N/A`。CI 會強制執行此規則。
