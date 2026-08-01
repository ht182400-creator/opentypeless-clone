# Ask Anything 可靠性设计规范

日期：2026-06-29
状态：已实现；自动化发布检查于 2026-06-30 通过；发布后平台冒烟测试仍待执行
范围：OpenTypeless 桌面端 Ask Anything + talkmore 云端代理

## 摘要

Ask Anything 不能只修一个超时或弹窗问题。它是一条横跨桌面端、麦克风、STT、LLM、云端计费、结果窗口的链路。发布前必须保证每个失败点都有明确的用户反馈，并且失败时不会继续调用后续付费阶段。

当前根因判断：

1. 结果弹窗丢失是真问题。快捷键路径在 Rust 里打开 Ask 窗口后立即发 `ask:result`，如果前端监听还没挂上，事件会丢，用户会看到胶囊完成但没有结果弹窗内容。
2. 无声音长时间 thinking 是真问题。当前 Ask STT finalize 最长等 `120s`，没有语音或音频流无法结束时，用户会长时间卡在 thinking。
3. STT、LLM、云端 quota、网络错误必须原样返回给用户。不能把 STT quota/auth/service error 覆盖成 `No speech detected`。
4. 空语音必须在桌面端止住：没有有效 transcript 时不能调用 `/api/proxy/ask`，因此不能产生 Ask LLM 计费。
5. 云端已有部分计费保护：AppSumo cloud words 模式下，`/api/proxy/stt` 空文本会 release reservation，`/api/proxy/ask` 只有被调用并成功回答后才 settle。但桌面端仍必须阻止空语音进入 ask 阶段。

## 用户契约

用户触发 Ask Anything 后，只会看到这些状态：

1. Recording（录音中）：胶囊显示正在录音。
2. Thinking（思考中）：用户停止录音后，开始处理 STT 和 LLM。
3. Result popup（结果弹窗）：成功时弹窗只显示最终答案，不显示输入框、不显示发送按钮、不显示上下文。
4. Error popup（错误弹窗）：失败时弹窗只显示一条可理解错误，不显示输入框、不显示发送按钮。

失败不能静默。只要链路中任一阶段失败，就必须让用户看到错误，并且恢复 idle。

## 端到端链路

### 桌面端快捷键流程

1. 用户按 Ask 快捷键。
2. `src-tauri/src/hotkey.rs` 调用 `start_ask_dictation`。
3. `start_ask_dictation` 加载配置、校验 STT 鉴权/配置、连接 STT provider、开始音频采集，发出 `PipelineState::Recording`。
4. 用户再次按 Ask 快捷键。
5. `stop_ask_dictation` 停止音频采集，发出 `PipelineState::Polishing`，等待 STT finalize，校验 transcript，然后调用 LLM。
6. 成功后，Rust 返回 `AskDictationResult { question, answer }`。
7. 快捷键处理器打开/聚焦 `ask` 窗口并投递结果。
8. `src/components/AskPanel/AskPanel.tsx` 渲染仅含答案的弹窗内容。

### 云端 STT 流程

1. 桌面端云端 STT 使用会话 token 作为 bearer token。
2. 云端 STT 请求包含操作元数据：
   - `operationId`
   - `stageKey = <operationId>:stt`
   - `requestType`
   - `clientVersion`
3. talkmore `/api/proxy/stt` 鉴权、预留 quota、调用 Groq STT。
4. 如果 STT provider 失败，释放预留并返回错误。
5. 在 AppSumo cloud words 模式下，空 transcript 会释放预留。
6. 在旧的双计量模式下，STT quota 在成功的 STT 调用后调整为实际解析时长。

### 云端 Ask LLM 流程

1. 桌面端仅在存在非空且已校验的 transcript 时才调用 `/api/proxy/ask`。
2. 请求包含操作元数据：
   - `operationId`
   - `stageKey = <operationId>:ask`
   - `requestType = ask_anything`
   - `clientVersion`
3. talkmore 在调用 OpenRouter 前预留 ask 阶段 quota。
4. OpenRouter 失败时，释放预留。
5. 成功后，使用问题 + 答案的 cloud words 以及 LLM token 元数据结算 quota。

## 失败矩阵

