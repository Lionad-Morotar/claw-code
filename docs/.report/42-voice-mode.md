# Unit 42: Voice Mode — 语音输入功能源码分析

**原始文档**: https://ccb.agent-aura.top/docs/features/voice-mode  
**输出文件**: `docs/.report/42-voice-mode.md`  
**分析日期**: 2026-04-09

---

## 一、功能概述

VOICE_MODE 实现"按键说话"（Push-to-Talk）语音输入功能。用户按住空格键录音，音频通过 WebSocket 流式传输到 Anthropic STT 端点（Nova 3），实时转录显示在终端中。

**Feature Flag**: `VOICE_MODE`（dev/build 默认启用）  
**实现状态**: 完整可用（需要 Anthropic OAuth）  
**引用数**: 46 处

### 核心特性

- **Push-to-Talk**: 长按空格键录音，释放后自动发送
- **流式转录**: 录音过程中实时显示中间转录结果
- **无缝集成**: 转录文本直接作为用户消息提交到对话

---

## 二、用户交互

### 操作行为

| 操作 | 行为 |
|------|------|
| 长按空格 | 开始录音，显示录音状态 |
| 释放空格 | 停止录音，等待最终转录 |
| 转录完成 | 自动插入到输入框并提交 |
| `/voice` 命令 | 切换语音模式开关 |

### UI 反馈

- **录音指示器**: 录音时显示红色/脉冲动画
- **中间转录**: 录音过程中显示 STT 实时识别文本
- **最终转录**: 完成后替换中间结果

---

## 三、实现架构

### 3.1 门控逻辑

**文件**: [`src/voice/voiceModeEnabled.ts`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/voice/voiceModeEnabled.ts)

三层检查机制：

```typescript
isVoiceModeEnabled() = hasVoiceAuth() && isVoiceGrowthBookEnabled()
```

1. **Feature Flag**: `feature('VOICE_MODE')` — 编译时/运行时开关
2. **GrowthBook Kill-Switch**: `tengu_amber_quartz_disabled` — 紧急关闭开关（默认 false = 未禁用）
3. **Auth 检查**: `hasVoiceAuth()` — 需要 Anthropic OAuth token（非 API key）

源码位置：`packages/ccb/src/voice/voiceModeEnabled.ts#L16-L23`

```typescript
export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}
```

源码位置：`packages/ccb/src/voice/voiceModeEnabled.ts#L32-L44`

```typescript
export function hasVoiceAuth(): boolean {
  // Voice mode requires Anthropic OAuth — it uses the voice_stream
  // endpoint on claude.ai which is not available with API keys,
  // Bedrock, Vertex, or Foundry.
  if (!isAnthropicAuthEnabled()) {
    return false
  }
  const tokens = getClaudeAIOAuthTokens()
  return Boolean(tokens?.accessToken)
}
```

### 3.2 核心模块

| 模块 | 职责 | 行数 |
|------|------|------|
| [`src/voice/voiceModeEnabled.ts`](packages/ccb/src/voice/voiceModeEnabled.ts) | Feature flag + GrowthBook + Auth 三层门控 | L1-L55 |
| [`src/hooks/useVoice.ts`](packages/ccb/src/hooks/useVoice.ts) | React hook 管理录音状态和 WebSocket 连接 | L1-L1144 |
| [`src/services/voiceStreamSTT.ts`](packages/ccb/src/services/voiceStreamSTT.ts) | WebSocket 流式传输到 Anthropic STT | L1-L545 |
| [`src/services/voice.ts`](packages/ccb/src/services/voice.ts) | 本地音频录制（原生/SoX/arecord） | L1-L525 |
| [`src/hooks/useVoiceIntegration.tsx`](packages/ccb/src/hooks/useVoiceIntegration.tsx) | 键盘事件处理与输入框集成 | L1-L722 |
| [`src/commands/voice/voice.ts`](packages/ccb/src/commands/voice/voice.ts) | `/voice` 命令实现 | L1-L151 |

### 3.3 数据流

