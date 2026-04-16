# TTS Integration

## Overview

TTS（Text-to-Speech）模块负责将 LLM 生成的文本实时转换为语音。数字人系统采用**第三方 TTS 服务**实现流式语音合成。

---

## Directory Structure

```
server/
├── tts/
│   ├── tts.go           # TTS 接口定义
│   ├── mock.go          # Mock TTS 实现
│   └── real.go          # 真实 TTS 实现（第三方服务）
```

---

## Interface Definition

```go
type TTS interface {
    // Synthesize 处理文本流，返回音频流
    Synthesize(ctx context.Context, textIn <-chan TextChunk) <-chan AudioChunk

    // Close 关闭 TTS
    Close()
}
```

---

## Data Structures

### TextChunk

```go
type TextChunk struct {
    SessionID string
    TraceID   string
    Seq       int
    Text      string
    Final     bool        // 是否为最终文本
}
```

### AudioChunk

```go
type AudioChunk struct {
    SessionID string
    TraceID   string
    Seq       int
    Data      []byte      // 音频数据 (PCM/WAV)
    SampleRate int        // 采样率 (16000)
    End       bool        // 是否为结束标记
}
```

---

## Supported Third-party Services

### 1. 阿里云语音合成

- **API**: DashScope TTS API
- **Voice**: zhitianxiaoyao, zhizhang
- **Streaming**: 支持 SSE

### 2. 百度语音合成

- **API**: Baidu TTS API
- **Voice**: 多种中文音色
- **Streaming**: 支持流式合成

### 3. 讯飞语音合成

- **API**: iFlytek TTS API
- **Voice**: 多种中文音色
- **Streaming**: 支持流式合成

### 4. Google Cloud Text-to-Speech

- **API**: Google Cloud TTS API
- **Voice**: 多种语言和音色
- **Streaming**: 支持 SSE

### 5. Azure Speech Services

- **API**: Azure Speech SDK
- **Voice**: 多种语言和音色
- **Streaming**: 支持流式合成

---

## Mock Implementation

用于开发和测试：

```go
type MockTTS struct{}

func (m *MockTTS) Synthesize(ctx context.Context, textIn <-chan TextChunk) <-chan AudioChunk {
    out := make(chan AudioChunk, 50)
    go func() {
        defer close(out)
        seq := 0
        for {
            select {
            case <-ctx.Done():
                return
            case text, ok := <-textIn:
                if !ok {
                    // 发送结束标记
                    out <- AudioChunk{
                        Seq: seq,
                        End: true,
                    }
                    return
                }

                // 模拟 TTS 处理
                time.Sleep(50 * time.Millisecond)

                // 生成 mock 音频数据
                audioData := generateMockAudio(len(text.Text))

                out <- AudioChunk{
                    SessionID: text.SessionID,
                    TraceID:   text.TraceID,
                    Seq:       seq,
                    Data:      audioData,
                    SampleRate: 16000,
                    End:       false,
                }
                seq++
            }
        }
    }()
    return out
}

func (m *MockTTS) Close() {}

func generateMockAudio(textLen int) []byte {
    // 生成指定长度的 mock PCM 数据
    data := make([]byte, textLen*160) // 假设每个字符 160 字节
    for i := range data {
        data[i] = byte(i % 256)
    }
    return data
}
```

---

## Real Implementation (阿里云 DashScope Example)

```go
type DashScopeTTS struct {
    apiKey   string
    voice    string
}

func NewDashScopeTTS(apiKey, voice string) *DashScopeTTS {
    return &DashScopeTTS{
        apiKey: apiKey,
        voice:  voice,
    }
}

func (m *DashScopeTTS) Synthesize(ctx context.Context, textIn <-chan TextChunk) <-chan AudioChunk {
    out := make(chan AudioChunk, 50)

    // 创建 SSE 连接
    req, err := m.createSSERequest()
    if err != nil {
        log.Fatalf("Failed to create request: %v", err)
    }

    // 收集完整文本
    go func() {
        var fullText strings.Builder
        for {
            select {
            case <-ctx.Done():
                return
            case text, ok := <-textIn:
                if !ok {
                    // 发送完整文本进行合成
                    m.synthesizeText(req, fullText.String())
                    return
                }
                fullText.WriteString(text.Text)
            }
        }
    }()

    // 接收音频流
    go func() {
        defer close(out)
        seq := 0
        for audio := range m.receiveAudio(req) {
            out <- AudioChunk{
                SessionID:  audio.SessionID,
                TraceID:    audio.TraceID,
                Seq:        seq,
                Data:       audio.Data,
                SampleRate: 16000,
                End:        audio.End,
            }
            seq++
        }
    }()

    return out
}

func (m *DashScopeTTS) Close() {}
```

---

## Real Implementation (百度 TTS Example)

