# Task: server_agent_v1

## Goal

实现服务端 Agent 模块

---

## Input

- 识别文本
- 记忆上下文
- 会话历史

---

## Output

- 响应文本
- 情绪
- 意图

---

## Dependencies

- server/llm_integration.md
- agent/agent_design.md
- agent/memory_spec.md

---

## Requirements

- 支持记忆检索
- 支持 prompt 构建
- 支持 LLM 调用

---

## Non-Goals

- 不实现 LLM 模型
- 不实现 ASR/TTS

---

## Acceptance Criteria

- 可以生成合理响应
- 可以管理记忆

---

## Trae Prompt

实现一个 C++ Agent 模块：

- 输入：识别文本、记忆上下文、会话历史
- 输出：响应文本、情绪、意图
- 支持记忆检索、prompt 构建
- 代码结构清晰（class + interface）

参考：
- agent/agent_design.md