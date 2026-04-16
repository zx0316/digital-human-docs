# ASR Integration

## Overview

ASR（Automatic Speech Recognition）模块负责将客户端发送的音频流实时转换为文本。数字人系统采用**第三方 ASR 服务**实现流式语音识别。

---

## Directory Structure

```
server/
├── asr/
│   ├── asr.go           # ASR 接口定义
│   ├── mock.go          # Mock ASR 实现
│   └── real.go          # 真实 ASR 实现（第三方服务）
```

---

## Interface Definition

```go
type ASR interface {
    // Stream 处理音频流，返回识别结果
    Stream(ctx context.Context, audioCh <-chan AudioChunk) <-chan TextChunk

    // Close 关闭 ASR
    Close()
}
```

---

## Data Structures

### AudioChunk

```go
type AudioChunk struct {
    SessionID string
    TraceID   string
    Seq       int
    Data      []byte      // PCM 数据
    SampleRate int        // 采样率 (16000)
    Timestamp  int64
}
```

### TextChunk

```go
type TextChunk struct {
    SessionID string
    TraceID   string
    Seq       int
    Text      string
    Final     bool        // 是否为最终结果
}
```

---

## Supported Third-party Services

### 1. 阿里云语音识别

- **API**: DashScope ASR API
- **Model**: paraformer-zh
- **Streaming**: 支持实时音频流

### 2. 百度语音识别

- **API**: Baidu ASR API
- **Model**: ASR 模型
- **Streaming**: 支持实时音频流

### 3. 讯飞语音识别

- **API**: iFlytek ASR API
- **Streaming**: 支持实时音频流

### 4. Google Cloud Speech-to-Text

- **API**: Google Cloud Speech API
- **Streaming**: 支持 StreamingRecognize

### 5. Azure Speech Services

- **API**: Azure Speech SDK
- **Streaming**: 支持 Pull/Push Audio

---

## Mock Implementation

用于开发和测试：

```go
type MockASR struct{}

func (m *MockASR) Stream(ctx context.Context, audioCh <-chan AudioChunk) <-chan TextChunk {
    out := make(chan TextChunk, 20)
    go func() {
        defer close(out)
        seq := 0
        for {
            select {
            case <-ctx.Done():
                return
            case audio, ok := <-audioCh:
                if !ok {
                    return
                }
                seq++
                // 模拟识别延迟
                time.Sleep(100 * time.Millisecond)

                // 返回 partial 结果
                out <- TextChunk{
                    SessionID: audio.SessionID,
                    TraceID:   audio.TraceID,
                    Seq:       seq,
                    Text:      fmt.Sprintf("mock text %d", seq),
                    Final:     false,
                }

                // 模拟最终结果
                if seq%10 == 0 {
                    out <- TextChunk{
                        SessionID: audio.SessionID,
                        TraceID:   audio.TraceID,
                        Seq:       seq,
                        Text:      fmt.Sprintf("final text %d", seq),
                        Final:     true,
                    }
                }
            }
        }
    }()
    return out
}

func (m *MockASR) Close() {}
```

---

## Real Implementation (阿里云 DashScope Example)

```go
type DashScopeASR struct {
    apiKey   string
    model    string
}

func NewDashScopeASR(apiKey string) *DashScopeASR {
    return &DashScopeASR{
        apiKey: apiKey,
        model:  "paraformer-zh",
    }
}

func (m *DashScopeASR) Stream(ctx context.Context, audioCh <-chan AudioChunk) <-chan TextChunk {
    out := make(chan TextChunk, 20)

    // 创建 WebSocket 连接
    ws, err := m.createWebSocket()
    if err != nil {
        log.Fatalf("Failed to create WebSocket: %v", err)
    }

    // 发送音频流
    go func() {
        for {
            select {
            case <-ctx.Done():
                ws.Close()
                return
            case audio, ok := <-audioCh:
                if !ok {
                    ws.Close()
                    return
                }
                // 发送音频数据
                m.sendAudio(ws, audio)
            }
        }
    }()

    // 接收识别结果
    go func() {
        defer close(out)
        seq := 0
        for {
            resp, err := m.receive(ws)
            if err != nil {
                log.Printf("Failed to receive: %v", err)
                return
            }

            seq++
            out <- TextChunk{
                SessionID: resp.SessionID,
                TraceID:   resp.TraceID,
                Seq:       seq,
                Text:      resp.Text,
                Final:     resp.Final,
            }
        }
    }()

    return out
}

func (m *DashScopeASR) Close() {}
```

