# AI Rules

## 1. 单一职责

每个模块只做一件事

---

## 2. 不跨模块修改

除非任务明确要求

---

## 3. 严格遵守 protocol

不得随意增加字段，遵循 protocol/message_schema.json (v3.0)

---

## 4. 优先使用已有结构

不要重复造轮子

---

## 5. 所有逻辑必须可解释

---

## 6. 技术栈约束

- **客户端**：C++（跨平台，Windows/macOS/Linux）
  - 使用原生系统 API
  - 保持高性能和低延迟
  - 代码结构：class + interface

- **服务端**：Go
  - 使用标准库和主流开源框架
  - 保持高并发处理能力
  - 代码结构：package + interface

---

## 7. 接口设计原则

- 客户端与服务端通过 WebSocket 通信
- 消息格式严格遵循 protocol/message_schema.json (v3.0)
- 统一消息信封格式：

```json
{
  "type": "audio_chunk",
  "session_id": "uuid",
  "trace_id": "uuid",
  "seq": 1,
  "timestamp": 123456789,
  "payload": {}
}
```

---

## 8. 性能要求

- 客户端音频处理延迟 < 20ms
- 服务端 AI Pipeline 延迟 < 300ms
- 中断响应时间 < 100ms

---

## 9. Backpressure 控制

- 所有 channel 必须有 buffer
- 使用 non-blocking 写入
- 允许选择性丢数据（音频丢旧保新）

---

## 10. Trace-based 设计

- 每轮对话使用唯一的 trace_id
- 中断后生成新的 trace_id
- 所有消息关联 trace_id 用于追踪