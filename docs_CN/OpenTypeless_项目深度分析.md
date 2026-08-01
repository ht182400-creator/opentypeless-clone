# OpenTypeless 项目深度分析文档

> 文档类型：项目分析（需求 / 框架 / 实现细节 / 使用）
> 分析视角：文档专家（宏观定位 + 微观实现）
> 分析时间：2026-07-31
> 代码基线：`main` 分支（工作区干净，最近一次提交 `9ff6b94 docs(repo): add VISION.md to ground the project narrative`）
> 适用读者：新接手的高级工程师、技术负责人、产品经理、开源贡献者

---

## 0. 摘要（TL;DR）

OpenTypeless 是一款**跨平台桌面语音输入 / 润色 / 翻译**应用，是商业闭源产品 **Typeless**（typeless.ai）的开源版本。技术形态为 **Tauri 2 + React + Rust**：前端用 React + TypeScript 提供三个窗口（主窗口 / 浮窗 Capsule / Ask 对话窗），后端用 Rust 承载音频采集、STT 识别、LLM 润色、键盘/剪贴板输出、活动应用检测等系统级能力。

核心卖点：**全局热键随时口述 → 实时转写 → 可选 LLM 润色（应用感知上下文 + 词典 + 纠错规则）→ 直接键入光标处或写入剪贴板**；并通过 "App-aware writing"（按当前活动应用切换写作风格）、"Voice Intent"（语音意图路由）、"Scenes"（写作场景预设）、词典与纠错规则、Ask 对话、本地优先 + 可选云端管理，构成完整的生产力闭环。

这是一个**工程完成度很高、抽象清晰、可扩展性强**的项目，但**文档散落在 `docs/` 下的 20+ 份设计规格（spec）中，缺少一份面向全局的"地图"**——本文档正是补上这张地图。

---

## 1. 宏观视角：项目定位与需求

### 1.1 项目是什么（产品定位）

| 维度 | 说明 |
|---|---|
| 产品形态 | 桌面端（Windows / macOS / Linux）原生应用 |
| 开源身份 | Typeless 商业产品的开源版（OpenTypeless = Open + Typeless） |
| 核心价值 | 键盘替代——"说话即输入"，并智能润色/翻译 |
| 目标用户 | 作家、程序员、客服、跨境沟通者、效率/无障碍需求者 |
| 竞品参照 | SuperWhisper、MacWhisper、Wispr Flow、Voice In 等 |
| 商业模式 | 自托管 / BYOK（自带密钥）免费；托管云（OpenTypeless Cloud）订阅增值（AppSumo 终身版已上线） |

### 1.2 功能需求（从代码反推的真实需求）

1. **全局语音听写（Dictation）**：任意应用内按热键开始说话，松开/再按停止，文本落到当前光标处。
2. **语音润色（Polish）**：识别结果经 LLM 按"应用类别 + 场景 + 词典 + 纠错规则 + 语气"润色。
3. **语音意图路由（Voice Intent）**：口述中以自然语言触发动作（编辑选中文本 / 翻译 / 打开 Ask / 执行语音动作 / 搜索）。
4. **翻译（Translate）**：听写时实时翻译为目标语言（最多 5 个目标，见 `appStore.syncTranslationConfig`）。
5. **Ask 对话（Ask Anything）**：独立浮窗，边听边问、流式回答、可把答案送入当前应用。
6. **应用感知写作（App-aware writing）**：检测当前活动 App → 归类（Email / Chat / Code / Document / General）→ 套用对应风格 Prompt。
7. **词典（Dictionary）**：自定义术语表，保障专有名词、缩写、产品名正确。
8. **纠错规则（Correction Rules）**：正则式替换，自动修正识别错误（如 "麦克风"→"麦克风"，品牌误识）。
9. **写作场景（Scenes）**：可保存的写作风格预设（语气、长度、格式），本地优先（`docs/2026-06-30-local-first-custom-scenes-spec.md`）。
10. **历史（History）**：本地 SQLite 记录每次转录/润色，可检索、复用。
11. **云账户与同步**：登录、订阅校验、云端备份/恢复（allow-list 合并策略）、用量配额。
12. **系统托盘 + 浮窗（Capsule）**：常驻迷你窗，显示录音时长、状态、快捷切换。
13. **自动化更新（Updater）**：Tauri updater 推送新版本。

