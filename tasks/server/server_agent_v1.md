# Task: server_agent_v1

## Goal

实现服务端 Agent 模块

---

## Input

- asr_final 文本
- memory_context
- session_history

---

## Output

- LLM 响应文本
- emotion 信息

---

## Dependencies

- protocol/message_schema.json
- server/server_architecture.md
- agent/agent_design.md
- agent/memory_spec.md

---

## Requirements

- 支持 memory 检索
- 支持 prompt 构建
- 支持 LLM 调用
- 输出结构化响应

---

## Non-Goals

- 不实现 LLM（采用第三方 LLM 服务）
- 不实现 ASR
- 不实现 TTS

---

## Acceptance Criteria

- 可以检索 memory
- 可以构建 prompt
- 可以调用 LLM
- 可以输出结构化响应

---

## Trae Prompt

实现一个 Go Agent 模块：

- 输入：文本、memory、history
- 输出：LLM 响应、emotion
- 支持 memory 检索
- 支持 prompt 构建
- 代码结构清晰（package + interface）

参考：
- protocol/message_schema.json (v3.0)
- server/server_architecture.md
- agent/agent_design.md
- agent/memory_spec.md