```go
type BaiduTTS struct {
    appID     string
    apiKey    string
    secretKey string
    voice     string
}

func NewBaiduTTS(appID, apiKey, secretKey string) *BaiduTTS {
    return &BaiduTTS{
        appID:     appID,
        apiKey:    apiKey,
        secretKey: secretKey,
        voice:     "0", // 默认音色
    }
}

func (m *BaiduTTS) Synthesize(ctx context.Context, textIn <-chan TextChunk) <-chan AudioChunk {
    out := make(chan AudioChunk, 50)

    // 获取 access token
    token, err := m.getAccessToken()
    if err != nil {
        log.Fatalf("Failed to get access token: %v", err)
    }

    // 收集完整文本
    go func() {
        var fullText strings.Builder
        var sessionID, traceID string
        for {
            select {
            case <-ctx.Done():
                return
            case text, ok := <-textIn:
                if !ok {
                    // 发送完整文本进行合成
                    audioData, err := m.synthesizeText(token, fullText.String())
                    if err != nil {
                        log.Printf("Failed to synthesize: %v", err)
                        return
                    }
                    out <- AudioChunk{
                        SessionID:  sessionID,
                        TraceID:    traceID,
                        Seq:        0,
                        Data:       audioData,
                        SampleRate: 16000,
                        End:        true,
                    }
                    close(out)
                    return
                }
                fullText.WriteString(text.Text)
                sessionID = text.SessionID
                traceID = text.TraceID
            }
        }
    }()

    return out
}

func (m *BaiduTTS) Close() {}
```

---

## Integration with Pipeline

TTS 模块是 Pipeline 的最后一个 Stage：

```go
type Pipeline struct {
    asr    ASR
    agent  Agent
    tts    TTS
}

func (p *Pipeline) Run(ctx context.Context, audioIn <-chan AudioChunk) <-chan AudioChunk {
    // ASR Stage
    textOut := p.asr.Stream(ctx, audioIn)

    // Agent Stage
    agentOut := p.agent.Process(ctx, textOut)

    // TTS Stage
    ttsOut := p.tts.Synthesize(ctx, agentOut)

    return ttsOut
}
```

---

## LLM → TTS Parallel Processing

为了实现更低延迟，TTS 可以在 LLM 输出的第一个 token 时就开始合成：

```go
type ParallelTTS struct {
    llm    LLM
    tts    TTS
}

func (p *ParallelTTS) Process(ctx context.Context, textIn <-chan TextChunk) <-chan AudioChunk {
    out := make(chan AudioChunk, 50)

    // 创建 LLM 输出 channel
    llmOut := make(chan TextChunk, 20)

    // 启动 LLM 处理
    go func() {
        defer close(llmOut)
        p.llm.Process(ctx, textIn, llmOut)
    }()

    // 启动 TTS 处理（尽早开始）
    go func() {
        defer close(out)
        p.tts.Synthesize(ctx, llmOut, out)
    }()

    return out
}
```

---

## Backpressure Handling

TTS 模块需要处理下游客户端的背压：

```go
func (m *MockTTS) Synthesize(ctx context.Context, textIn <-chan TextChunk) <-chan AudioChunk {
    out := make(chan AudioChunk, 50)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case text, ok := <-textIn:
                if !ok {
                    return
                }

                // 生成音频
                audio := m.synthesize(text)

                // 使用 non-blocking 发送
                select {
                case out <- audio:
                default:
                    // 客户端繁忙，丢弃音频块
                    // 记录丢弃的块
                    log.Printf("Dropping audio chunk due to backpressure")
                }
            }
        }
    }()
    return out
}
```

---

## Emotion Support

TTS 需要支持 emotion 参数以生成不同情感的语音：

```go
type TextChunkWithEmotion struct {
    TextChunk
    Emotion string  // "neutral", "happy", "sad", "angry"
}

func (m *RealTTS) SynthesizeWithEmotion(ctx context.Context, textIn <-chan TextChunkWithEmotion) <-chan AudioChunk {
    // 根据 emotion 选择不同的语音配置
    voiceConfigs := map[string]string{
        "neutral": "zhitianxiaoyao",
        "happy":   "zhitianxiaoyao",
        "sad":     "zhizhang",
        "angry":   "zhizhang",
    }

    // 实现...
}
```

---

## Configuration

通过配置文件管理第三方服务：

```yaml
tts:
  provider: "dashscope"  # dashscope | baidu | iflytek | google | azure
  voice: "zhitianxiaoyao"
  api_key: "${TTS_API_KEY}"

  # 提供商特定配置
  dashscope:
    base_url: "https://dashscope.aliyuncs.com/api/v1"
    format: "audio-16khz-32kbitrate-mono-mp3"

  baidu:
    app_id: "${BAIDU_TTS_APP_ID}"
    api_key: "${BAIDU_TTS_API_KEY}"
    secret_key: "${BAIDU_TTS_SECRET_KEY}"
    per: 0  # 音色参数

  iflytek:
    app_id: "${IFLYTEK_TTS_APP_ID}"
    api_key: "${IFLYTEK_TTS_API_KEY}"
    api_secret: "${IFLYTEK_TTS_API_SECRET}"

  google:
    credentials_file: "/path/to/credentials.json"
    language_code: "zh-CN"
    ssml_gender: "FEMALE"

  azure:
    region: "eastus"
    key: "${AZURE_SPEECH_KEY}"
    voice_name: "zh-CN-XiaoxiaoNeural"
```

---

## Implementation Checklist

- [ ] TTS Interface 定义
- [ ] Mock TTS 实现
- [ ] 阿里云 DashScope TTS 集成
- [ ] 百度 TTS 集成
- [ ] 讯飞 TTS 集成（可选）
- [ ] Google Cloud TTS 集成（可选）
- [ ] Azure Speech 集成（可选）
- [ ] 流式处理支持
- [ ] Backpressure 控制
- [ ] Emotion 支持
- [ ] Context 取消支持
- [ ] 错误处理和重试
- [ ] 性能监控