### 1.3 非功能需求

- **跨平台一致性**：同一份 React 代码驱动三窗口；Rust 层对每个平台（mac/win/linux）分别实现音频、热键、可访问性权限、应用检测、键盘输出。
- **隐私 / 本地优先**：默认所有数据本地（SQLite + store 插件）；云为可选。凭证存系统钥匙库（keyring），不落明文。
- **低延迟**：STT 采用流式（WebSocket）provider 实时回传 partial；LLM 流式 chunk 直插光标。
- **可扩展**：STT / LLM 均以 trait 抽象 + 工厂函数，新增 provider 仅需实现 trait 并注册。
- **可诊断**：每个 provider 提供 connection/benchmark 测试、诊断命令、热键注册失败上报。
- **多语言 UI**：`react-i18next`，locales 含 10 种语言（en / zh-Hans / zh-Hant / ja / ko / de / fr / es / it / pt 等，见 `i18n/locales`）。
- **无障碍（Accessibility）**：macOS 需要辅助功能权限才能注入文本；前端 `checkAccessibilityPermission` 显式引导。

---

## 2. 整体框架（架构）

### 2.1 技术栈总览

```
┌──────────────────────────────────────────────────────────────┐
│  表现层（Webview，React 18 + TS + Vite + Tailwind + framer）   │
│  三窗口：#main  |  #capsule  |  #ask                          │
│  状态：zustand (appStore / authStore)   国际化：react-i18next  │
└───────────────────────────┬──────────────────────────────────┘
                            │ Tauri IPC  (invoke + emit 事件)
┌───────────────────────────┴──────────────────────────────────┐
│  Rust 后端（Tauri 2 命令 + 后台线程）                          │
│  lib.rs 注册 60+ #[tauri::command]                            │
│  ┌────────┐ ┌──────┐ ┌─────┐ ┌────────┐ ┌────────┐ ┌──────┐  │
│  │ audio  │ │ stt  │ │ llm │ │ output │ │app_det │ │voice │  │
│  │(cpal)  │ │(8家) │ │(2家)│ │(enigo) │ │ector   │ │_intent│ │
│  └───┬────┘ └──┬───┘ └──┬──┘ └───┬────┘ └───┬────┘ └──┬───┘  │
│      └─────────┴── pipeline.rs（状态机编排）───┴────────┘     │
│  storage(SQLite+store) │ credentials(keyring) │ hotkey/tray   │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 进程与窗口模型

- **三个 Webview 窗口**，由 `window.location.hash` 分流（`src/App.tsx:227-232`）：
  - `#capsule` → `CapsuleApp`：浮窗，仅 `Capsule` 组件（录音计时、状态）。
  - `#ask` → `AskApp`：Ask 对话窗 + Toast。
  - 默认 → `MainApp`：主窗口，内部按 `useRoute()` 渲染 `home / settings / history / upgrade / account` 五路由（`src/lib/router.ts`）。
- **Rust 端单进程多 OS 线程**：音频采集跑在独立 cpal 线程；STT/LLM 走 tokio 异步；管线状态机在 `PipelineHandle` 内协调。
- **事件总线**：后端用 `app_handle.emit("pipeline:state" | "stt:partial" | "stt:final" | "llm:chunk" | "pipeline:error" | "pipeline:timing", ...)` 向前端广播；前端 `useTauriEvents` 订阅。

### 2.3 后端模块职责（20 个顶层模块）

