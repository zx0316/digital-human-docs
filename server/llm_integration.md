# LLM Integration

## Overview

LLM（Large Language Model）模块负责理解用户输入、检索记忆、构建 prompt 并生成回复。数字人系统采用第三方 LLM 服务实现。

---

## Directory Structure

```
server/
├── llm/
│   ├── llm.go           # LLM 接口定义
│   ├── mock.go          # Mock LLM 实现
│   └── real.go          # 真实 LLM 实现（第三方服务）
```

---

## Interface Definition

```go
type LLM interface {
    // Process 处理文本流，返回 LLM 响应
    Process(ctx context.Context, textIn <-chan TextChunk) <-chan TextChunk

    // Close 关闭 LLM
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

### AgentResponse

```go
type AgentResponse struct {
    Text    string            // LLM 回复文本
    Emotion string            // 情感标签 (neutral|happy|sad|angry)
    Intent  string            // 意图类型 (chat|task|emotion)
}
```

---

## Supported Third-party Services

### 1. OpenAI GPT

- **API**: OpenAI Chat Completion API
- **Model**: gpt-4o, gpt-4-turbo, gpt-3.5-turbo
- **Streaming**: 支持 Server-Sent Events (SSE)

### 2. Claude (Anthropic)

- **API**: Anthropic Messages API
- **Model**: claude-3-5-sonnet, claude-3-opus
- **Streaming**: 支持 SSE

### 3. 阿里云通义千问

- **API**: DashScope API
- **Model**: qwen-turbo, qwen-plus, qwen-max
- **Streaming**: 支持 SSE

### 4. 百度文心一言

- **API**: ERNIE Bot API
- **Model**: ernie-4.0, ernie-3.5
- **Streaming**: 支持 SSE

---

## Mock Implementation

用于开发和测试：

```go
type MockLLM struct{}

func (m *MockLLM) Process(ctx context.Context, textIn <-chan TextChunk) <-chan TextChunk {
    out := make(chan TextChunk, 20)
    go func() {
        defer close(out)
        var textBuilder strings.Builder
        seq := 0

        for {
            select {
            case <-ctx.Done():
                return
            case text, ok := <-textIn:
                if !ok {
                    return
                }
                textBuilder.WriteString(text.Text)

                if text.Final {
                    // 模拟 LLM 处理延迟
                    time.Sleep(200 * time.Millisecond)

                    // 生成 mock 回复
                    mockResponse := fmt.Sprintf("Mock response to: %s", textBuilder.String())

                    // 流式发送回复
                    for i := 0; i < len(mockResponse); i += 10 {
                        seq++
                        end := i + 10
                        if end > len(mockResponse) {
                            end = len(mockResponse)
                        }

                        out <- TextChunk{
                            SessionID: text.SessionID,
                            TraceID:   text.TraceID,
                            Seq:       seq,
                            Text:      mockResponse[i:end],
                            Final:     end == len(mockResponse),
                        }
                    }
                }
            }
        }
    }()
    return out
}

func (m *MockLLM) Close() {}
```

---

## Real Implementation (OpenAI GPT Example)

```go
type OpenAILLM struct {
    client   *openai.Client
    model    string
    apiKey   string
}

func NewOpenAILLM(apiKey, model string) *OpenAILLM {
    return &OpenAILLM{
        client: openai.NewClient(apiKey),
        model:  model,
    }
}