---

## Real Implementation (百度 ASR Example)

```go
type BaiduASR struct {
    appID     string
    apiKey    string
    secretKey string
}

func NewBaiduASR(appID, apiKey, secretKey string) *BaiduASR {
    return &BaiduASR{
        appID:     appID,
        apiKey:    apiKey,
        secretKey: secretKey,
    }
}

func (m *BaiduASR) Stream(ctx context.Context, audioCh <-chan AudioChunk) <-chan TextChunk {
    out := make(chan TextChunk, 20)

    // 获取 access token
    token, err := m.getAccessToken()
    if err != nil {
        log.Fatalf("Failed to get access token: %v", err)
    }

    // 创建 WebSocket 连接
    ws, err := m.createWebSocket(token)
    if err != nil {
        log.Fatalf("Failed to create WebSocket: %v", err)
    }

    // 发送音频流
    go func() {
        for {
            select {
            case <-ctx.Done():
                ws.Close()
                return
            case audio, ok := <-audioCh:
                if !ok {
                    ws.Close()
                    return
                }
                // 发送音频数据
                m.sendAudio(ws, audio)
            }
        }
    }()

    // 接收识别结果
    go func() {
        defer close(out)
        seq := 0
        for {
            resp, err := m.receive(ws)
            if err != nil {
                log.Printf("Failed to receive: %v", err)
                return
            }

            seq++
            out <- TextChunk{
                SessionID: resp.SessionID,
                TraceID:   resp.TraceID,
                Seq:       seq,
                Text:      resp.Text,
                Final:     resp.Final,
            }
        }
    }()

    return out
}

func (m *BaiduASR) Close() {}
```

---

## Integration with Pipeline

ASR 模块是 Pipeline 的第一个 Stage：

```go
type Pipeline struct {
    asr    ASR
    agent  Agent
    tts    TTS
}

func (p *Pipeline) Run(ctx context.Context, audioIn <-chan AudioChunk) <-chan TTSChunk {
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

## Backpressure Handling

ASR 模块需要处理上游音频流的背压：

```go
func (m *MockASR) Stream(ctx context.Context, audioCh <-chan AudioChunk) <-chan TextChunk {
    out := make(chan TextChunk, 20)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case audio, ok := <-audioCh:
                if !ok {
                    return
                }
                // 处理音频
                // 使用 non-blocking 发送到下游
                select {
                case out <- m.recognize(audio):
                default:
                    // 下游繁忙，丢弃最旧的 partial 结果
                    // 继续处理新的音频
                }
            }
        }
    }()
    return out
}
```

---

## Configuration

通过配置文件管理第三方服务：

```yaml
asr:
  provider: "dashscope"  # dashscope | baidu | iflytek | google | azure
  model: "paraformer-zh"
  api_key: "${ASR_API_KEY}"

  # 提供商特定配置
  dashscope:
    base_url: "wss://dashscope.aliyuncs.com/api/v1"

  baidu:
    app_id: "${BAIDU_ASR_APP_ID}"
    api_key: "${BAIDU_ASR_API_KEY}"
    secret_key: "${BAIDU_ASR_SECRET_KEY}"

  iflytek:
    app_id: "${IFLYTEK_ASR_APP_ID}"
    api_key: "${IFLYTEK_ASR_API_KEY}"
    api_secret: "${IFLYTEK_ASR_API_SECRET}"

  google:
    credentials_file: "/path/to/credentials.json"

  azure:
    region: "eastus"
    key: "${AZURE_SPEECH_KEY}"
```

---

## Implementation Checklist

- [ ] ASR Interface 定义
- [ ] Mock ASR 实现
- [ ] 阿里云 DashScope ASR 集成
- [ ] 百度 ASR 集成
- [ ] 讯飞 ASR 集成（可选）
- [ ] Google Cloud Speech 集成（可选）
- [ ] Azure Speech 集成（可选）
- [ ] 流式处理支持
- [ ] Backpressure 控制
- [ ] Context 取消支持
- [ ] 错误处理和重试
- [ ] 性能监控