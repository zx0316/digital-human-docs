# Digital Human System Docs

## 目标

构建一个端云协同的实时数字人系统，具备：

- 实时语音交互
- 可打断能力（Interrupt）
- 低延迟流式语音输出
- Agent（记忆 + 人格 + 决策）
- Spine驱动的实时表情系统

---

## 系统构成

Client:
- 音频采集
- 实时播放
- Spine渲染
- WebSocket通信

Server:
- ASR（语音识别）
- Agent（记忆 + prompt + 决策）
- LLM（对话生成）
- TTS（语音合成）
- Session管理

---

## 核心设计原则

1. 流式优先（Streaming First）
2. 时间轴统一（Unified Timeline）
3. 端云解耦（Decoupled Execution）
4. 可中断（Interruptible）
5. 记忆驱动（Memory-Driven Agent）

---

## 关键文档阅读顺序

1. architecture/overview.md
2. protocol/message_schema.json
3. dataflow/realtime_pipeline.md
4. agent/agent_design.md
