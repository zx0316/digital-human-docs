# Agent Design

## 核心职责

- 理解输入文本
- 检索 memory
- 构建 prompt
- 调用 LLM
- 输出结构化 response

---

## 输入

- current_user_text
- memory_context
- session_history

---

## 输出

{
  "response_text": "",
  "emotion": "neutral|happy|sad",
  "intent": "chat|task|emotion"
}

---

## 设计原则

- memory first
- context aware
- persona consistent