| 模块 | 文件 | 职责 |
|---|---|---|
| `audio` | `audio/capture.rs`, `audio/mod.rs` | cpal 采集，16kHz 单声道 16bit PCM，独立线程 |
| `stt` | `stt/mod.rs` + 8 个 provider | STT 抽象 + Deepgram/AssemblyAI/Cloud/Whisper/阿里Qwen3/Apple/火山豆包 |
| `llm` | `llm/mod.rs` + `openai.rs` `cloud.rs` | LLM 抽象 + Prompt 构造 + 上下文策略 |
| `output` | `output/mod.rs` `keyboard.rs` `clipboard.rs` `windows_sendinput.rs` | 键盘注入 / 剪贴板 / Windows SendInput |
| `storage` | `storage/mod.rs`（最大文件 ~130KB） | ConfigManager + HistoryStore + DictionaryStore（SQLite） |
| `app_detector` | `app_detector/*` `platform/{macos,windows,linux}.rs` | 活动应用检测 + 分类 |
| `voice_intent` | `voice_intent/*` `grammar/{en,zh_hans,zh_hant}.rs` | 意图识别 + 执行 |
| `commands` | `commands/*.rs` | 12 个命令分组文件，前端调用入口 |
| `credentials` | `credentials.rs` | keyring 读写 + 旧版迁移 |
| `hotkey` / `native_hotkey` | — | 热键解析 + 运行时 + `HotkeySupervisor` 重试 |
| `pipeline` | `pipeline.rs` | 核心状态机编排 |
| `tray` / `selection` / `recording_deadline` / `platform` / `error` | — | 托盘、选中文本、录音截止、平台判定、错误类型 |

### 2.4 Tauri 命令清单（前端 ↔ 后端契约）

`src-tauri/src/lib.rs:1151` 的 `generate_handler!` 注册了 **60+ 命令**，大致分组：

- **录音/管线**：`start_recording` / `stop_recording` / `abort_recording`
- **Ask**：`ask_anything` / `start_ask_dictation` / `stop_ask_dictation` / `start_ask_flow` / `stop_ask_flow` / `take_pending_ask_message` / `show_ask_window`
- **翻译**：`set_active_translation_target`
- **App 映射**：`get_latest_mapping_candidate` / `list_custom_app_mappings` / `save_custom_app_mapping` / `update_custom_app_mapping` / `set_custom_app_mapping_enabled` / `delete_custom_app_mapping` / `reset_custom_app_mappings` / `set_family_scene_assignment`
- **配置/凭证**：`get_config` / `update_config` / `get_credential_status` / `read_credential` / `set_credential` / `clear_credential` / `migrate_legacy_credentials`
- **STT/LLM 诊断**：`get_stt_provider_diagnostics` / `get_stt_recording_capability` / `test_stt_connection` / `bench_stt_connection` / `test_llm_connection` / `bench_llm_connection` / `get_llm_model_capability` / `fetch_llm_models`
- **历史**：`get_history` / `clear_history`
- **备份**：`restore_backup_data`
- **词典**：`get_dictionary` / `add_dictionary_entry` / `update_dictionary_entry` / `remove_dictionary_entry` / `get_correction_rules` / `add_correction_rule` / `update_correction_rule` / `remove_correction_rule` / `set_correction_rule_enabled` / `preview_dictionary_import` / `commit_dictionary_import` / `export_dictionary_json` / `export_dictionary_csv`
- **热键/权限**：`update_hotkey` / `update_ask_hotkey` / `pause_hotkey` / `check_accessibility_permission` / `request_accessibility_permission` / `request_browser_access`

---

## 3. 微观视角：核心实现细节

### 3.1 核心管线状态机（`pipeline.rs`）

管线是一个**事件驱动的异步状态机**，状态用 `AtomicU8` 表示（`pipeline.rs:147-156`）：

```rust
pub enum PipelineState {
    Idle, Preparing, Recording, Transcribing,
    Polishing, Outputting, AskRecording, AskThinking,
}
```

完整数据流（听写场景）：