```
用户按下空格键
    ▼
useVoice hook 激活
    ▼
macOS 原生音频 / SoX 开始录音
    ▼
WebSocket 连接到 Anthropic STT 端点
    ├──→ 中间转录结果 → 实时显示
    ▼
用户释放空格键
    ▼
停止录音，等待最终转录
    ▼
转录文本 → 插入输入框 → 自动提交
```

### 3.4 音频录制

支持两种音频后端：

1. **原生音频模块**（优先）: `audio-capture-napi` — macOS CoreAudio/AudioUnit，跨平台（macOS/Linux/Windows）
2. **SoX（Sound eXchange）**: 回退方案，Linux/macOS
3. **arecord（ALSA）**: Linux 专用回退

源码位置：`packages/ccb/src/services/voice.ts#L183-L212`

```typescript
export async function startRecording(
  onData: (chunk: Buffer) => void,
  onEnd: () => void,
  options?: { silenceDetection?: boolean },
): Promise<boolean> {
  // Try native audio module first (macOS, Linux, Windows via cpal)
  const napi = await loadAudioNapi()
  const nativeAvailable = napi.isNativeAudioAvailable()
  if (nativeAvailable) {
    const started = napi.startNativeRecording(...)
    if (started) return true
  }
  // Fallback: arecord on Linux
  if (process.platform === 'linux' && hasCommand('arecord')) {
    return startArecordRecording(onData, onEnd)
  }
  // Fallback: SoX rec
  return startSoxRecording(onData, onEnd, options)
}
```

音频流通过 WebSocket 发送到 Anthropic 的 Nova 3 STT 模型。

---

## 四、关键设计决策

### 4.1 OAuth 独占

语音模式使用 `voice_stream` 端点（claude.ai），仅 Anthropic OAuth 用户可用。API key、Bedrock、Vertex 用户无法使用。

源码位置：`packages/ccb/src/services/voiceStreamSTT.ts#L124-L131`

```typescript
// voice_stream is a private_api route, but /api/ws/ is also exposed on
// the api.anthropic.com listener. We target that host instead of claude.ai
// because the claude.ai CF zone uses TLS fingerprinting and challenges
// non-browser clients. Same private-api pod, same OAuth Bearer auth.
```

### 4.2 GrowthBook 负向门控

`tengu_amber_quartz_disabled` 默认 `false`，新安装自动可用（无需等 GrowthBook 初始化）。

源码位置：`packages/ccb/src/voice/voiceModeEnabled.ts#L8-L15`

```typescript
/**
 * Kill-switch check for voice mode. Returns true unless the
 * `tengu_amber_quartz_disabled` GrowthBook flag is flipped on (emergency
 * off). Default `false` means a missing/stale disk cache reads as "not
 * killed" — so fresh installs get voice working immediately without
 * waiting for GrowthBook init.
 */
```

### 4.3 Keychain 缓存

`getClaudeAIOAuthTokens()` 首次调用访问 macOS keychain（~20-50ms），后续缓存命中。

源码位置：`packages/ccb/src/hooks/useVoiceEnabled.ts#L8-L18`

```typescript
/**
 * Combines user intent (settings.voiceEnabled) with auth + GB kill-switch.
 * Only the auth half is memoized on authVersion — it's the expensive one
 * (cold getClaudeAIOAuthTokens memoize → sync `security` spawn, ~60ms/call).
 */
```

### 4.4 独立于主 feature flag

`isVoiceGrowthBookEnabled()` 在 feature flag 关闭时短路返回 `false`，不触发任何模块加载。

---

## 五、WebSocket 流式传输

### 5.1 连接建立

源码位置：`packages/ccb/src/services/voiceStreamSTT.ts#L111-L195`