| 阶段 | 失败 | 要求的用户反馈 | 是否进入下一阶段？ | 计费要求 |
| --- | --- | --- | --- | --- |
| Start | Ask 已在处理中 | 忽略重复的快捷键 | 否 | 无新计费 |
| Config | 配置加载失败 | `Could not load settings. Please retry.`（无法加载设置，请重试） | 否 | 不计费 |
| Auth | 选择了云端但无 session token | `Sign in to use Cloud Ask, or switch to BYOK.`（请登录以使用云端 Ask，或切换到 BYOK） | 否 | 不计费 |
| STT config | provider 需要 key 但 key 缺失 | `Configure speech recognition before using Ask.`（使用 Ask 前请先配置语音识别） | 否 | 不计费 |
| STT connect | provider 连接失败 | provider 错误文本（已脱敏） | 否 | 不计费 |
| Audio | 无输入设备 | `Microphone unavailable. Check your input device.`（麦克风不可用，请检查输入设备） | 否 | 不计费 |
| Audio | 权限被拒 | `Microphone permission is required.`（需要麦克风权限） | 否 | 不计费 |
| Recording | 发送音频失败 | STT 错误文本（已脱敏） | 否 | 释放任何预留 |
| STT | provider 返回 auth/quota/service 错误 | 精确映射的错误：auth、quota 或 service | 否 | 服务端错误时释放预留 |
| STT | 空 transcript | `No speech detected. Please try again.`（未检测到语音，请重试） | 否 | 无 Ask LLM 计费；AppSumo 路径下 cloud words STT 释放 |
| STT | 无 transcript 时 finalize 超时 | `No speech detected. Please try again.` | 否 | 无 Ask LLM 计费 |
| STT | 已有最终 transcript 时 finalize 超时 | 使用已收集的最终 transcript 继续 | 是 | 正常计费 |
| Transcript | 超过 500 字符 | `Question is too long.`（问题过长） | 否 | 无 Ask LLM 计费 |
| LLM config | 无 BYOK LLM 且无云端 token | `Sign in or configure an LLM provider.`（请登录或配置一个 LLM provider） | 否 | 无 LLM 计费 |
| LLM cloud | quota 超限 | `Cloud words used up. Please switch to BYOK mode or wait until reset.`（云端 words 已用完，请切换到 BYOK 模式或等待重置） | 否 | 释放失败的预留 |
| LLM cloud | OpenRouter/service 错误 | `Ask service error. Please try again.`（Ask 服务错误，请重试） | 否 | 释放失败的预留 |
| Popup | 原生结果事件被遗漏 | 前端在挂载时拉取一次 pending 结果 | 不适用 | 无额外计费 |
| Popup | 原生错误事件被遗漏 | 前端在挂载时拉取一次 pending 错误 | 不适用 | 无额外计费 |

## 错误处理要求

桌面端必须使用一种规范化的 Ask 消息形态：

```ts
type AskPopupMessage =
  | { kind: 'result'; payload: { question: string; answer: string } }
  | { kind: 'error'; payload: string }
```

规则：

1. 快捷键路径在展示 Ask 窗口前，把最新结果/错误存入 Rust。
2. Ask 窗口监听 `ask:result` 与 `ask:error`。
3. Ask 窗口在监听器就绪后，还调用一次 `take_pending_ask_message`。
4. `take_pending_ask_message` 精确消费一次消息。
5. 事件投递与 pending 消息投递都必须渲染完全一致的"仅答案/仅错误"弹窗 UI。
6. 任何错误发生时，桌面端发出 idle 状态并清除 busy/recording 状态。
7. STT 错误不能被后续的空 transcript 校验覆盖。
8. 空 transcript 绝不能调用 `answer_question`。
9. 展示给用户的错误字符串必须脱敏且有长度上限。

## 计费要求

Ask Anything 在云端模式下有两个计费阶段：

1. STT 阶段：语音转文本。
2. Ask 阶段：LLM 回答。

硬性规则：

1. 没有有效 transcript 意味着不发起 `/api/proxy/ask` 请求。
2. STT 开始/连接/音频失败不得调用 LLM。
3. STT provider 的 auth/quota/service 失败不得调用 LLM。
4. 空 transcript 不得调用 LLM。
5. `question` 校验必须在云端 ask 预留之前执行。
6. 云端 ask 输出上限固定为 `ASK_OUTPUT_TOKEN_LIMIT = 80`。
7. talkmore AppSumo cloud words 路径必须在 provider 失败时释放预留。
8. talkmore AppSumo cloud words 路径必须在空 transcript 时释放 STT 预留。
9. talkmore ask 路径必须在成功后才结算 LLM token 与 provider/model。
10. 重复弹窗重试不得重新调用 STT 或 LLM。

## 当前代码证据

桌面端：

1. `src-tauri/src/commands/ask.rs` 在 LLM 前校验空问题。
2. `src-tauri/src/commands/ask.rs` 在发布构建中等待 STT finalize 过久，需要更短的有界等待。
3. `src-tauri/src/hotkey.rs` 当前在显示 Ask 窗口后立即发出 result/error；这可能丢失事件。
4. `src/components/AskPanel/AskPanel.tsx` 把非嵌入弹窗渲染为"仅结果/仅错误"，但需要为遗漏的原生事件增加 pending 消息恢复。

云端：

1. `talkmore/src/app/api/proxy/stt/route.ts` 在付费 STT API 调用前拒绝缺失鉴权。
2. `talkmore/src/app/api/proxy/stt/route.ts` 当 STT 返回空文本时释放 AppSumo 云端预留。
3. `talkmore/src/app/api/proxy/ask/route.ts` 把输出上限设为 `80` token。
4. `talkmore/src/app/api/proxy/ask/route.ts` 在 OpenRouter 失败时释放预留。
5. `talkmore/src/lib/cloud-quota.ts` 在结算时记录云端 STT 计费秒数与 LLM token。