1. **热键触发** → `PipelineHandle::start_with_options(&PipelineStartOptions)`（`pipeline.rs:1063`）。
2. **预加载**：config / 应用上下文 / 词典 / 纠错规则；fail-fast 校验 API key。
3. **并发启动**：`audio::AudioCaptureHandle::start()` 与 `SttProvider::connect()`；`await_recording_startup` 30s 超时。
4. **录音中**：音频 chunk 实时喂 `SttProvider::send_audio()`；流式 provider 通过 `recv_transcript()` 回传 `TranscriptEvent::Partial`，后端 emit `stt:partial`。
5. **停止** → `PipelineHandle::stop()`（`pipeline.rs:1751`）：捕获选中文本 → 关音频 → `disconnect()` 取最终文本 → emit `stt:final`。
6. **意图路由** → `route_pipeline_voice_intent()`（`pipeline.rs:318`）：判断是纯听写 / 编辑选中 / 翻译 / Ask / 语音动作。
7. **润色** → `polish_text()`（`pipeline.rs:2052`）：构造 `PolishRequest` → `LlmProvider::polish()`（流式可选）。`on_chunk` 回调 emit `llm:chunk`，并可经 `streaming_sender` 直插光标。
8. **执行意图** → `execute_voice_intent()`：经 `VoiceExecutionBackend` 输出（键盘 / 剪贴板 / Windows SendInput / 弹窗 / 搜索）。
9. **写历史** → 回 `Idle`。

录音时长限制由各 STT provider 的 `recording_limit_override_seconds()` 决定（如 Apple Speech 有上限），否则用配置 `max_recording_seconds`（`pipeline.rs:1445` 的 `resolve_recording_limit`）。

### 3.2 STT 抽象与 8 家 Provider（`stt/mod.rs`）

统一抽象（`stt/mod.rs:55-69`）：

```rust
#[async_trait]
pub trait SttProvider: Send + Sync {
    async fn connect(&mut self, config: &SttConfig) -> Result<(), AppError>;
    async fn send_audio(&mut self, chunk: &[u8]) -> Result<(), AppError>;
    async fn recv_transcript(&mut self) -> Result<Option<TranscriptEvent>, AppError>;
    async fn disconnect(&mut self) -> Result<Option<String>, AppError>;
    fn name(&self) -> &str;
}
```

`SttConfig` 默认值明确：**`sample_rate: 16000`**，单声道 16bit PCM（`stt/mod.rs:31-44`）。

`create_provider(name, custom_whisper_config, client)` 工厂分发（`:71-118`），支持的 provider：

| Provider 名 | 类 | 模式 | 上传格式 |
|---|---|---|---|
| `cloud` | `CloudSttProvider` | 自托管 Cloud（token 鉴权） | 短=WAV / 长=**OggOpus**（受管编码） |
| `deepgram` | `DeepgramProvider` | 实时 WebSocket（Nova-3） | 流式 partial |
| `assemblyai` | `AssemblyAiProvider` | 实时 WebSocket v3 | 流式 |
| `aliyun-qwen3-asr` | `AliyunQwen3AsrProvider` | 阿里云实时 ASR | 流式 |
| `volcengine-doubao` | `VolcengineDoubaoProvider` | 火山引擎豆包实时 ASR | 流式 |
| `apple-speech` | `AppleSpeechProvider` | macOS 本机离线 | 离线 |
| `custom-whisper` | `WhisperCompatProvider` | 本地/自托管 OpenAI 兼容 | 整段 WAV |
| 其它"known whisper" | `WhisperCompatProvider` | groq / glm-asr / siliconflow / openai 等 | 整段 WAV |

**两种实现范式**：
- **流式（real-time）**：`recv_transcript()` 真返回 `Partial/Final/...` 事件；Deepgram/AssemblyAI/阿里/火山/Apple 属此。
- **整段（file-based）**：`send_audio()` 仅缓冲；`recv_transcript()` 用 `std::future::pending().await` 永不返回（避免 select 忙轮询）；**最终文本在 `disconnect()` 返回**。Cloud / Whisper 兼容属此。