```typescript
export async function connectVoiceStream(
  callbacks: VoiceStreamCallbacks,
  options?: { language?: string; keyterms?: string[] },
): Promise<VoiceStreamConnection | null> {
  await checkAndRefreshOAuthTokenIfNeeded()
  
  const tokens = getClaudeAIOAuthTokens()
  if (!tokens?.accessToken) return null
  
  const wsBaseUrl = process.env.VOICE_STREAM_BASE_URL ||
    getOauthConfig().BASE_API_URL.replace('https://', 'wss://')
  
  const params = new URLSearchParams({
    encoding: 'linear16',
    sample_rate: '16000',
    channels: '1',
    endpointing_ms: '300',
    utterance_end_ms: '1000',
    language: options?.language ?? 'en',
  })
  
  // Nova 3 gate
  const isNova3 = getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_frost', false)
  if (isNova3) {
    params.set('use_conversation_engine', 'true')
    params.set('stt_provider', 'deepgram-nova3')
  }
  
  const ws = new WebSocket(url, {
    headers: {
      Authorization: `Bearer ${tokens.accessToken}`,
      'User-Agent': getUserAgent(),
      'x-app': 'cli',
    },
  })
}
```

### 5.2 消息协议

WebSocket 使用 JSON 控制消息和二进制音频帧：

- **发送**: `KeepAlive`、`CloseStream` + 二进制音频数据
- **接收**: `TranscriptText`、`TranscriptEndpoint`、`TranscriptError`

源码位置：`packages/ccb/src/services/voiceStreamSTT.ts#L74-L94`

```typescript
type VoiceStreamTranscriptText = {
  type: 'TranscriptText'
  data: string
}

type VoiceStreamTranscriptEndpoint = {
  type: 'TranscriptEndpoint'
}

type VoiceStreamTranscriptError = {
  type: 'TranscriptError'
  error_code?: string
  description?: string
}
```

### 5.3 Finalize 机制

`finalize()` 发送 `CloseStream` 并等待服务器确认，包含 3 个超时机制：

源码位置：`packages/ccb/src/services/voiceStreamSTT.ts#L44-L47`

```typescript
export const FINALIZE_TIMEOUTS_MS = {
  safety: 5_000,      // 安全超时
  noData: 1_500,      // 无数据超时
}
```

源码位置：`packages/ccb/src/services/voiceStreamSTT.ts#L239-L304`

```typescript
finalize(): Promise<FinalizeSource> {
  return new Promise<FinalizeSource>(resolve => {
    const safetyTimer = setTimeout(() => resolveFinalize?.('safety_timeout'), 5000)
    const noDataTimer = setTimeout(() => resolveFinalize?.('no_data_timeout'), 1500)
    
    resolveFinalize = (source: FinalizeSource) => {
      clearTimeout(safetyTimer)
      clearTimeout(noDataTimer)
      // Promote any unreported interim text to final
      if (lastTranscriptText) {
        callbacks.onTranscript(lastTranscriptText, true)
      }
      resolve(source)
    }
    
    setTimeout(() => {
      finalized = true
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(CLOSE_STREAM_MSG)
      }
    }, 0)
  })
}
```

### 5.4 静默丢弃重放

~1% 的会话会遇到 sticky-broken CE pod，接受音频但返回零转录。系统自动重放音频缓冲区。

源码位置：`packages/ccb/src/hooks/useVoice.ts#L385-L450`

```typescript
// Silent-drop replay: server accepted audio, mic captured signal,
// but finalize timed out with zero transcript — replay on fresh WS once.
if (
  finalizeSource === 'no_data_timeout' &&
  hadAudioSignal &&
  wsConnected &&
  !focusTriggered &&
  accumulatedRef.current.trim() === '' &&
  !silentDropRetriedRef.current
) {
  silentDropRetriedRef.current = true
  // Replay buffered audio on fresh connection
  await new Promise<void>(resolve => {
    connectVoiceStream({ onReady: conn => {
      conn.send(Buffer.concat(fullAudioRef.current))
      void conn.finalize().then(() => resolve())
    }}, { language: stt.code, keyterms })
  })
}
```

---

## 六、按键事件处理

### 6.1 握持检测逻辑

源码位置：`packages/ccb/src/hooks/useVoiceIntegration.tsx#L426-L500`