func (m *OpenAILLM) Process(ctx context.Context, textIn <-chan TextChunk) <-chan TextChunk {
    out := make(chan TextChunk, 20)

    // 收集输入文本
    var userInput strings.Builder
    var sessionID, traceID string

    go func() {
        defer close(out)

        // 收集完整输入
        for {
            select {
            case <-ctx.Done():
                return
            case text, ok := <-textIn:
                if !ok {
                    goto process
                }
                userInput.WriteString(text.Text)
                sessionID = text.SessionID
                traceID = text.TraceID
            }
        }

    process:
        // 调用 OpenAI API
        req := openai.ChatCompletionRequest{
            Model: m.model,
            Messages: []openai.ChatCompletionMessage{
                {
                    Role:    openai.ChatMessageRoleUser,
                    Content: userInput.String(),
                },
            },
            Stream: true,
        }

        stream, err := m.client.CreateChatCompletionStream(ctx, req)
        if err != nil {
            log.Printf("Failed to create stream: %v", err)
            return
        }
        defer stream.Close()

        seq := 0
        for {
            select {
            case <-ctx.Done():
                return
            default:
                resp, err := stream.Recv()
                if err == io.EOF {
                    // 发送结束标记
                    out <- TextChunk{
                        SessionID: sessionID,
                        TraceID:   traceID,
                        Seq:       seq,
                        Text:      "",
                        Final:     true,
                    }
                    return
                }
                if err != nil {
                    log.Printf("Failed to receive: %v", err)
                    return
                }

                seq++
                content := ""
                if len(resp.Choices) > 0 {
                    content = resp.Choices[0].Delta.Content
                }

                out <- TextChunk{
                    SessionID: sessionID,
                    TraceID:   traceID,
                    Seq:       seq,
                    Text:      content,
                    Final:     false,
                }
            }
        }
    }()

    return out
}

func (m *OpenAILLM) Close() {}
```

---

## Integration with Agent

LLM 通常与 Agent 模块结合使用：

```go
type Agent struct {
    llm        LLM
    memory     Memory
    promptBuilder *PromptBuilder
}

func (a *Agent) Process(ctx context.Context, textIn <-chan TextChunk) <-chan TextChunk {
    out := make(chan TextChunk, 20)

    go func() {
        defer close(out)

        // 收集用户输入
        var userText strings.Builder
        var sessionID, traceID string
        for text := range textIn {
            userText.WriteString(text.Text)
            sessionID = text.SessionID
            traceID = text.TraceID
        }

        // 检索记忆
        memoryContext := a.memory.Retrieve(ctx, userText.String())

        // 构建 prompt
        prompt := a.promptBuilder.Build(sessionID, memoryContext, userText.String())

        // 调用 LLM
        llmOut := a.llm.Process(ctx, make(chan TextChunk))

        // 发送结果
        for chunk := range llmOut {
            chunk.SessionID = sessionID
            chunk.TraceID = traceID
            out <- chunk
        }
    }()

    return out
}
```

---

## Backpressure Handling

LLM 模块需要处理背压：

```go
func (m *OpenAILLM) Process(ctx context.Context, textIn <-chan TextChunk) <-chan TextChunk {
    out := make(chan TextChunk, 20)
    go func() {
        defer close(out)
        seq := 0
        for {
            select {
            case <-ctx.Done():
                return
            case chunk, ok := <-out:
                if !ok {
                    return
                }
                // 使用 non-blocking 发送
                select {
                case out <- chunk:
                default:
                    // 下游繁忙，丢弃
                    log.Printf("Dropping LLM chunk due to backpressure")
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
llm:
  provider: "openai"  # openai | claude | dashscope | ernie
  model: "gpt-4o"
  api_key: "${OPENAI_API_KEY}"

  # 提供商特定配置
  openai:
    base_url: "https://api.openai.com/v1"
    organization: ""

  claude:
    base_url: "https://api.anthropic.com"
    version: "2023-06-01"

  dashscope:
    base_url: "https://dashscope.aliyuncs.com/api/v1"
    api_key: "${DASHSCOPE_API_KEY}"

  ernie:
    base_url: "https://aip.baidubce.com"
    api_key: "${ERNIE_API_KEY}"
    secret_key: "${ERNIE_SECRET_KEY}"
```

---

## Implementation Checklist

- [ ] LLM Interface 定义
- [ ] Mock LLM 实现
- [ ] OpenAI GPT 集成
- [ ] Claude 集成（可选）
- [ ] 阿里云通义千问集成（可选）
- [ ] 百度文心一言集成（可选）
- [ ] 流式处理支持
- [ ] Backpressure 控制
- [ ] Context 取消支持
- [ ] 错误处理和重试
- [ ] 性能监控
- [ ] 配置管理