**音频编码**：本地 Whisper 全部 WAV 整段上传（`build_wav()` 手工写 44 字节 WAV 头）；Cloud provider 仅在 `disconnect()` 时按 `managed_audio` 配置：短录音/非 16k/编码失败 → WAV，长录音且 Opus 可用 → OggOpus（压缩至 `max_audio_bytes = 24MB` 内）。上限 `MAX_AUDIO_BYTES = 24 * 1024 * 1024`（约 12.5 分钟 @16k mono）。

### 3.3 LLM 抽象与 Prompt 构造（`llm/mod.rs`）

统一抽象（`llm/mod.rs:79-89`）：

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    async fn polish(&self, config: &LlmConfig, req: &PolishRequest,
                    on_chunk: Option<&ChunkCallback>) -> Result<PolishResponse, AppError>;
    fn name(&self) -> &str;
}
```

- 默认 `LlmConfig`：**`provider="zhipu"`, `model="glm-4.7"`**，`base_url="https://open.bigmodel.cn/api/paas/v4"`（`llm/mod.rs:23-34`）。
- `create_provider`：仅 `"cloud"` 走 `CloudLlmProvider`（Better Auth 会话 token），**其余一切 provider 走 `OpenAiProvider`**（OpenAI 兼容，含智谱 GLM、Ollama、Groq 等），体现"以 OpenAI 协议为事实标准"的工程取舍。
- **Ollama keyless 支持**：`provider_requires_api_key()` 对 `ollama` 返回 false（`llm/mod.rs:91-93`）。
- `OpenAiProvider`：POST `{base_url}/chat/completions`，SSE 流式；特殊处理 GLM 思考模式；500/超时/连接错误 **最多 3 次指数退避重试**。
- **Prompt 构造**（`llm/prompt.rs`）：按 `context（ContextProfileSummary）/ dictionary / correction_rules / polish_style / mapped_scene_prompt / active_scene_prompt / selected_text / voice_intent` 组装 system prompt，由 11 个带优先级的 section 构成；`context_policy.rs` 负责上下文策略，`model_capabilities.rs` 提供模型能力元数据。

### 3.4 应用感知写作（`app_detector` + LLM Prompt 注入）

`app_detector` 检测当前活动窗口的应用，归类为 `AppType`（`llm/mod.rs:66-74`）：

```rust
pub enum AppType { Email, Chat, Code, Document, #[default] General }
```

- 平台实现：`app_detector/platform/{macos,windows,linux}.rs` 分别用系统 API 取前台应用；`cache.rs` 缓存、减少开销；`profiles.rs` / `registry.rs` / `user_mappings.rs` 支持用户自定义映射（把某 App 手动归到某类别 / 绑定场景）。
- 分类结果（`ContextProfileSummary`）随 `PolishRequest.context` 注入 Prompt，实现"在邮件里更正式、在代码里保留术语、在聊天里更口语"。

### 3.5 语音意图路由（`voice_intent`）

`voice_intent/router.rs` + `grammar/{en,zh_hans,zh_hant}.rs` + `executor.rs` 实现中英文意图识别：

- 意图类型（`VoiceIntent`）：纯听写 / 编辑选中文本 / 翻译 / 打开 Ask / 语音动作（如"搜索 XXX"）。
- `grammar/` 用中英（简/繁）语法规则把口述指令解析为意图；`normalize.rs` 归一化、`search.rs` 支持搜索意图、`guards.rs` 做安全拦截。
- 执行侧 `execute_voice_intent()` 通过 `VoiceExecutionBackend` 分派到键盘/剪贴板/弹窗/搜索。

### 3.6 输出策略（`output`）

`output/mod.rs` 做策略路由：

- `keyboard.rs`：基于 `enigo` 的跨平台键盘模拟，把文本逐字键入光标处（支持流式插入 worker）。
- `clipboard.rs`：基于 `arboard` 写入剪贴板（失败时回退方案）。
- `windows_sendinput.rs`：Windows 专用 `SendInput` 实现，规避修饰键（Ctrl/Alt/Win）卡死问题（`docs/2026-07-17-windows-modifier-safe-output-spec.md` 的 `windows_modifier_guard.rs` 配套）。
- 含 clipboard restore（输出后恢复用户原剪贴板）、输出失败诊断。

### 3.7 存储与凭证（`storage` + `credentials`）

- **配置**：`tauri-plugin-store` 持久化 `AppConfig`（含 STT/LLM 选择、热键、场景、翻译目标等）。
- **历史 / 词典 / 纠错规则**：`rusqlite` 本地 SQLite（`storage/mod.rs` 内的 `HistoryStore` / `DictionaryStore`）。
- **密钥**：`credentials.rs` 用系统钥匙库（keyring）存 API key；提供 `migrate_legacy_credentials` 从旧版明文迁移。
- **字典/规则导入导出**：`dictionary_io.rs` 支持 JSON/CSV；`preview_*_import` / `commit_*_import` 两步导入（先预览后提交）。

### 3.8 热键、托盘与云（`hotkey` / `tray` / `commands/backup.rs`）

- 热键：支持**多绑定列表的 `HotkeyConfig`**（旧式单热键自动同步，见 `appStore.setConfig` 的 `syncHotkeyConfig`）。`HotkeySupervisor` 处理冲突/失败重试；注册失败经 `getHotkeyRegistrationError` 上报前端。
- 托盘：`tray.rs` 构建系统菜单（settings/history/about 直接改 hash 路由）。
- 云/账户：`commands/backup.rs` 的 `restore_backup_data` 实现**allow-list 合并**（只恢复白名单字段，避免覆盖本地敏感配置）；`credentials.rs` 与前端 `authStore` 协同做订阅配额校验。

### 3.9 前端结构（React）

- **状态**：`appStore.ts`（~26.7KB 核心 store，管管线状态、config、词典、连接测试、权限）、`authStore.ts`（~16.5KB，账户/订阅）。
- **关键 hooks**：`useTauriEvents`（订阅后端事件）、`useTheme`、`useCapsuleResize`（解决 macOS 隐藏窗 rAF 暂停问题）、录音 hook。
- **组件**：`Settings`（General/Llm/Stt/Scenes/Dictionary/Account 等多面板）、`Capsule`、`History`、`Onboarding`、`AskPanel`、`HomePage`、`UpgradePage`、`AccountPage`、`UpdatePrompt`、`Toast`。
- **lib 工具**：`tauri.ts`（IPC 封装）、`router.ts`、`constants.ts`、`api client`、`deep-link`、`subscription-refresh-policy`、`voice-error-mapping`。
- **测试**：Vitest，组件/hook/store 均有 `__tests__`，如 `appStore.test.ts`、`useTauriEvents.test.ts`。

---

## 4. 如何使用（面向最终用户与开发者）

### 4.1 最终用户：安装与配置

1. **安装**：从 GitHub Releases / 官网下载对应平台安装包；或自编译（见 4.2）。
2. **首次启动**：走 `Onboarding`（`onboardingCompleted` 控制）——选择 `cloud`（托管云，开箱即用）或 `byok`（自带密钥）。
3. **配置 STT**：设置 → STT 面板，选 provider（Deepgram / AssemblyAI / Cloud / 本地 Whisper / 阿里 / 火山 / Apple 等），填入 API key（存钥匙库）。可用 `test_stt_connection` / `bench_stt_connection` 验证。
4. **配置 LLM（可选润色）**：设置 → LLM 面板，选 OpenAI 兼容 / Cloud / Ollama；填 base_url、model、key。GLM-4.7 为默认。可用 `test_llm_connection` / `bench_llm_connection` / `fetch_llm_models`。
5. **设热键**：如 `Cmd/Ctrl+Shift+Space` 开始听写；`update_hotkey` 持久化。
6. **（macOS）授予辅助功能权限**：`request_accessibility_permission`，否则无法注入文本。
7. **使用**：在任意应用按热键说话 → 松手落字；可在 Capsule 浮窗看计时；设置里配词典/纠错规则/场景；需要翻译时开翻译模式；需要问答时唤出 Ask 窗。

### 4.2 开发者：构建与运行

依赖与脚本见 `package.json` 与 `src-tauri/Cargo.toml`：

```bash
# 安装 JS 依赖
npm install

# 开发模式（Tauri 热重载）
npm run tauri dev

# 生产构建
npm run tauri build

# 前端单测
npm test            # vitest

# Rust 单测 + 检查
cd src-tauri && cargo test && cargo clippy
```

- 多语言构建产物：Tauri 2 打包 Windows（msi/nsis）、macOS（dmg）、Linux（deb/AppImage）。
- 自定义 Whisper：在 STT 面板选 "Local / Custom Whisper"，填 base_url + model（如 `Systran/faster-whisper-large-v3`）。

### 4.3 故障排查（基于诊断命令）

| 现象 | 排查命令/路径 |
|---|---|
| 热键无效 | `getHotkeyRegistrationError`；检查冲突/权限 |
| macOS 不落字 | `check_accessibility_permission`；授予辅助功能 |
| STT 超时 | `get_stt_provider_diagnostics` / `test_stt_connection` / `bench_stt_connection` |
| LLM 润色失败 | `test_llm_connection` / `get_llm_model_capability`；看 500/超时重试日志 |
| 长录音被截断 | 检查 provider 的 `recording_limit_override_seconds` 与配置 `max_recording_seconds` |
| 词典/纠错不生效 | `get_correction_rules` 是否 enabled；导入走 preview→commit |

---

## 5. 测试与质量

- **Rust 测试**：大量 `#[cfg(test)]`，如 `stt/mod.rs` 测试 `create_provider` 工厂（未知 provider 报错、custom-whisper 必须有显式 config、各 provider name 正确）；`pipeline`、`voice_intent`、`storage` 等均有测试。
- **前端测试**：Vitest，覆盖 store/ hook / 组件渲染与事件，并配 `test-setup.ts`。
- **诊断命令体系**：STT/LLM 的 connection + benchmark + capability 形成可量化质量闭环。
- **规范约束**（见仓库规则）：编译 + 单测全绿才允许称"完成"；日志分级；try-catch 不吞异常；禁止硬编码；单文件 ≤ 500 行；改签名查所有调用方。

---

## 6. 既有文档资产（`docs/`）

`docs/` 下已有 **20+ 份设计规格（spec/notes）**，是宝贵但分散的资产，例如：

- `VISION.md`：产品愿景叙事。
- `superpowers/`：14 篇能力/设计笔记。
- 平台发布稳定化：`2026-06-13-platform-release-stabilization-spec.md`
- AppSumo 终身云客户端：`2026-06-20-appsumo-lifetime-cloud-client-spec.md`
- Ask 可靠性：`2026-06-29-ask-anything-reliability-spec.md`
- 本地优先自定义场景：`2026-06-30-local-first-custom-scenes-spec.md`
- OpenLess 系列（开源闭源差异分析）：`2026-07-06-openless-*.md`
- Windows 修饰键安全输出：`2026-07-17-windows-modifier-safe-output-spec.md`
- 桌面复制录音限制 UX：`2026-07-23-desktop-copy-recording-limit-ux-spec.md`
- i18n 设计：`2026-05-22-国际化设计.md` / `2026-05-22-国际化计划.md`
- 发布签名：`release-signing.md`

> 文档专家的观感：规格详尽但**没有索引页**串联；本文档即扮演"总地图"角色，建议后续在 `docs/README.md` 增加目录索引，把设计 spec 与本文分析、使用手册互链。

---

## 7. 评价与建议

### 7.1 优点

1. **抽象清晰、可扩展**：STT/LLM 以 trait + 工厂模式，新增 provider 成本极低（实现 `connect/send_audio/recv_transcript/disconnect` 并注册）。
2. **跨平台工程成熟**：音频、热键、应用检测、键盘输出均按平台分文件实现，并主动处理 macOS 隐藏窗 rAF、Windows 修饰键卡死等坑。
3. **隐私优先**：本地 SQLite + keyring + 本地优先场景，云为可选项。
4. **质量内建**：诊断命令、流式回退、重试、错误处理、测试覆盖齐备。
5. **产品完成度高**：听写/润色/翻译/Ask/词典/场景/历史/云同步/更新一应俱全。

### 7.2 风险与待改进

1. **文档碎片化**：20+ spec 无统一索引，新人无从下手（本文即补此缺口）。
2. **`storage/mod.rs` 单体过大（~130KB）**：违反"单文件 ≤ 500 行"的仓库自定规则，建议按 Config/History/Dictionary 拆分为子模块。
3. **样式硬编码争议**：`App.tsx` 中 `console.error`（如 `console.error('Failed to load config...')`）与规则"禁止 print 做调试"存在张力——前端容错日志属可接受范围，但建议统一到前端日志层。
4. **Cloud provider 耦合**：`cloud` STT/LLM 依赖自托管后端契约，开源社区自建需配套服务端。
5. **测试广度不一**：核心抽象有单测，但部分 UI 交互测试依赖 Vitest 伪 DOM，端到端（真实音频→真实注入）缺集成测试。

### 7.3 给接手者的上手路线

1. 先读 `VISION.md` 与本文第 1–2 节建立全局观。
2. 跟一遍 `pipeline.rs` 状态机（§3.1）——这是系统心脏。
3. 选一个 STT provider（如 Deepgram）通读 `connect/send_audio/recv_transcript/disconnect` 理解流式范式。
4. 读 `llm/prompt.rs` 理解润色 Prompt 如何被 app 类别 + 词典 + 场景注入。
5. 用 `npm run tauri dev` 跑起来，配一个 BYOK provider 实测一次完整听写。
6. 按 `docs/` 中相关 spec 深入特定功能（Ask、Windows 输出、本地场景等）。

---

## 8. 附：关键文件速查表

| 关注点 | 文件 |
|---|---|
| 前端入口 / 三窗口分流 | `src/App.tsx:227` |
| 前端核心状态 | `src/stores/appStore.ts` |
| 前端路由 | `src/lib/router.ts` |
| STT 抽象 + 工厂 | `src-tauri/src/stt/mod.rs` |
| 各 STT provider | `src-tauri/src/stt/{deepgram,assemblyai,cloud,whisper_compat,aliyun_qwen3_asr,apple_speech,volcengine}.rs` |
| LLM 抽象 + 默认配置 | `src-tauri/src/llm/mod.rs` |
| LLM Prompt 构造 | `src-tauri/src/llm/prompt.rs` |
| 管线状态机 | `src-tauri/src/pipeline.rs` |
| 应用检测 | `src-tauri/src/app_detector/` |
| 语音意图 | `src-tauri/src/voice_intent/` |
| 输出（键盘/剪贴板） | `src-tauri/src/output/` |
| 存储（SQLite） | `src-tauri/src/storage/mod.rs` |
| 凭证（keyring） | `src-tauri/src/credentials.rs` |
| 命令注册 | `src-tauri/src/lib.rs:1151` |
| Tauri 配置 | `src-tauri/tauri.conf.json` |
| 构建依赖 | `src-tauri/Cargo.toml` / `package.json` |
| 设计规格集合 | `docs/*.md` |

---

*本文档由文档专家基于全仓库阅读（源码 + 配置 + 既有 spec）撰写，宏观定位与微观实现均有代码引用佐证。*