```typescript
const handleKeyDown = (e: KeyboardEvent): void => {
  // Match the configured key (default: space)
  if (bareChar !== null) {
    if (e.ctrl || e.meta || e.shift) return
    const normalized = bareChar === ' ' ? normalizeFullWidthSpace(e.key) : e.key
    if (normalized[0] !== bareChar) return
    repeatCount = normalized.length
  }
  
  // Guard: only swallow keypresses when recording triggered by key-hold
  if (isHoldActiveRef.current && currentVoiceState !== 'idle') {
    e.stopImmediatePropagation()
    stripTrailing(repeatCount, { char: bareChar, floor: recordingFloorRef.current })
    voiceHandleKeyEvent()
    return
  }
  
  // Activation: modifier combos on first press, bare chars after threshold
  if (bareChar === null || rapidCountRef.current >= HOLD_THRESHOLD) {
    e.stopImmediatePropagation()
    isHoldActiveRef.current = true
    stripTrailing(charsInInputRef.current + repeatCount, { char: bareChar, anchor: true })
    voiceHandleKeyEvent()
    return
  }
  
  // Warmup: first WARMUP_THRESHOLD chars flow to text input
  if (countBefore >= WARMUP_THRESHOLD) {
    e.stopImmediatePropagation()
    stripTrailing(repeatCount, { char: bareChar, floor: charsInInputRef.current })
  }
}
```

### 6.2 关键常量

源码位置：`packages/ccb/src/hooks/useVoiceIntegration.tsx#L40-L46`

```typescript
const RAPID_KEY_GAP_MS = 120       // 快速按键间隔
const HOLD_THRESHOLD = 5           // 握持激活阈值
const WARMUP_THRESHOLD = 2         // 暖场显示阈值
```

### 6.3 释放检测

源码位置：`packages/ccb/src/hooks/useVoice.ts#L1028-L1095`

```typescript
const handleKeyEvent = useCallback((fallbackMs = REPEAT_FALLBACK_MS): void => {
  if (currentState === 'idle') {
    // Start recording on first keypress
    void startRecordingSession()
    // Fallback: arm release timer if no auto-repeat arrives
    repeatFallbackTimerRef.current = setTimeout(() => {
      if (!seenRepeatRef.current) {
        seenRepeatRef.current = true
        releaseTimerRef.current = setTimeout(finishRecording, RELEASE_TIMEOUT_MS)
      }
    }, fallbackMs)
  } else if (currentState === 'recording') {
    // Second+ keypress — auto-repeat has started
    seenRepeatRef.current = true
    clearTimeout(repeatFallbackTimerRef.current)
  }
  
  // Reset release timer on every keypress
  clearTimeout(releaseTimerRef.current)
  
  // Arm release timer only after auto-repeat seen
  if (stateRef.current === 'recording' && seenRepeatRef.current) {
    releaseTimerRef.current = setTimeout(finishRecording, RELEASE_TIMEOUT_MS)
  }
}, [enabled, focusMode, cleanup])
```

源码位置：`packages/ccb/src/hooks/useVoice.ts#L140-L146`

```typescript
const RELEASE_TIMEOUT_MS = 200     // 按键释放检测超时
const REPEAT_FALLBACK_MS = 600    // OS 初始重复延迟 fallback
```

---

## 七、Focus 模式

Focus 模式下，录音由终端焦点驱动而非按键：

源码位置：`packages/ccb/src/hooks/useVoice.ts#L543-L604`

```typescript
// Focus-driven recording: start on focus, stop on blur
useEffect(() => {
  if (!enabled || !focusMode) return
  
  if (isFocused && stateRef.current === 'idle') {
    focusTriggeredRef.current = true
    void startRecordingSession()
    armFocusSilenceTimer()
  } else if (!isFocused) {
    silenceTimedOutRef.current = false
    if (stateRef.current === 'recording') {
      finishRecording()
    }
  }
}, [enabled, focusMode, isFocused])
```

静默超时自动结束：

源码位置：`packages/ccb/src/hooks/useVoice.ts#L517-L541`

```typescript
function armFocusSilenceTimer(): void {
  focusSilenceTimerRef.current = setTimeout(() => {
    if (stateRef.current === 'recording' && focusTriggeredRef.current) {
      silenceTimedOutRef.current = true
      finishRecording()
    }
  }, FOCUS_SILENCE_TIMEOUT_MS)
}
```