## 需要的实现任务

### 任务 1：结果/错误弹窗投递

文件：

1. `src-tauri/src/commands/ask.rs`
2. `src-tauri/src/hotkey.rs`
3. `src-tauri/src/lib.rs`
4. `src/lib/tauri.ts`
5. `src/components/AskPanel/AskPanel.tsx`
6. `src/components/AskPanel/__tests__/AskPanel.test.tsx`
7. `src/lib/__tests__/tauri-ask.test.ts`

验收：

1. 在 Rust 中新增 pending 一次性 Ask 结果/错误状态。
2. 注册 `take_pending_ask_message`。
3. 前端在监听器挂载后拉取 pending 消息。
4. 测试覆盖遗漏的原生结果事件。
5. 测试覆盖 Tauri 包装器调用。

命令：

```bash
npm test -- --run src/components/AskPanel/__tests__/AskPanel.test.tsx src/lib/__tests__/tauri-ask.test.ts
cargo test ask::tests --manifest-path src-tauri/Cargo.toml
```

### 任务 2：无语音与超时处理

文件：

1. `src-tauri/src/commands/ask.rs`
2. `src-tauri/src/audio/capture.rs`
3. `src-tauri/src/stt/*.rs`（经 provider 专项评审后按需）

验收：

1. 停止路径使用短暂有界的 STT finalize 等待。
2. 无 transcript 返回 `No speech detected. Please try again.`。
3. 空 transcript 绝不调用 LLM。
4. 现有 STT provider 错误优先于无语音。
5. 如果 finalize 超时前已存在最终 transcript，使用该 transcript 继续。
6. Windows 手动测试覆盖静音与真实语音。
7. macOS 手动测试覆盖静音与真实语音。

命令：

```bash
cargo test ask::tests --manifest-path src-tauri/Cargo.toml
npm test -- --run src/components/AskPanel/__tests__/AskPanel.test.tsx
```

### 任务 3：错误映射审计

文件：

1. `src-tauri/src/commands/ask.rs`
2. `src-tauri/src/stt/cloud.rs`
3. `src-tauri/src/stt/config.rs`
4. `src/lib/i18n` locale 文件（若新增 Ask 专属本地化错误）

验收：

1. 缺失云端 session 应有云端专属消息，而不是通用的 API key 文案。
2. 麦克风权限/设备错误变为用户可读的消息。
3. STT quota/auth/网络/service 错误保持可见，且不被无语音覆盖。
4. LLM quota/auth/网络/service 错误保持可见。
5. 错误足够短，能放进 Ask 弹窗。

### 任务 4：云端计费回归测试

文件：

1. `talkmore/src/app/api/proxy/stt/route.ts`
2. `talkmore/src/app/api/proxy/ask/route.ts`
3. 现有或新增的 talkmore API 测试

验收：

1. 空的 AppSumo STT 响应释放云端预留。
2. STT provider 错误释放云端预留。
3. Ask provider 错误释放云端预留。
4. 成功的 Ask 结算 cloud words 与 LLM token。
5. Ask 端点在预留前拒绝空问题。

## 手动发布门禁

发布前验证下面所有行：

| 平台 | 用例 | 预期结果 |
| --- | --- | --- |
| Windows | 按 Ask，什么都不说，停止 | 出现错误弹窗；不调用 LLM |
| Windows | 按 Ask，说一个问题，停止 | 出现仅含答案的弹窗 |
| Windows | 禁用/拒绝麦克风 | 出现错误弹窗 |
| Windows | 云端 quota 耗尽的测试账号 | 出现 quota 错误弹窗 |
| macOS | 按 Ask，什么都不说，停止 | 出现错误弹窗；不调用 LLM |
| macOS | 按 Ask，说一个问题，停止 | 出现仅含答案的弹窗 |
| macOS | 结果到达前 Ask 窗口未加载 | pending 结果仍会出现 |
| Linux | 按 Ask，说一个问题，停止 | 出现仅含答案的弹窗 |
| Cloud | 空 transcript 路径 | 释放 STT 预留；无 ask 阶段 |
| Cloud | STT 成功 + Ask 失败 | STT 已结算；ask 预留已释放 |

## 发布决策

该功能仅在以下情况下达到"可 CI 打包"的发布就绪：

1. pending 弹窗与无语音的单元测试通过。
2. 桌面端构建通过。
3. talkmore 构建通过。
4. 云端 quota 测试或直接部署日志确认没有空语音的 LLM 计费。
5. 不基于局部本地改动发布新版本。

发布后冒烟验证仍需覆盖：

1. Windows 手动 Ask 测试通过。
2. macOS 手动 Ask 测试通过。
3. 发布产物能在目标平台安装、签名并启动。