源码位置：`packages/ccb/src/hooks/useVoice.ts#L150`

```typescript
const FOCUS_SILENCE_TIMEOUT_MS = 5_000
```

---

## 八、语言支持

源码位置：`packages/ccb/src/hooks/useVoice.ts#L44-L117`

```typescript
const LANGUAGE_NAME_TO_CODE: Record<string, string> = {
  english: 'en',
  spanish: 'es',
  français: 'fr',
  日本語：'ja',
  deutsch: 'de',
  // ... 22 种语言
}

const SUPPORTED_LANGUAGE_CODES = new Set([
  'en', 'es', 'fr', 'ja', 'de', 'pt', 'it', 'ko',
  'hi', 'id', 'ru', 'pl', 'tr', 'nl', 'uk', 'el',
  'cs', 'da', 'sv', 'no',
])

export function normalizeLanguageForSTT(language: string | undefined): {
  code: string
  fellBackFrom?: string
} {
  if (!language) return { code: DEFAULT_STT_LANGUAGE }
  const lower = language.toLowerCase().trim()
  if (SUPPORTED_LANGUAGE_CODES.has(lower)) return { code: lower }
  const fromName = LANGUAGE_NAME_TO_CODE[lower]
  if (fromName) return { code: fromName }
  return { code: DEFAULT_STT_LANGUAGE, fellBackFrom: language }
}
```

---

## 九、/voice 命令

源码位置：`packages/ccb/src/commands/voice/voice.ts#L16-L150`

```typescript
export const call: LocalCommandCall = async () => {
  // Check auth and kill-switch
  if (!isVoiceModeEnabled()) {
    if (!isAnthropicAuthEnabled()) {
      return { type: 'text', value: 'Voice mode requires a Claude.ai account...' }
    }
    return { type: 'text', value: 'Voice mode is not available.' }
  }
  
  // Toggle OFF
  if (currentSettings.voiceEnabled === true) {
    updateSettingsForSource('userSettings', { voiceEnabled: false })
    return { type: 'text', value: 'Voice mode disabled.' }
  }
  
  // Toggle ON — pre-flight checks
  const recording = await checkRecordingAvailability()
  if (!recording.available) return { type: 'text', value: recording.reason }
  
  const deps = await checkVoiceDependencies()
  if (!deps.available) return { type: 'text', value: `No audio recording tool found...` }
  
  // Enable voice
  updateSettingsForSource('userSettings', { voiceEnabled: true })
  return {
    type: 'text',
    value: `Voice mode enabled. Hold ${key} to record.${langNote}`,
  }
}
```

---

## 十、UI 集成

### 10.1  interim 转录显示

源码位置：`packages/ccb/src/hooks/useVoiceIntegration.tsx#L268-L302`

```typescript
useEffect(() => {
  if (voicePrefixRef.current === null) return
  const prefix = voicePrefixRef.current
  const suffix = voiceSuffixRef.current
  const newValue = prefix + ' ' + voiceInterimTranscript + ' ' + suffix
  const cursorPos = prefix.length + 1 + voiceInterimTranscript.length
  insertTextRef.current.setInputWithCursor(newValue, cursorPos)
  lastSetInputRef.current = newValue
}, [voiceInterimTranscript, setInputValueRaw, inputValueRef, insertTextRef])
```

### 10.2 最终转录注入

源码位置：`packages/ccb/src/hooks/useVoiceIntegration.tsx#L304-L341`

```typescript
const handleVoiceTranscript = useCallback((text: string) => {
  const prefix = voicePrefixRef.current
  if (prefix === null) return
  const suffix = voiceSuffixRef.current
  const newInput = prefix + ' ' + text + ' ' + suffix
  const cursorPos = prefix.length + 1 + text.length
  insertTextRef.current.setInputWithCursor(newInput, cursorPos)
  lastSetInputRef.current = newInput
  voicePrefixRef.current = prefix + ' ' + text
}, [setInputValueRaw, inputValueRef, insertTextRef])
```

### 10.3 VoiceModeNotice

源码位置：`packages/ccb/src/components/LogoV2/VoiceModeNotice.tsx#L19-L51`

```typescript
function VoiceModeNoticeInner(): React.ReactNode {
  const [show] = useState(() =>
    isVoiceModeEnabled() &&
    getInitialSettings().voiceEnabled !== true &&
    (getGlobalConfig().voiceNoticeSeenCount ?? 0) < MAX_SHOW_COUNT
  )
  
  if (!show) return null
  
  return (
    <Box paddingLeft={2}>
      <AnimatedAsterisk />
      <Text dimColor> Voice mode is now available · /voice to enable</Text>
    </Box>
  )
}
```

---

## 十一、外部依赖

| 依赖 | 说明 |
|------|------|
| Anthropic OAuth | claude.ai 订阅登录，非 API key |
| GrowthBook | `tengu_amber_quartz_disabled` 紧急关闭 |
| audio-capture-napi | 原生音频录制模块（cpal） |
| SoX 或 arecord | Linux 回退方案 |
| Nova 3 STT | 语音转文本模型 |

---

## 十二、文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/ccb/src/voice/voiceModeEnabled.ts` | L1-L55 | 三层门控逻辑 |
| `packages/ccb/src/hooks/useVoice.ts` | L1-L1144 | React hook（录音状态 + WebSocket） |
| `packages/ccb/src/hooks/useVoiceEnabled.ts` | L1-L26 | 启用状态 memoized hook |
| `packages/ccb/src/hooks/useVoiceIntegration.tsx` | L1-L722 | 键盘事件与输入框集成 |
| `packages/ccb/src/services/voiceStreamSTT.ts` | L1-L545 | STT WebSocket 流式传输 |
| `packages/ccb/src/services/voice.ts` | L1-L525 | 本地音频录制 |
| `packages/ccb/src/context/voice.tsx` | L1-L77 | Voice 状态 Context |
| `packages/ccb/src/commands/voice/voice.ts` | L1-L151 | `/voice` 命令实现 |
| `packages/ccb/src/commands/voice/index.ts` | L1-L21 | 命令注册 |
| `packages/ccb/src/components/LogoV2/VoiceModeNotice.tsx` | L1-L52 | UI 通知组件 |

---

## 十三、关键类型定义

源码位置：`packages/ccb/src/context/voice.tsx#L9-L15`

```typescript
type VoiceState = {
  voiceState: 'idle' | 'recording' | 'processing'
  voiceError: string | null
  voiceInterimTranscript: string
  voiceAudioLevels: number[]  // 16 个音频电平条
  voiceWarmingUp: boolean
}
```

源码位置：`packages/ccb/src/services/voiceStreamSTT.ts#L51-L72`

```typescript
type VoiceStreamCallbacks = {
  onTranscript: (text: string, isFinal: boolean) => void
  onError: (error: string, opts?: { fatal?: boolean }) => void
  onClose: () => void
  onReady: (connection: VoiceStreamConnection) => void
}

type VoiceStreamConnection = {
  send: (audioChunk: Buffer) => void
  finalize: () => Promise<FinalizeSource>
  close: () => void
  isConnected: () => boolean
}

type FinalizeSource =
  | 'post_closestream_endpoint'
  | 'no_data_timeout'
  | 'safety_timeout'
  | 'ws_close'
  | 'ws_already_closed'
```

---

## 十四、Analytics 事件

源码位置：`packages/ccb/src/hooks/useVoice.ts`

```typescript
logEvent('tengu_voice_recording_started', {...})      // L717
logEvent('tengu_voice_recording_completed', {...})    // L478
logEvent('tengu_voice_silent_drop_replay', {...})     // L395
logEvent('tengu_voice_stream_early_retry', {...})     // L873
logEvent('tengu_voice_toggled', {enabled: true/false}) // L124
```

---

**报告完成时间**: 2026-04-09  
**源码版本**: claw-code @ commit 586f56d
